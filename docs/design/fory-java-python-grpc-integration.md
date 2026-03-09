## Apache Fory Java & Python gRPC Integration Design

### Overview

This document proposes an end-to-end design for adding **Java and Python gRPC integration** to Apache Fory using only Fory’s serialization formats (no protobuf payloads). The design is based on:

- The existing **compiler pipeline** in `compiler/fory_compiler/**`
- The **Java runtime** in `java/fory-core/**` (plus `fory-format` as needed)
- The **Python runtime** in `python/pyfory/**`
- Existing IDL support for `service`/`rpc` in the compiler IR

It targets the following artifacts:

- **Java**: `*Service.java`, `*Grpc.java`
- **Python**: `*_service.py`, `*_grpc.py`

With requirements:

- Fory serialization only
- Unary + streaming RPCs
- Cross-language interop:
  - Java server ↔ Python client
  - Python server ↔ Java client
- Zero-copy decoding where possible, with fallbacks
- CI round-trip interoperability tests

---

### 1. Existing Architecture (Compiler & Runtimes)

#### 1.1 Compiler pipeline

**Key modules (all under `compiler/fory_compiler/`)**:

- **CLI & orchestration**: `cli.py`
  - Entry: `main()` / `compile_file()` / `compile_file_recursive()`
  - Uses `frontend.utils.parse_idl_file()` to parse `.fdl`, `.proto`, `.fbs`
  - Merges imports into a unified `Schema` IR
  - Runs `SchemaValidator` (`ir/validator.py`)
  - Dispatches to `GENERATORS[lang]` (`generators/__init__.py`)

- **IDL parsing**:
  - FDL:
    - `frontend/fdl/lexer.py` (`Lexer`)
    - `frontend/fdl/parser.py` (`Parser.parse() -> Schema`)
  - Protobuf / FlatBuffers:
    - `frontend/proto/**`, `frontend/fbs/**`
  - Dispatch:
    - `frontend/utils.py::parse_idl_file(path)`

- **IR definition**:
  - `ir/ast.py`:
    - `Schema`, `Message`, `Enum`, `Union`, `Service`, `RpcMethod`, `Field`, `Import`, etc.
  - `ir/types.py`: `PrimitiveKind`, `PRIMITIVE_TYPES`
  - `ir/validator.py`: `SchemaValidator`
  - `ir/emitter.py`: `FDLEmitter` (for `--emit-fdl`)

- **Code generation**:
  - Base: `generators/base.py` (`BaseGenerator`, `GeneratorOptions`, `GeneratedFile`)
  - Java: `generators/java.py` (`JavaGenerator`)
    - Generates POJOs, union classes, `XxxForyRegistration`
  - Python: `generators/python.py` (`PythonGenerator`)
    - Generates `@pyfory.dataclass` models, union types, `register_*_types`, `_get_fory()`

**Current gap**: Services are already part of IR (`Service`, `RpcMethod` in `ir/ast.py`), but there is **no generator** for Java/Python gRPC artifacts.

---

#### 1.2 Java runtime architecture

**Core packages (under `java/fory-core/src/main/java/org/apache/fory/`)**:

- `Fory.java`
  - Main serialization/deserialization engine
  - Builder-wired `Config` (xlang/native, ref tracking, meta sharing, etc.)
  - Methods:
    - `serialize(Object)`, `serialize(MemoryBuffer, Object)`
    - `deserialize(byte[])`, `deserialize(MemoryBuffer, Class<T>)`, etc.
  - Uses `RefResolver`, `TypeResolver`, `SerializationContext`, `MetaStringResolver`, `MemoryBuffer`

- `resolver/TypeResolver.java` (and related)
  - Maps `Class<?>` ↔ `TypeInfo` (type IDs, `Serializer<?>`, schema meta)
  - `register`, `registerUnion`, `writeTypeInfo`, `readTypeInfo`
  - Bridges **internal type IDs** and **user type IDs** with Fory protocol

- `serializer/*`
  - Primitive and composite serializers
  - `ObjectSerializer`, `GeneratedObjectSerializer`, `GeneratedMetaSharedSerializer`, `UnionSerializer`, etc.

- `codegen/*`, `builder/*`
  - Expression-based JIT serializers
  - `JITContext`, `CodegenSerializer`

- Buffer & util (exact file names may vary):
  - `MemoryBuffer` (under `util`/`io`)
  - Varint encoding, string encoding, bit operations, buffer pooling

**Serialization lifecycle (Java)**:

