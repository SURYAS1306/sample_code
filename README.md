# Apache Fory gRPC Integration Design

This repository contains architecture and design documents for adding **gRPC support to Apache Fory** across Java and Python runtimes.

## Overview

The design explores how Apache Fory's serialization system can integrate with gRPC while preserving cross-language compatibility and high performance.

## High-Level Architecture

```mermaid
flowchart LR
IDL --> Compiler
Compiler --> GeneratedCode
GeneratedCode --> ForyCodec
ForyCodec --> gRPC
gRPC --> Network
```

## Documents

### 1. Fory Java & Python gRPC Integration Design
Location:
docs/design/fory-java-python-grpc-integration.md

Covers:
- Compiler pipeline extensions
- Generated API surfaces
- gRPC codec integration
- Serialization boundaries
- Cross-language compatibility
- CI interoperability tests

### 2. gRPC Runtime Architecture (Protobuf & FlatBuffers)
Location:
docs/design/grpc-runtime-architecture.md

Covers:
- gRPC runtime lifecycle
- Stub and runtime interaction
- Serialization triggers
- Streaming RPC behavior
- Memory ownership
- Zero-copy constraints
- Transport message framing

## Goal

These documents were prepared as part of a design exploration for adding **Fory-native gRPC integration** enabling:

- Java ↔ Python cross-language RPC
- Fory-based serialization instead of Protobuf
- High-performance serialization with zero-copy possibilities
