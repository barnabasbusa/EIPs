---
eip: TBD
title: SSZ Encoding for Engine API
description: Replace JSON encoding with SSZ encoding in the Engine API to reduce serialization overhead and enable scaling
author: Barnabas Busa (@barnabasbusa)
discussions-to: https://ethereum-magicians.org/t/eip-ssz-encoding-for-engine-api/TBD
status: Draft
type: Standards Track
category: Interface
created: 2026-01-22
requires: 3675, 7495, 7688, 7916
---

## Abstract

This EIP proposes replacing JSON encoding with Simple Serialize (SSZ) encoding for the Engine API communication between execution layer (EL) and consensus layer (CL) clients. The current JSON-RPC based Engine API introduces significant serialization overhead that becomes a bottleneck as block sizes increase. SSZ encoding provides deterministic, efficient binary serialization that reduces CPU overhead, decreases payload sizes, and aligns the Engine API with the native encoding format already used by the consensus layer.

## Motivation

The Engine API, introduced in [EIP-3675](./eip-3675.md), defines the communication interface between execution and consensus layer clients. Currently, this API uses JSON-RPC 2.0 with JSON encoding for all payloads. While JSON provides human readability and wide tooling support, it introduces several inefficiencies:

### Serialization Overhead

JSON encoding requires:
- Converting binary data (hashes, addresses, bytecode) to hexadecimal strings, doubling their size
- String escaping and quoting for all keys and string values
- Decimal string representation of integers
- Parsing and validation of the textual format

Profiling of production clients shows that JSON serialization and deserialization can consume 5-15% of block processing time, with this percentage increasing as blocks grow larger.

### Scaling Limitations

As Ethereum scales through various mechanisms (increased gas limits, blob transactions, state growth), the Engine API payloads grow correspondingly:

- `ExecutionPayload` objects can exceed 1MB with full transaction data
- `GetPayloadResponse` includes both the payload and associated blob data
- Each block requires multiple Engine API round-trips (newPayload, forkchoiceUpdated, getPayload, getBlobs)

The JSON encoding overhead compounds with payload size, creating a significant bottleneck for block propagation and validation timing.

### Format Mismatch

The consensus layer natively uses SSZ for all internal data structures and network communication. The current architecture requires:

1. CL deserializes SSZ beacon block
2. CL serializes execution payload to JSON
3. EL deserializes JSON execution payload
4. EL processes and responds with JSON
5. CL deserializes JSON response
6. CL re-serializes to SSZ for storage/networking

This double conversion at the API boundary is wasteful and error-prone.

### Determinism

JSON encoding is not deterministic—the same data can have multiple valid JSON representations (key ordering, whitespace, number formats). While the Engine API specifies conventions, SSZ provides inherent determinism, eliminating potential inconsistencies.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Transport Protocol

The Engine API SHALL support SSZ-encoded requests and responses over the existing HTTP transport. The encoding format is negotiated via HTTP content-type headers:

- `Content-Type: application/ssz` indicates an SSZ-encoded request body
- `Accept: application/ssz` indicates the client prefers an SSZ-encoded response
- `Content-Type: application/json` (current default) continues to indicate JSON encoding

Clients MUST support both encodings during the transition period. Clients MAY deprecate JSON encoding in future hard forks.

### SSZ Schemas

All Engine API types SHALL have corresponding SSZ container definitions. The SSZ schemas mirror the existing JSON type definitions with the following mappings:

| JSON Type | SSZ Type |
|-----------|----------|
| `DATA` (hex string) | `ByteList[max_length]` or `ByteVector[length]` |
| `QUANTITY` (hex integer) | `uint64`, `uint256`, or appropriate fixed-width integer |
| `Hash32` | `Bytes32` |
| `Address` | `Bytes20` |
| `Bloom` | `ByteVector[256]` |
| `Array<T>` | `ProgressiveList[T]` or `List[T, max_length]` |
| `null` | Absent field (bit unset in `active_fields` bitvector) |

### Stable Containers (ProgressiveContainer)