```mermaid
flowchart LR
  A["User / Generated Java Model"] --> B["ThreadSafeFory\n(from XxxForyRegistration)"]
  B --> C["Fory.serialize(obj)"]
  C --> D["RefResolver.writeRefOrNull"]
  D --> E["TypeResolver.getTypeInfo + writeTypeInfo"]
  E --> F["Serializer.write(buffer, obj)"]
  F --> G["MemoryBuffer.toByteArray()\nSerialized Payload"]
```

**Deserialization lifecycle (Java)**:

```mermaid
flowchart LR
  A["Serialized Payload (byte[])"] --> B["MemoryBuffer.wrap(bytes)"]
  B --> C["Fory.deserialize(buffer)"]
  C --> D["RefResolver.tryPreserveRefId"]
  D --> E["TypeResolver.readTypeInfo"]
  E --> F["Serializer.read(buffer)"]
  F --> G["Object Graph"]
```

**Zero-copy & buffer handling**:

- `MemoryBuffer` reuses internal `byte[]` where possible
- Out-of-band buffers for large blobs:
  - xlang header bit indicates out-of-band data
  - `Fory.deserialize` can take `Iterable<MemoryBuffer>` so data can remain separate
- Row format (`java/fory-format/**`) and SIMD (`java/fory-simd/**`) offer additional high-performance, cache-friendly pathways when used

---

#### 1.3 Python runtime architecture

**Key modules (under `python/pyfory/`)**:

- Pure Python engine:
  - `_fory.py`
    - `class Fory`: pure-Python serializer
  - `registry.py`
    - `class TypeResolver`: Python type registry & type info
  - `serializer.py`
    - Serializer hierarchy for primitives, collections, structs, unions
  - `resolver.py`
    - `MapRefResolver` for reference tracking
  - `buffer.py` / `buffer.pyx`
    - Buffer abstraction, varint/string encoding

- Cython engine:
  - `serialization.pyx`
    - `class Fory`: Cython-backed serializer
  - `buffer.pxi`, C++ includes under `includes/`
    - `CBuffer`-based buffer, `TypeId` helpers, etc.

**Serialization lifecycle (Python)**:

```mermaid
flowchart LR
  A["Generated Python Dataclass / Union"] --> B["_get_fory()\n(ThreadSafeFory)"]
  B --> C["pyfory.Fory.serialize(obj)"]
  C --> D["MapRefResolver.write_ref_or_null"]
  D --> E["TypeResolver.get_type_info + write_type_info"]
  E --> F["Serializer.write(buffer, obj)"]
  F --> G["buffer.to_bytes()\nSerialized Payload"]
```

**Deserialization lifecycle (Python)**:

```mermaid
flowchart LR
  A["Serialized Payload (bytes)"] --> B["Buffer.wrap(bytes)"]
  B --> C["pyfory.Fory.deserialize(buffer)"]
  C --> D["MapRefResolver.try_preserve_ref_id"]
  D --> E["TypeResolver.read_type_info"]
  E --> F["Serializer.read(buffer)"]
  F --> G["Object Graph"]
```

**Zero-copy & buffer handling**:

- Out-of-band semantics similar to Java:
  - Header flags + buffer objects (`BytesBufferObject`, `NDArrayBufferObject`, etc. in `serializer.py`)
- Cython engine uses C++ `CBuffer` for performance
- Numpy / `memoryview`-based serializers can **avoid copying** array data

---

#### 1.4 Cross-language compatibility points (Java ↔ Python)

- Shared protocol spec:
  - `docs/specification/xlang_serialization_spec.md`
  - `docs/specification/xlang_type_mapping.md`
- Consistent type IDs:
  - Java: `org.apache.fory.type.Types`
  - Python: `pyfory.TypeId` (and associated constants)
- Same xlang header bits, reference flags, meta string encoding, TypeDef (schema meta)
- Compiler-generated Java and Python code:
  - Based on **same IR** (`compiler/fory_compiler/ir/ast.py`)
  - Uses **same type IDs and schema options** from FDL/FDL-compatible inputs
- Integration tests in `integration_tests/idl_tests/**` already verify cross-language struct/union round-trips; gRPC tests will layer on top.

---

#### 1.5 Logical integration points for gRPC

**Compiler side (`/compiler`)**:

- IR already has:
  - `Service`, `RpcMethod` in `ir/ast.py`
- Place to extend:
  - `generators/java.py`:
    - Add generation for `*Service.java` and `*Grpc.java`
  - `generators/python.py`:
    - Add generation for `*_service.py` and `*_grpc.py`
- Additional configuration:
  - File-level or service-level options in FDL:
    - e.g. `option java_grpc_package = "..."`, `option python_grpc_module = "..."`

**Java runtime side (`/java`)**:

- Integration point:
  - gRPC server: custom `Marshaller`/`MethodDescriptor` using `Fory` instead of protobuf
  - gRPC client: same `Marshaller` usage and generated stubs
