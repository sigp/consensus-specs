# Optimistic Sync -- Beacon Chain

**Notice**: This document is a work-in-progress for researchers and implementers.

## Introduction

This document contains a specification for the *optimistic sync* variation of
the post-merge beacon chain. It extends the (merge
specifications)[specs/merge].

## Beacon chain state transition function

### Execution engine

#### `execute_payload`

```python
@dataclass
class PayloadState:
	status: str
	latest_valid_hash: Hash
```

```python
def execute_payload(self: ExecutionEngine, execution_payload: ExecutionPayload) -> Union[Valid, Invalid, Syncing]:
	"""
	Return ``PayloadState("SYNCING", Hash())`` if the ``execution_payload.parent_hash`` is not known to ``self.execution_state``.
    Return ``"PayloadState("INVALID, latest_valid_hash")"`` if and only if ``execution_payload`` is invalid with respect to ``self.execution_state``.
    Return ``"PayloadState("VALID, payload.block_hash")"`` if and only if ``execution_payload`` is valid with respect to ``self.execution_state``.
	"""
    ...
```

### Block processing

#### Execution payload

##### `process_execution_payload`

```python
def process_execution_payload(state: BeaconState, payload: ExecutionPayload, execution_engine: ExecutionEngine) -> None:
    # Verify consistency of the parent hash with respect to the previous execution payload header
    if is_merge_transition_complete(state):
        assert payload.parent_hash == state.latest_execution_payload_header.block_hash
    # Verify random
    assert payload.random == get_randao_mix(state, get_current_epoch(state))
    # Verify timestamp
    assert payload.timestamp == compute_timestamp_at_slot(state, state.slot)
    # Verify the execution payload is valid
	payload_status = execution_engine.execute_payload(payload).status
    assert payload_status == "VALID" or payload_status == "SYNCING"
    # Cache execution payload header
    state.latest_execution_payload_header = ExecutionPayloadHeader(
        parent_hash=payload.parent_hash,
        fee_recipient=payload.fee_recipient,
        state_root=payload.state_root,
        receipt_root=payload.receipt_root,
        logs_bloom=payload.logs_bloom,
        random=payload.random,
        block_number=payload.block_number,
        gas_limit=payload.gas_limit,
        gas_used=payload.gas_used,
        timestamp=payload.timestamp,
        extra_data=payload.extra_data,
        base_fee_per_gas=payload.base_fee_per_gas,
        block_hash=payload.block_hash,
        transactions_root=hash_tree_root(payload.transactions),
    )
```