This EIP adopts the forward-compatible SSZ types defined in [EIP-7495](./eip-7495.md) (SSZ ProgressiveContainer) and [EIP-7916](./eip-7916.md) (SSZ ProgressiveList). These types provide stable generalized indices (gindices) that remain constant even when fields are added or removed across fork boundaries.

Using `ProgressiveContainer` instead of regular `Container` ensures:

1. **Merkle Proof Stability**: Light clients and smart contract verifiers can validate proofs without updating when unrelated fields change
2. **Forward Compatibility**: New Engine API versions can add fields without breaking existing implementations
3. **Alignment with Consensus Layer**: [EIP-7688](./eip-7688.md) converts consensus data structures including `ExecutionPayload` to use `ProgressiveContainer`

The `active_fields` bitvector indicates which fields are present in a given version. When a field is absent, it does not consume serialization space but maintains its position in the Merkle tree.

### ExecutionPayload SSZ Schema

The `ExecutionPayload` schema uses `ProgressiveContainer` to enable forward-compatible evolution:

```python
class WithdrawalSSZ(ProgressiveContainer):
    index: uint64
    validator_index: uint64
    address: Bytes20
    amount: uint64

class ExecutionPayloadSSZ(ProgressiveContainer):
    parent_hash: Bytes32
    fee_recipient: Bytes20
    state_root: Bytes32
    receipts_root: Bytes32
    logs_bloom: ByteVector[256]
    prev_randao: Bytes32
    block_number: uint64
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[32]
    base_fee_per_gas: uint256
    block_hash: Bytes32
    transactions: ProgressiveList[ByteList[MAX_BYTES_PER_TRANSACTION]]
    withdrawals: ProgressiveList[WithdrawalSSZ]
    blob_gas_used: uint64
    excess_blob_gas: uint64
```

Future fields (e.g., from EIP-7702 authorizations, block-level access lists) can be appended to this schema by extending the `active_fields` bitvector without invalidating existing Merkle proofs for unchanged fields.

### Request/Response Envelope

Engine API methods using SSZ encoding SHALL use the following envelope structure:

```python
class EngineRequest(Container):
    id: uint64
    method: ByteList[64]
    params: ByteList[MAX_REQUEST_SIZE]  # SSZ-encoded method-specific params

class EngineResponse(ProgressiveContainer):
    id: uint64
    result: ByteList[MAX_RESPONSE_SIZE]  # SSZ-encoded method-specific result
    error: EngineError  # absent via active_fields when no error

class EngineError(Container):
    code: int32
    message: ByteList[1024]
```

### Method-Specific Schemas

Each Engine API method SHALL define SSZ schemas for its parameters and return value. For example:

```python
class NewPayloadRequest(ProgressiveContainer):
    execution_payload: ExecutionPayloadSSZ
    expected_blob_versioned_hashes: ProgressiveList[Bytes32]
    parent_beacon_block_root: Bytes32

class PayloadStatusSSZ(ProgressiveContainer):
    status: uint8  # 0=VALID, 1=INVALID, 2=SYNCING, 3=ACCEPTED, 4=INVALID_BLOCK_HASH
    latest_valid_hash: Bytes32  # absent via active_fields when null
    validation_error: ByteList[1024]  # absent via active_fields when null
```

### Version Negotiation

Upon connection, clients SHALL perform version negotiation to determine supported encodings:

1. The `engine_exchangeCapabilities` method SHALL include encoding capabilities:
   - `"sszV1"` indicates support for SSZ encoding as specified in this EIP

2. If both clients support SSZ, subsequent requests SHOULD use SSZ encoding

3. Clients MUST fall back to JSON encoding if the peer does not advertise SSZ support

### Streaming Support

For large payloads, SSZ encoding enables efficient streaming:

- Clients MAY use chunked transfer encoding for SSZ payloads
- The fixed-offset nature of SSZ containers allows partial parsing before full receipt
- Blob data MAY be streamed separately using the `engine_getBlobsV3` method

## Rationale

### Why SSZ over other binary formats?

Several binary serialization formats were considered:

- **Protocol Buffers**: Requires schema compilation, not natively used by Ethereum
- **MessagePack**: More compact than JSON but lacks determinism guarantees
- **CBOR**: Standardized but not used elsewhere in Ethereum
- **RLP**: Used by EL but lacks the efficiency features of SSZ (fixed offsets, Merkleization)

SSZ was chosen because:
1. The consensus layer already uses SSZ extensively
2. SSZ provides deterministic encoding essential for consensus
3. SSZ's fixed-offset design enables zero-copy deserialization
4. Merkleization support enables efficient proofs over API data
5. `ProgressiveContainer` ([EIP-7495](./eip-7495.md)) provides forward-compatible schema evolution

### Why ProgressiveContainer?

Regular SSZ containers have a limitation: when fields are added or removed, the Merkle tree shape changes, breaking all existing Merkle proofs. This is problematic for:

- **Light clients**: Would need updates every fork
- **Smart contract verifiers**: Cannot be easily upgraded
- **Hardware wallets**: Have slow update cycles

`ProgressiveContainer` solves this by assigning stable generalized indices to each field position. The `active_fields` bitvector indicates which fields are present, allowing:

1. Fields to be added by extending `active_fields` with trailing `1`s
2. Fields to be deprecated by changing their bit to `0`
3. Merkle proofs to remain valid for unchanged fields across versions

This aligns the Engine API with [EIP-7688](./eip-7688.md) which converts consensus structures to use `ProgressiveContainer`.

### Backwards Compatibility Approach

The content-type negotiation approach allows gradual rollout:
1. Clients can upgrade independently
2. Mixed networks continue to function via JSON fallback
3. Operators can test SSZ encoding before full commitment

### Why not a new transport protocol?

Replacing HTTP/JSON-RPC entirely (e.g., with gRPC or a custom protocol) was considered but rejected:
1. HTTP provides well-understood operational characteristics
2. Existing monitoring, proxying, and debugging tools continue to work
3. The primary overhead is encoding, not transport

### Relationship to Existing SSZ Proposals

[EIP-7807](./eip-7807.md) proposes converting execution block structures to use SSZ natively, including replacing keccak256-based block hashing with SHA256-based `hash_tree_root`. This EIP is complementary: EIP-7807 addresses the data model while this EIP addresses the transport encoding. Full benefits are realized when both are adopted, though this EIP can provide encoding efficiency improvements independently.

The broader SSZ migration effort is tracked in [EIP-7919](./eip-7919.md) (Pureth Meta), which bundles related proposals including SSZ transactions ([EIP-6404](./eip-6404.md)), SSZ withdrawals ([EIP-6465](./eip-6465.md)), and SSZ receipts ([EIP-6466](./eip-6466.md)).

## Backwards Compatibility

This EIP is backwards compatible. Clients implementing this specification:

- MUST continue to support JSON encoding for interoperability
- MUST support the `engine_exchangeCapabilities` negotiation mechanism
- SHOULD prefer SSZ encoding when communicating with SSZ-capable peers

Operators running mixed client versions will experience automatic fallback to JSON encoding. Performance benefits are only realized when both EL and CL clients support SSZ.

## Test Cases

### Encoding Equivalence

For each Engine API method, implementations MUST produce equivalent semantic results whether using JSON or SSZ encoding. Test vectors SHALL be provided demonstrating:

1. A JSON-encoded request/response pair
2. The equivalent SSZ-encoded request/response pair
3. Verification that both produce identical execution results

### Example: NewPayloadV4

JSON encoding (current):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "engine_newPayloadV4",
  "params": [
    {
      "parentHash": "0x1234...",
      "feeRecipient": "0xabcd...",
      ...
    },
    ["0xversioned_hash_1", "0xversioned_hash_2"],
    "0xparent_beacon_root"
  ]
}
```

SSZ encoding (proposed):
```
Content-Type: application/ssz