- Potential new package:
  - `org.apache.fory.grpc` in `java/fory-core` or a new submodule (e.g. `java/fory-grpc`)

**Python runtime side (`/python`)**:

- Integration point:
  - gRPC Python (`grpcio`) custom `Codec` / `CallCredentials` / `GenericRpcHandler` with Fory bytes
- Potential module:
  - `pyfory.grpc` (new module) for shared codec utilities

---

### 2. Proposed Compiler Extensions for gRPC

#### 2.1 Compiler pipeline with service generation

```mermaid
flowchart LR
  A[".fdl/.proto/.fbs"] --> B["frontend.*\nparse_idl_file"]
  B --> C["Schema (IR)\n(ir/ast.py)"]
  C --> D["SchemaValidator\n(ir/validator.py)"]
  D --> E["JavaGenerator\n(generators/java.py)"]
  D --> F["PythonGenerator\n(generators/python.py)"]

  E --> E1["Models + Fory Registration\n(current)"]
  E --> E2["*Service.java\n*Grpc.java\n(new)"]

  F --> F1["Models + Fory Registration\n(current)"]
  F --> F2["*_service.py\n*_grpc.py\n(new)"]
```

**Changes in `generators/java.py`**:

- For each `Service` in `Schema`:
  - Generate:
    - `FooService.java`: **user-implementable service interface**
    - `FooGrpc.java`: **gRPC binding + factory**

**Changes in `generators/python.py`**:

- For each `Service`:
  - Generate:
    - `foo_service.py`: **user-implementable service base class / ABC**
    - `foo_grpc.py`: **gRPC server + stub helpers**

FDL example (conceptual):

```fdl
service Greeter {
  rpc SayHello(HelloRequest) returns (HelloReply);
  rpc LotsOfReplies(HelloRequest) returns (stream HelloReply);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloReply);
  rpc BidiHello(stream HelloRequest) returns (stream HelloReply);
}
```

The IR already captures streaming information (or can be extended minimally).

---

#### 2.2 Generated Java API surface

**Target package**: `org.example.generated` (from FDL `package` / `java_package`)

1. **Service interface** – `GreeterService.java`

```java
public interface GreeterService {

    // Unary
    HelloReply sayHello(HelloRequest request, io.grpc.Context context);

    // Server streaming
    void lotsOfReplies(
        HelloRequest request,
        org.apache.fory.grpc.ForyStreamObserver<HelloReply> responseObserver,
        io.grpc.Context context);

    // Client streaming
    org.apache.fory.grpc.ForyStreamObserver<HelloRequest> lotsOfGreetings(
        org.apache.fory.grpc.ForyStreamObserver<HelloReply> responseObserver,
        io.grpc.Context context);

    // Bidi streaming
    org.apache.fory.grpc.ForyStreamObserver<HelloRequest> bidiHello(
        org.apache.fory.grpc.ForyStreamObserver<HelloReply> responseObserver,
        io.grpc.Context context);
}
```

2. **gRPC binding** – `GreeterGrpc.java`

```java
public final class GreeterGrpc {
    public static final String SERVICE_NAME = "example.Greeter";

    // Marshaller built on Fory
    private static final io.grpc.MethodDescriptor<HelloRequest, HelloReply> METHOD_SAY_HELLO =
        org.apache.fory.grpc.ForyDescriptors.unaryMethod(
            SERVICE_NAME,
            "SayHello",
            HelloRequest.class,
            HelloReply.class);

    // Additional MethodDescriptors for streaming rpcs...

    public static abstract class GreeterImplBase implements io.grpc.BindableService {
        public void sayHello(
            HelloRequest request,
            io.grpc.stub.StreamObserver<HelloReply> responseObserver) {
            // default UNIMPLEMENTED
            io.grpc.stub.ServerCalls.asyncUnimplementedUnaryCall(
                METHOD_SAY_HELLO, responseObserver);
        }

        // streaming default methods...

        @Override
        public final io.grpc.ServerServiceDefinition bindService() {
            return io.grpc.ServerServiceDefinition
                .builder(SERVICE_NAME)
                .addMethod(
                    METHOD_SAY_HELLO,
                    org.apache.fory.grpc.ForyServerCalls.unary(
                        this::sayHello,
                        HelloRequest.class,
                        HelloReply.class))
                // add other methods...
                .build();
        }
    }

    public static final class GreeterStub extends io.grpc.stub.AbstractStub<GreeterStub> {
        // uses ForyClientCalls with Fory-based marshaller
    }
}
```

3. **Runtime support classes** (proposed new package `org.apache.fory.grpc`):

