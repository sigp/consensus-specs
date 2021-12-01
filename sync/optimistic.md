# Optimistic sync

This document contains a specification for performing an *optimistic sync* of a
post-merge beacon chain.

## Introduction

A consensus engine requires an execution engine in order to full verify beacon
blocks. However, that doesn't necessitate that both engines verify blocks in lock-step.

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

## Fork Choice

#### `Store`

```python
@dataclass
class Store(object):
    time: uint64
    genesis_time: uint64
    justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    best_justified_checkpoint: Checkpoint
    proposer_boost_root: Root
    blocks: Dict[Root, BeaconBlock] = field(default_factory=dict)
    block_states: Dict[Root, BeaconState] = field(default_factory=dict)
    checkpoint_states: Dict[Checkpoint, BeaconState] = field(default_factory=dict)
    latest_messages: Dict[ValidatorIndex, LatestMessage] = field(default_factory=dict)
	valid_payload_block_hashes: Set[Hash]
```

```python
def process_valid_payload(store: Store, block_root: Root):
	while block_root in store.blocks.keys():
		block = store.blocks[block_root]
		store.valid_payload_block_hashes.add(block.message.execution_payload.block_hash)
		block_root = block.parent_root
```

```python
def process_invalid_payload(store: Store, block: BeaconBlock, latest_valid_hash: Hash):
	invalidated_roots = Set()

	while True:
		if block.message.execution_payload.block_hash == latest_valid_hash:
			process_valid_payload(store, block_root)
			break

		invalidated_roots.add(block_root)
		block_root = block.parent_root

		if block_root not in store.blocks.keys():
			break
		block = store.blocks[block_root]

	while True:
		descendants = [
			block for block in store.blocks
			if block.parent_root in invalidated_roots
				and hash_tree_root(block) not in invalidated_roots
		]
		invalidated_roots += descendants
		if len(descendants) == 0:
			break
```

```python
def on_block(store: Store, signed_block: SignedBeaconBlock) -> None:
    """
    Run ``on_block`` upon receiving a new block.

    A block that is asserted as invalid due to unavailable PoW block may be valid at a later time,
    consider scheduling it for later processing in such case.
    """
    block = signed_block.message
    # Parent block must be known
    assert block.parent_root in store.block_states
    # Make a copy of the state to avoid mutability issues
    pre_state = copy(store.block_states[block.parent_root])
    # Blocks cannot be in the future. If they are, their consideration must be delayed until the are in the past.
    assert get_current_slot(store) >= block.slot

    # Check that block is later than the finalized epoch slot (optimization to reduce calls to get_ancestor)
    finalized_slot = compute_start_slot_at_epoch(store.finalized_checkpoint.epoch)
    assert block.slot > finalized_slot
    # Check block is a descendant of the finalized block at the checkpoint finalized slot
    assert get_ancestor(store, block.parent_root, finalized_slot) == store.finalized_checkpoint.root

    # [New in Optimistic Sync]
	payload = block.body.execution_payload
	payload_state = execution_engine.execute_payload(payload)
	if payload_state.status == "VALID":
		store.valid_payload_block_hashes.add(block.message.execution_payload.block_hash)
		process_valid_payload(block.parent_root)
	elif payload_state.status == "INVALID":
		process_invalid_payload(block.)
		# todo(paul): update valid ancestors.
		# todo(paul): update invalid descendants.

    # Check the block is valid and compute the post-state
    state = pre_state.copy()
    state_transition(state, signed_block, True)

    # [New in Merge]
    if is_merge_transition_block(pre_state, block.body):
        validate_merge_block(block)

    # Add new block to the store
    store.blocks[hash_tree_root(block)] = block
    # Add new state for this block to the store
    store.block_states[hash_tree_root(block)] = state

    # Add proposer score boost if the block is timely
    time_into_slot = (store.time - store.genesis_time) % SECONDS_PER_SLOT
    is_before_attesting_interval = time_into_slot < SECONDS_PER_SLOT // INTERVALS_PER_SLOT
    if get_current_slot(store) == block.slot and is_before_attesting_interval:
        store.proposer_boost_root = hash_tree_root(block)

    # Update justified checkpoint
    if state.current_justified_checkpoint.epoch > store.justified_checkpoint.epoch:
        if state.current_justified_checkpoint.epoch > store.best_justified_checkpoint.epoch:
            store.best_justified_checkpoint = state.current_justified_checkpoint
        if should_update_justified_checkpoint(store, state.current_justified_checkpoint):
            store.justified_checkpoint = state.current_justified_checkpoint

    # Update finalized checkpoint
    if state.finalized_checkpoint.epoch > store.finalized_checkpoint.epoch:
        store.finalized_checkpoint = state.finalized_checkpoint
        store.justified_checkpoint = state.current_justified_checkpoint
```

```python
def filter_block_tree(store: Store, block_root: Root, blocks: Dict[Root, BeaconBlock]) -> bool:
    block = store.blocks[block_root]

	# [New in Optimistic Sync]
	if block.execution_payload.block_hash not in store.valid_payload_block_hashes:
		return False

    children = [
        root for root in store.blocks.keys()
        if store.blocks[root].parent_root == block_root
    ]

    # If any children branches contain expected finalized/justified checkpoints,
    # add to filtered block-tree and signal viability to parent.
    if any(children):
        filter_block_tree_result = [filter_block_tree(store, child, blocks) for child in children]
        if any(filter_block_tree_result):
            blocks[block_root] = block
            return True
        return False

    # If leaf block, check finalized/justified checkpoints as matching latest.
    head_state = store.block_states[block_root]

    correct_justified = (
        store.justified_checkpoint.epoch == GENESIS_EPOCH
        or head_state.current_justified_checkpoint == store.justified_checkpoint
    )
    correct_finalized = (
        store.finalized_checkpoint.epoch == GENESIS_EPOCH
        or head_state.finalized_checkpoint == store.finalized_checkpoint
    )
    # If expected finalized/justified, add to viable block-tree and signal viability to parent.
    if correct_justified and correct_finalized:
        blocks[block_root] = block
        return True

    # Otherwise, branch not viable
    return False
```

```python
def get_latest_attesting_balance(store: Store, root: Root) -> Gwei:
	# [New in Optimistic Sync]
	if block.execution_payload.block_hash not in store.valid_payload_block_hashes:
		return Gwei(0)

    state = store.checkpoint_states[store.justified_checkpoint]
    active_indices = get_active_validator_indices(state, get_current_epoch(state))
    attestation_score = Gwei(sum(
        state.validators[i].effective_balance for i in active_indices
        if (i in store.latest_messages
            and get_ancestor(store, store.latest_messages[i].root, store.blocks[root].slot) == root)
    ))
    proposer_score = Gwei(0)
    if store.proposer_boost_root != Root():
        block = store.blocks[root]
        if get_ancestor(store, root, block.slot) == store.proposer_boost_root:
            num_validators = len(get_active_validator_indices(state, get_current_epoch(state)))
            avg_balance = get_total_active_balance(state) // num_validators
            committee_size = num_validators // SLOTS_PER_EPOCH
            committee_weight = committee_size * avg_balance
            proposer_score = (committee_weight * PROPOSER_SCORE_BOOST) // 100
    return attestation_score + proposer_score
```