<binary SSZ encoding of NewPayloadRequest container>
```

The SSZ-encoded payload is approximately 40-60% smaller than the equivalent JSON for typical execution payloads.

## Reference Implementation

A reference implementation demonstrating SSZ encoding for the Engine API is available at:

- Execution layer: [Link to reference EL implementation]
- Consensus layer: [Link to reference CL implementation]

The implementation includes:
1. SSZ schema definitions for all Engine API types
2. Content-type negotiation in the HTTP handler
3. Automatic encoding selection based on peer capabilities
4. Benchmark suite comparing JSON and SSZ performance

### Pseudocode

#### HTTP Handler with Content-Type Negotiation

```python
def handle_engine_request(request: HTTPRequest) -> HTTPResponse:
    content_type = request.headers.get("Content-Type", "application/json")
    accept = request.headers.get("Accept", "application/json")

    if content_type == "application/ssz":
        engine_request = decode_ssz(request.body, EngineRequest)
    elif content_type == "application/json":
        engine_request = decode_json(request.body)
    else:
        return HTTPResponse(status=415, body="Unsupported Media Type")

    result = process_engine_method(engine_request.method, engine_request.params)

    if accept == "application/ssz":
        response_body = encode_ssz(result)
        return HTTPResponse(
            status=200,
            headers={"Content-Type": "application/ssz"},
            body=response_body
        )
    else:
        response_body = encode_json(result)
        return HTTPResponse(
            status=200,
            headers={"Content-Type": "application/json"},
            body=response_body
        )
```

#### SSZ Encoding/Decoding with ProgressiveContainer

```python
def encode_ssz(obj: ProgressiveContainer) -> bytes:
    active_fields = obj.get_active_fields()

    fixed_parts = []
    variable_parts = []
    variable_offsets = []

    current_offset = sum(
        field.fixed_size() if field.is_fixed_size() else 4
        for i, field in enumerate(obj.fields())
        if active_fields[i]
    )

    for i, (name, field) in enumerate(obj.fields()):
        if not active_fields[i]:
            continue

        value = getattr(obj, name)

        if field.is_fixed_size():
            fixed_parts.append(serialize_fixed(value))
        else:
            fixed_parts.append(uint32_to_bytes(current_offset))
            serialized = serialize_variable(value)
            variable_parts.append(serialized)
            current_offset += len(serialized)

    return b''.join(fixed_parts) + b''.join(variable_parts)


def decode_ssz(data: bytes, schema: Type[ProgressiveContainer]) -> ProgressiveContainer:
    active_fields = schema.get_active_fields()
    offset = 0
    values = {}
    variable_offsets = []

    for i, (name, field) in enumerate(schema.fields()):
        if not active_fields[i]:
            values[name] = None
            continue

        if field.is_fixed_size():
            size = field.fixed_size()
            values[name] = deserialize_fixed(data[offset:offset+size], field)
            offset += size
        else:
            var_offset = bytes_to_uint32(data[offset:offset+4])
            variable_offsets.append((name, field, var_offset))
            offset += 4

    for j, (name, field, start) in enumerate(variable_offsets):
        end = variable_offsets[j+1][2] if j+1 < len(variable_offsets) else len(data)
        values[name] = deserialize_variable(data[start:end], field)

    return schema(**values)
```

#### Capability Exchange

```python
SUPPORTED_CAPABILITIES = [
    "engine_newPayloadV4",
    "engine_forkchoiceUpdatedV3",
    "engine_getPayloadV5",
    "engine_getPayloadBodiesByHashV2",
    "engine_getPayloadBodiesByRangeV2",
    "engine_getBlobsV3",
    "sszV1",  # SSZ encoding support
]

def engine_exchangeCapabilities(peer_capabilities: List[str]) -> List[str]:
    return SUPPORTED_CAPABILITIES

def should_use_ssz(peer_capabilities: List[str]) -> bool:
    return "sszV1" in peer_capabilities and "sszV1" in SUPPORTED_CAPABILITIES