- `ForyMarshaller<T>`
  - Implements `io.grpc.MethodDescriptor.Marshaller<T>`
  - Uses `ThreadSafeFory` configured for the IDL module (through generated registration).

- `ForyDescriptors`
  - Helpers for building `MethodDescriptor` instances for unary/streaming.

- `ForyServerCalls`, `ForyClientCalls`
  - Analogous to `io.grpc.stub.ServerCalls` / `ClientCalls`, but parameterized with Fory marshallers and providing zero-copy-friendly hooks.

---

#### 2.3 Generated Python API surface

Assume standard `grpcio` (v1) usage.

1. **Service base** – `greeter_service.py`

```python
import abc
from typing import AsyncIterator, Iterator
import pyfory

from .hello_models import HelloRequest, HelloReply  # generated dataclasses


class GreeterService(abc.ABC):
    """User-implemented service base class."""

    # Unary
    @abc.abstractmethod
    async def SayHello(self, request: HelloRequest, context) -> HelloReply:
        ...

    # Server streaming
    @abc.abstractmethod
    async def LotsOfReplies(
        self, request: HelloRequest, context
    ) -> AsyncIterator[HelloReply]:
        ...

    # Client streaming
    @abc.abstractmethod
    async def LotsOfGreetings(
        self, request_iter: AsyncIterator[HelloRequest], context
    ) -> HelloReply:
        ...

    # Bidi streaming
    @abc.abstractmethod
    async def BidiHello(
        self, request_iter: AsyncIterator[HelloRequest], context
    ) -> AsyncIterator[HelloReply]:
        ...
```

2. **gRPC wiring** – `greeter_grpc.py`

```python
import grpc
import pyfory

from . import greeter_service
from .hello_models import HelloRequest, HelloReply
from .hello_models import _get_fory as _get_hello_fory  # generated


class ForyGreeterServicer(greeter_service.GreeterService):
    """Adapter base implementing grpc Servicer interface using Fory codec."""

    # This class may remain abstract; concrete implementations subclass it.


def add_GreeterServicer_to_server(servicer: greeter_service.GreeterService, server: grpc.Server):
    codec = pyfory.grpc.ForyCodec(_get_hello_fory)

    rpc_method_handlers = {
        "SayHello": grpc.unary_unary_rpc_method_handler(
            _wrap_unary(servicer.SayHello, codec, HelloRequest, HelloReply),
            request_deserializer=codec.deserialize(HelloRequest),
            response_serializer=codec.serialize(HelloReply),
        ),
        # ... streaming handlers ...
    }

    generic_handler = grpc.method_handlers_generic_handler(
        "example.Greeter", rpc_method_handlers
    )
    server.add_generic_rpc_handlers((generic_handler,))


class GreeterStub:
    """Client stub using Fory serialization."""
    def __init__(self, channel: grpc.Channel):
        self._codec = pyfory.grpc.ForyCodec(_get_hello_fory)
        self._channel = channel

    def SayHello(self, request: HelloRequest, timeout=None, metadata=None) -> HelloReply:
        return self._channel.unary_unary(
            "/example.Greeter/SayHello",
            request_serializer=self._codec.serialize(HelloRequest),
            response_deserializer=self._codec.deserialize(HelloReply),
        )(request, timeout=timeout, metadata=metadata)
```

3. **Runtime support (`pyfory.grpc`)**:

- `class ForyCodec:`
  - Holds `ThreadSafeFory` factory for the IDL module.
  - Provides `serialize(cls) -> Callable[obj -> bytes]` and `deserialize(cls) -> Callable[bytes -> obj]`.
  - For streaming:
    - `iter_serialize` / `iter_deserialize` helpers to integrate with generator- and async-iterator-based handlers.

---

### 3. gRPC Request Lifecycle with Fory

#### 3.1 High-level lifecycle (cross-language)

```mermaid
sequenceDiagram
    participant JClient as Java Client
    participant JCodec as Java ForyCodec
    participant Transport as HTTP/2 + gRPC
    participant PCodec as Python ForyCodec
    participant PServer as Python Service Impl

    JClient->>JCodec: serialize(HelloRequest)
    JCodec->>Transport: gRPC unary call (bytes)
    Transport->>PCodec: deliver bytes
    PCodec->>PServer: deserialize -> HelloRequest object
    PServer->>PCodec: HelloReply object
    PCodec->>Transport: serialize(HelloReply) -> bytes
    Transport->>JCodec: deliver bytes
    JCodec->>JClient: deserialize -> HelloReply object
```

For streaming calls, the same pipeline is just repeated per message in each direction.

#### 3.2 Serialization boundaries

- **Java**:
  - Generated stubs call:
    - `ForyMarshaller<T>.stream(T)` / `parse(InputStream)` internally, implemented using `Fory.serialize` / `Fory.deserialize`.
  - Boundary:
    - Between **Java objects** (generated models) and **`byte[]` payloads** in gRPC frame.

- **Python**:
  - Generated stubs and server handlers call:
    - `ForyCodec.serialize(cls)(obj)` / `ForyCodec.deserialize(cls)(bytes)`.
  - Boundary:
    - Between **Python dataclasses** and **`bytes`** for gRPC.

---

### 4. Zero-copy Feasibility and Fallbacks

#### 4.1 Java

- **Feasible**:
  - Reuse `MemoryBuffer` and underlying `byte[]` to avoid copies.
  - Zero-copy into application for certain large types if:
    - `Fory` uses **out-of-band buffers** and returns references by wrapping them without copying.
  - Potential extension: gRPC **unsafe** optimizations using Netty’s `ByteBuf` and Fory decoders that read directly from `ByteBuf` (advanced).

- **Constraints**:
  - gRPC Java’s `Marshaller` API typically deals with `InputStream` and `byte[]`:
    - We must at least materialize a contiguous `byte[]` for the message.
  - Zero-copy is mostly limited to:
    - Avoiding extra copies *inside* Fory.
    - Sharing underlying buffers with application code where safe.

- **Fallbacks**:
  - If zero-copy cannot be obtained (most typical cases), fallback is:
    - `Fory.serialize` → copy to byte[] → gRPC writes.
    - On read, `byte[]` from gRPC → wrap in `MemoryBuffer` → `Fory.deserialize`.

#### 4.2 Python

- **Feasible**:
  - With `serialization.pyx` + C++ `CBuffer`:
    - Efficient copying into Python `bytes`.
  - For **large arrays** / numpy:
    - Use `NDArrayBufferObject` and similar constructs to hold data without copying.
    - However, gRPC Python still expects `bytes` on the wire.

- **Constraints**:
  - Python `grpcio` interface is `bytes`-based for messages.
  - True zero-copy from network to Python objects is hard:
    - You must at least read from `socket` into some buffer.

- **Fallbacks**:
  - Standard path:
    - `ForyCodec.serialize`: `Fory.serialize(obj) -> bytes`.
    - `ForyCodec.deserialize`: `Fory.deserialize(bytes) -> obj`.

---

### 5. CI Structure for Interoperability

#### 5.1 Test topology

We will extend `integration_tests` with a new module, e.g. `integration_tests/grpc_xlang_tests`:

- **Tests**:

  1. **Java server ↔ Python client (unary + streaming)**:
     - Start Java gRPC server (using generated `*Grpc.java`) with Fory codec.
     - Python test code:
       - Use generated `*_grpc.py` stub to call unary, server streaming, client streaming, bidi streaming.
     - Verify payloads and behavior.

  2. **Python server ↔ Java client (unary + streaming)**:
     - Start Python gRPC server with generated `*_service.py` + `*_grpc.py`.
     - Java test code:
       - Use generated Java stub in `*Grpc.java`.

  3. **IDL-driven round-trip**:
     - Use canonical `.fdl` in `integration_tests/idl_tests` for messages and services.
     - Compile to both Java and Python.
     - Run tests via Maven (`java`) and Python (`pytest`) orchestrated by a shell script or Maven profile.

#### 5.2 CI diagram

```mermaid
flowchart TD
  A["IDL Schemas (.fdl)"] --> B["compiler/fory_compiler/cli.py\n(foryc)"]
  B --> C["Generated Java Artifacts\nModels + *Service.java + *Grpc.java"]
  B --> D["Generated Python Artifacts\nModels + *_service.py + *_grpc.py"]

  C --> E["Build Java server & client\n(java/fory-core + gRPC)"]
  D --> F["Build Python server & client\n(python/pyfory + grpcio)"]

  E --> G["Integration Tests\nJava server ↔ Python client"]
  F --> H["Integration Tests\nPython server ↔ Java client"]

  G --> I["CI Pass/Fail"]
  H --> I
```

- Hooks:
  - Maven profile in `java/pom.xml` to run gRPC xlang tests.
  - Python `pytest` tests under `integration_tests/grpc_xlang_tests/python`.
  - Combined driver script under `integration_tests/grpc_xlang_tests/run.sh`.

---

### 6. Implementation Strategy Summary

1. **Extend IR and validation (if needed)**:
   - Ensure `Service`/`RpcMethod` supports:
     - Streaming flags (client/server/bidi).
     - Fory-specific options (e.g. service name, package overrides).