```

#### NewPayload with SSZ

```python
def engine_newPayload_ssz(request: NewPayloadRequest) -> PayloadStatusSSZ:
    payload = request.execution_payload
    expected_hashes = request.expected_blob_versioned_hashes
    parent_beacon_root = request.parent_beacon_block_root

    if not verify_block_hash(payload):
        return PayloadStatusSSZ(
            status=PayloadStatus.INVALID_BLOCK_HASH,
            latest_valid_hash=None,
            validation_error="Block hash mismatch"
        )

    if not verify_blob_versioned_hashes(payload, expected_hashes):
        return PayloadStatusSSZ(
            status=PayloadStatus.INVALID,
            latest_valid_hash=payload.parent_hash,
            validation_error="Blob versioned hash mismatch"
        )

    if is_syncing():
        return PayloadStatusSSZ(
            status=PayloadStatus.SYNCING,
            latest_valid_hash=None,
            validation_error=None
        )

    validation_result = execute_payload(payload, parent_beacon_root)

    if validation_result.is_valid:
        return PayloadStatusSSZ(
            status=PayloadStatus.VALID,
            latest_valid_hash=payload.block_hash,
            validation_error=None
        )
    else:
        return PayloadStatusSSZ(
            status=PayloadStatus.INVALID,
            latest_valid_hash=find_latest_valid_hash(payload),
            validation_error=validation_result.error
        )
```

#### Merkleization for Proofs

```python
def compute_merkle_root(container: ProgressiveContainer) -> Bytes32:
    active_fields = container.get_active_fields()
    chunks = []

    for i, (name, field) in enumerate(container.fields()):
        if active_fields[i]:
            value = getattr(container, name)
            chunks.append(hash_tree_root(value))
        else:
            chunks.append(ZERO_HASH)

    chunks = pad_to_power_of_2(chunks)

    while len(chunks) > 1:
        chunks = [
            hash_concat(chunks[i], chunks[i+1])
            for i in range(0, len(chunks), 2)
        ]

    active_fields_root = hash_tree_root(active_fields)
    return hash_concat(chunks[0], active_fields_root)


def generate_merkle_proof(
    container: ProgressiveContainer,
    field_index: int
) -> MerkleProof:
    chunks = [hash_tree_root(getattr(container, f.name)) for f in container.fields()]
    chunks = pad_to_power_of_2(chunks)

    proof = []
    index = field_index

    while len(chunks) > 1:
        sibling_index = index ^ 1
        proof.append(chunks[sibling_index])

        chunks = [
            hash_concat(chunks[i], chunks[i+1])
            for i in range(0, len(chunks), 2)
        ]
        index //= 2

    return MerkleProof(
        leaf=hash_tree_root(getattr(container, container.fields()[field_index].name)),
        proof=proof,
        gindex=compute_gindex(field_index, len(container.fields()))
    )
```

## Security Considerations

### Denial of Service

SSZ deserialization MUST enforce the same size limits as JSON deserialization. The `max_length` parameters in SSZ schemas MUST match the implicit limits in the JSON specification. Implementations MUST reject SSZ payloads exceeding defined maximum sizes before attempting full deserialization.

### Version Confusion

Clients MUST NOT assume SSZ support without explicit capability negotiation. Sending SSZ-encoded requests to a JSON-only endpoint could cause parsing failures or undefined behavior. The `engine_exchangeCapabilities` handshake MUST occur before using SSZ encoding.

### Schema Evolution

Future Engine API versions will require updated SSZ schemas. The use of `ProgressiveContainer` ([EIP-7495](./eip-7495.md)) mitigates breaking changes:

- New fields can be appended without breaking existing Merkle proofs
- Deprecated fields maintain their gindex positions (set to zero in `active_fields`)
- The versioning approach (`newPayloadV4`, `newPayloadV5`, etc.) continues for method signatures

However, implementations MUST still handle schema version mismatches gracefully. A client receiving a `ProgressiveContainer` with unknown active fields SHOULD:
1. Parse known fields normally
2. Skip unknown fields during processing
3. Preserve unknown field data for re-serialization if needed

### Implementation Complexity

Adding SSZ support increases implementation complexity and attack surface. Implementations SHOULD:
- Use well-tested SSZ libraries
- Fuzz test SSZ parsing extensively
- Maintain JSON support as a fallback for debugging

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