2. **Extend Java generator** (`compiler/fory_compiler/generators/java.py`):
   - For each `Service`:
     - Emit `*Service.java` interface.
     - Emit `*Grpc.java` with:
       - `MethodDescriptor`s using `ForyMarshaller`.
       - `ImplBase` and `Stub` types.

3. **Extend Python generator** (`compiler/fory_compiler/generators/python.py`):
   - For each `Service`:
     - Emit `*_service.py` base class.
     - Emit `*_grpc.py` integration with `grpcio` using `pyfory.grpc.ForyCodec`.

4. **Add runtime grpc support**:
   - Java: new `org.apache.fory.grpc` package with:
     - `ForyMarshaller`, `ForyDescriptors`, `ForyServerCalls`, `ForyClientCalls`.
   - Python: new `pyfory.grpc` module with:
     - `ForyCodec` and helpers for streaming.

5. **Add CI integration tests**:
   - New `integration_tests/grpc_xlang_tests` module.
   - Scripts to compile IDL, start Java server, run Python client tests, and vice versa.

This yields a cohesive, cross-language gRPC layer that remains **fully Fory-native** while aligning with existing gRPC usage patterns and Fory’s runtime design.

---

## gRPC Runtime Architecture with Protobuf and FlatBuffers

### Overview

This document explains gRPC’s runtime architecture and how it integrates with serialization frameworks, using **Protobuf** and **FlatBuffers** as concrete reference points. The goal is to inform the design of a Fory-based gRPC integration for Apache Fory, particularly in:

- Java (similar to how Protobuf and FlatBuffers integrate today)
- Python (`grpcio` with custom serializers)

We focus on request lifecycle, stub interactions, serialization triggers, memory/ownership, streaming behavior, and custom codec integration.

---

### 1. gRPC Runtime Request Lifecycle

#### 1.1 High-level flow

```mermaid
flowchart LR
  A[Client Stub] --> B[Request Serializer\n(Codec/Marshaller)]
  B --> C[gRPC Transport\nHTTP/2 Frames]
  C --> D[Server Deserializer\n(Codec/Marshaller)]
  D --> E[Service Handler\n(User Implementation)]
  E --> F[Response Serializer]
  F --> C
```

#### 1.2 Protobuf-based integration (reference)

**Java (typical stack, e.g. Google gRPC)**:

- Generated `*Grpc.java` (from `protoc` with `protoc-gen-grpc-java`):
  - Defines:
    - `MethodDescriptor<ReqT, RespT>` with:
      - `ProtoUtils.marshaller(ReqT.getDefaultInstance())`
    - `ImplBase` server base with `bindService()` using `ServerCalls.asyncUnaryCall`, etc.
    - `Stub` classes for clients.

- `ProtoUtils.marshaller` (in gRPC Java) is a **marshaller using Protobuf**:
  - `serialize(T message)` calls `message.toByteArray()`
  - `deserialize(InputStream is)` calls `T.parseFrom(InputStream)`

**Request lifecycle (Protobuf/Java)**:

```mermaid
sequenceDiagram
    participant ClientStub
    participant Marshaller
    participant Channel
    participant Server
    participant Handler

    ClientStub->>Marshaller: serialize(Req)
    Marshaller->>Channel: bytes
    Channel->>Server: HTTP/2 DATA frame
    Server->>Marshaller: parse(InputStream)
    Marshaller->>Handler: Req object
    Handler->>Marshaller: Resp object
    Marshaller->>Channel: bytes
    Channel->>ClientStub: bytes
    ClientStub->>Marshaller: parse(InputStream)
    Marshaller->>ClientStub: Resp object
```

#### 1.3 FlatBuffers-based integration (reference)

FlatBuffers usually integrate via **custom marshallers**:

- The request/response types often wrap a `ByteBuffer` around the incoming `byte[]`:
  - `MyTable.getRootAsMyTable(ByteBuffer)` (zero-copy read).
- The marshaller:
  - On serialize: builds a FlatBuffer and returns `byte[]`.
  - On deserialize:
    - Wraps `byte[]` into `ByteBuffer` and passes to `getRootAs…`.
    - No object allocation per field (FlatBuffers are zero-copy reads).

This pattern is analogous to how Fory can **wrap a `MemoryBuffer` or `CBuffer`** around bytes and decode without extra copying of internal data structures where possible.

---

### 2. Generated Stubs and gRPC Runtime

#### 2.1 Generated artifacts structure

**Protobuf/Java**:

- `FooGrpc.java`:

  - `FooGrpc.FooImplBase implements io.grpc.BindableService`
    - Methods: `public void myRpc(Req request, StreamObserver<Resp> responseObserver)`
    - `bindService` wires `MethodDescriptor`s and `ServerCalls.asyncUnaryCall` or streaming equivalents.

  - `FooGrpc.FooStub extends AbstractStub<FooStub>`
    - Uses `ClientCalls.unaryCall`, `blockingUnaryCall`, `asyncUnaryCall`, etc.

**FlatBuffers**:

- gRPC artifacts similar in shape; the only difference is how marshallers serialize/deserialize.

**Key pattern**:

- **Generated code is thin**:
  - Declares the service API.
  - Wires gRPC runtime (`MethodDescriptor`, `ServerCalls`, `ClientCalls`).
  - Delegates actual serialization/deserialization to a **marshaller** (codec).

---

### 3. Serialization & Deserialization Triggers

#### 3.1 Where serialization occurs

- **Client stub**:
  - For unary calls:
    - `ClientCalls.unaryCall(channel.newCall(method, callOptions), request)`
      - Triggers marshaller’s `stream(request)` / `toBytes()` for a Protobuf message.
  - For streaming:
    - On each `onNext(request)` from client code, the marshaller is invoked to turn request objects into bytes.

- **Server side**:
  - Prior to sending responses via `StreamObserver.onNext(response)`:
    - Server calls appropriate server call handler (e.g. `ServerCalls.asyncUnaryCall`), which:
      - Calls marshaller’s `stream(response)` before writing data frames.

#### 3.2 Where deserialization occurs

- **Server**:
  - Incoming data frames are aggregated by gRPC runtime.
  - Once a full message is assembled, `Marshaller.parse(InputStream)` is used to reconstruct objects (Protobuf `parseFrom` / FlatBuffers `getRootAs`).
- **Client**:
  - Similar: aggregated frames → marshaller.parse → objects delivered to `StreamObserver` or stub return values.

---

### 4. Memory Ownership and Object Lifecycle

#### 4.1 Protobuf

- **Serialization**:
  - `message.toByteArray()`:
    - Allocates a new byte array with the full message.
  - Typically no zero-copy; each message is a copy into `byte[]`.

- **Deserialization**:
  - `T.parseFrom(InputStream)`:
    - Allocates a new message object.
    - Parses internal fields, performing copies for strings, repeated fields, etc.

- **Ownership**:
  - The application fully owns objects returned by parse.
  - The runtime owns the wire `byte[]` and HTTP/2 frame buffers.

#### 4.2 FlatBuffers

- **Serialization**:
  - FlatBuffers builder creates a `ByteBuffer` with data; a `byte[]` may be returned from that buffer.
- **Deserialization**:
  - `MyTable.getRootAsMyTable(ByteBuffer buffer)`:
    - No deep copy; holds references into `ByteBuffer`.
- **Ownership**:
  - Application must ensure underlying `ByteBuffer` lives as long as the FlatBuffers objects referencing it.

---

### 5. Streaming RPC Behavior

#### 5.1 Unary

- **Protobuf / FlatBuffers**:
  - Single request → single response.
  - Marshaller invoked once per side.

#### 5.2 Client streaming

- Multiple `request.onNext()` calls each trigger marshalling.
- Server aggregates or processes streaming messages.
- Final `onCompleted` triggers server handler’s final response, marshalled once.

#### 5.3 Server streaming

- Single client request triggers server returning `onNext` multiple times.
- Each `onNext` triggers serialization.

#### 5.4 Bidirectional streaming

- Both ends send `onNext` events on their respective streams.
- Serialization/deserialization is invoked per-message, directionally symmetric.

**Pattern**:

```mermaid
sequenceDiagram
    participant Client
    participant CMarsh as Client Marshaller
    participant Channel
    participant SMarsh as Server Marshaller
    participant Server

    loop messages
        Client->>CMarsh: serialize(req_i)
        CMarsh->>Channel: bytes
        Channel->>SMarsh: deliver bytes
        SMarsh->>Server: deserialize -> req_i
        Server->>SMarsh: resp_j
        SMarsh->>Channel: serialize(resp_j)
        Channel->>CMarsh: bytes
        CMarsh->>Client: deserialize -> resp_j
    end
```

---

### 6. Custom Codec Integration

#### 6.1 Java custom marshaller

Protobuf’s `ProtoUtils.marshaller` is just one implementation of `Marshaller<T>`. Any custom codec (e.g., Fory) follows the same pattern:

```java
public final class ForyMarshaller<T> implements MethodDescriptor.Marshaller<T> {
    private final Class<T> clazz;
    private final org.apache.fory.ThreadSafeFory fory;

    @Override
    public InputStream stream(T value) {
        byte[] bytes = fory.serialize(value);
        return new ByteArrayInputStream(bytes);
    }

    @Override
    public T parse(InputStream stream) {
        byte[] bytes = readAllBytes(stream);
        return fory.deserialize(bytes, clazz);
    }
}
```

- With additional optimization:
  - Use gRPC’s `InputStream` wrappers that can avoid extra copies.
  - Integrate directly with `MemoryBuffer` for minimal overhead.

#### 6.2 Python custom codec

In `grpcio`:

- Request/response serializer/deserializer signatures:

```python
channel.unary_unary(
    "/Service/Method",
    request_serializer=lambda obj: bytes,
    response_deserializer=lambda b: obj,
)
```

- Custom Fory codec:

```python
class ForyCodec:
    def __init__(self, get_fory):
        self._get_fory = get_fory

    def serialize(self, cls):
        def _ser(obj):
            return self._get_fory().serialize(obj)
        return _ser

    def deserialize(self, cls):
        def _des(b):
            return self._get_fory().deserialize(b, cls)
        return _des
```

This is analogous to how Protobuf’s Python integration uses `message.SerializeToString()` and `cls.FromString()`.

---

### 7. Where Zero-Copy is Achievable vs Constrained

#### 7.1 Achievable

- **FlatBuffers**:
  - Immediate zero-copy for reads from `ByteBuffer` once bytes are in memory.
- **Fory (design) analogues**:
  - If gRPC can provide a `ByteBuffer` or a zero-copy `ByteBuf` (Netty), Fory could:
    - Wrap it in `MemoryBuffer` or `CBuffer` without copying content.
    - Read fields directly into objects (or row format base objects).

- **Row format / direct buffer access**:
  - With Fory row format, storing decoded row objects that still reference underlying buffer can achieve a similar effect to FlatBuffers.

#### 7.2 Constrained

- **Network stack**:
  - gRPC implementations generally assemble messages in their own buffers; user code sees `byte[]`/`ByteBuffer`/`bytes` abstractions.
- **gRPC API contracts**:
  - Java `Marshaller<T>` operates on `InputStream`, not arbitrarily on `ByteBuf`.
  - Python `grpcio` requires `bytes` for payloads.
- **Safety & lifecycle**:
  - Even when referencing shared `ByteBuffer`s, care must be taken to ensure they outlive the decoding objects.

Thus, **zero-copy** is mainly achievable **inside** the serialization framework once bytes are available, not necessarily from NIC → application without copies.

---

### 8. Protobuf vs FlatBuffers vs Fory (implications for Fory design)

#### 8.1 Protobuf-style (deep copy) vs FlatBuffers-style (zero-copy)

- **Protobuf**:
  - Simpler semantics; object graphs independent of wire buffer.
  - Cost: more allocations and copies.

- **FlatBuffers**:
  - High performance, zero-copy reads.
  - Constraints: objects must not outlive underlying buffer; random-access field reads.

- **Fory**:
  - Already implements both:
    - Object graph format (xlang, like Protobuf with references and schema meta).
    - Row format (like FlatBuffers/Arrow for random-access, zero-copy-ish).
  - gRPC design can:
    - For initial version: emulate **Protobuf-style** integration (full object reconstruction).
    - Later: add row-format-based handlers for **FlatBuffers-style** zero-copy flows.

#### 8.2 Applying to Apache Fory gRPC integration

- **Short term (GSoC scope)**:
  - Implement **Protobuf-like** integration:
    - Generate service APIs and stubs (Java/Python).
    - Use object-graph xlang serialization via `Fory.serialize`/`deserialize`.
    - Keep design open for future row-format streaming.

- **Long term**:
  - Introduce optional **row-based wire format** for gRPC:
    - Service methods that return row-format views or columnar results.
    - Align with `java/fory-format` and `python/pyfory/format`.

---

### 9. Summary

- gRPC’s runtime architecture centers around:
  - **Generated service and stub code** that is mostly wiring.
  - A **serialization codec** (marshaller) pluggable per method.
- Protobuf and FlatBuffers illustrate two ends of the spectrum:
  - Protobuf: deep-copy, object-centric; easy but less zero-copy.
  - FlatBuffers: buffer-centric; zero-copy, but lifecycle-sensitive.
- For Apache Fory:
  - We will **mirror Protobuf’s integration pattern**:
    - Generate `*Grpc.java` / `*_grpc.py` wrappers.
    - Use Fory-based marshallers/codecs in Java and Python.
  - We can later approach FlatBuffers-level zero-copy by:
    - Leveraging row format and tighter integration with transport buffers.

This understanding feeds directly into the **Fory gRPC integration design** (Document 1), ensuring the implementation fits gRPC’s architecture while leveraging Fory’s existing serialization and zero-copy capabilities.
