# EIP-7732 -- The Beacon Chain

*Note*: This document is a work-in-progress for researchers and implementers.

<!-- mdformat-toc start --slug=github --no-anchors --maxlevel=6 --minlevel=2 -->

- [Introduction](#introduction)
- [Constants](#constants)
  - [Domain types](#domain-types)
- [Preset](#preset)
  - [Misc](#misc)
  - [Max operations per block](#max-operations-per-block)
- [Containers](#containers)
  - [New containers](#new-containers)
    - [`PayloadAttestationData`](#payloadattestationdata)
    - [`PayloadAttestation`](#payloadattestation)
    - [`PayloadAttestationMessage`](#payloadattestationmessage)
    - [`IndexedPayloadAttestation`](#indexedpayloadattestation)
    - [`ExecutionPayloadEnvelope`](#executionpayloadenvelope)
    - [`SignedExecutionPayloadEnvelope`](#signedexecutionpayloadenvelope)
  - [Modified containers](#modified-containers)
    - [`BeaconBlockBody`](#beaconblockbody)
    - [`ExecutionPayloadHeader`](#executionpayloadheader)
    - [`BeaconState`](#beaconstate)
- [Helper functions](#helper-functions)
  - [Math](#math)
    - [New `bit_floor`](#new-bit_floor)
  - [Misc](#misc-1)
    - [New `remove_flag`](#new-remove_flag)
  - [Predicates](#predicates)
    - [New `is_attestation_same_slot`](#new-is_attestation_same_slot)
    - [New `is_valid_indexed_payload_attestation`](#new-is_valid_indexed_payload_attestation)
    - [New `is_parent_block_full`](#new-is_parent_block_full)
  - [Beacon State accessors](#beacon-state-accessors)
    - [New `get_attestation_participation_flag_indices`](#new-get_attestation_participation_flag_indices)
    - [New `get_ptc`](#new-get_ptc)
    - [New `get_payload_attesting_indices`](#new-get_payload_attesting_indices)
    - [New `get_indexed_payload_attestation`](#new-get_indexed_payload_attestation)
- [Beacon chain state transition function](#beacon-chain-state-transition-function)
  - [Modified `process_slot`](#modified-process_slot)
  - [Epoch processing](#epoch-processing)
  - [Block processing](#block-processing)
    - [Modified `process_withdrawals`](#modified-process_withdrawals)
    - [Operations](#operations)
      - [Modified `process_operations`](#modified-process_operations)
      - [Attestations](#attestations)
        - [Modified `process_attestation`](#modified-process_attestation)
      - [Payload Attestations](#payload-attestations)
        - [New `process_payload_attestation`](#new-process_payload_attestation)
    - [Modified `is_merge_transition_complete`](#modified-is_merge_transition_complete)
    - [Modified `validate_merge_block`](#modified-validate_merge_block)
  - [Execution payload processing](#execution-payload-processing)
    - [New `verify_execution_payload_envelope_signature`](#new-verify_execution_payload_envelope_signature)
    - [New `process_execution_payload`](#new-process_execution_payload)

<!-- mdformat-toc end -->

## Introduction

This is the beacon chain specification for separating the execution payload from
the beacon block.

*Note*: This specification is built upon
[Electra](../../electra/beacon-chain.md) and is under active development.

This feature implements a simplified execution-consensus separation through
pipelining. The slot is divided in **four** intervals. Proposers submit beacon
blocks containing `parent_execution_payload_header` at the beginning of the
slot, and later broadcast execution payload envelopes containing the actual
execution data. Validators selected to be members of the new **Payload
Timeliness Committee** (PTC) attest to the presence and timeliness of execution
payloads.

At any given slot, the status of the blockchain's head may be either

- A block from a previous slot (e.g. the current slot's proposer did not submit
  its block).
- An *empty* block from the current slot (e.g. the proposer submitted a timely
  beacon block, but did not reveal the execution payload on time).
- A full block for the current slot (both the beacon block and execution payload
  were revealed on time).

## Constants

### Domain types

| Name                  | Value                      |
| --------------------- | -------------------------- |
| `DOMAIN_PTC_ATTESTER` | `DomainType('0x0C000000')` |

## Preset

### Misc

| Name       | Value                 |
| ---------- | --------------------- |
| `PTC_SIZE` | `uint64(2**9)` (=512) |

### Max operations per block

| Name                       | Value |
| -------------------------- | ----- |
| `MAX_PAYLOAD_ATTESTATIONS` | `4`   |

## Containers

### New containers

#### `PayloadAttestationData`

```python
class PayloadAttestationData(Container):
    beacon_block_root: Root
    slot: Slot
    payload_present: boolean
```

#### `PayloadAttestation`

```python
class PayloadAttestation(Container):
    aggregation_bits: Bitvector[PTC_SIZE]
    data: PayloadAttestationData
    signature: BLSSignature
```

#### `PayloadAttestationMessage`

```python
class PayloadAttestationMessage(Container):
    validator_index: ValidatorIndex
    data: PayloadAttestationData
    signature: BLSSignature
```

#### `IndexedPayloadAttestation`

```python
class IndexedPayloadAttestation(Container):
    attesting_indices: List[ValidatorIndex, PTC_SIZE]
    data: PayloadAttestationData
    signature: BLSSignature
```

#### `ExecutionPayloadEnvelope`

```python
class ExecutionPayloadEnvelope(Container):
    payload: ExecutionPayload
    execution_requests: ExecutionRequests
    proposer_index: ValidatorIndex
    beacon_block_root: Root
    slot: Slot
    blob_kzg_commitments: List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]
    state_root: Root
```

#### `SignedExecutionPayloadEnvelope`

```python
class SignedExecutionPayloadEnvelope(Container):
    message: ExecutionPayloadEnvelope
    signature: BLSSignature
```

### Modified containers

#### `BeaconBlockBody`

*Note*: The `BeaconBlockBody` container is modified to contain a
`SignedExecutionPayloadHeader`. The containers `BeaconBlock` and
`SignedBeaconBlock` are modified indirectly. The field `execution_requests` is
removed from the beacon block body and moved into the signed execution payload
envelope.

```python
class BeaconBlockBody(Container):
    randao_reveal: BLSSignature
    eth1_data: Eth1Data
    graffiti: Bytes32
    proposer_slashings: List[ProposerSlashing, MAX_PROPOSER_SLASHINGS]
    attester_slashings: List[AttesterSlashing, MAX_ATTESTER_SLASHINGS_ELECTRA]
    attestations: List[Attestation, MAX_ATTESTATIONS_ELECTRA]
    deposits: List[Deposit, MAX_DEPOSITS]
    voluntary_exits: List[SignedVoluntaryExit, MAX_VOLUNTARY_EXITS]
    sync_aggregate: SyncAggregate
    # [Modified in EIP7732]
    # Removed `execution_payload`
    bls_to_execution_changes: List[SignedBLSToExecutionChange, MAX_BLS_TO_EXECUTION_CHANGES]
    # [Modified in EIP7732]
    # Removed `blob_kzg_commitments`
    # [Modified in EIP7732]
    # Removed `execution_requests`
    # [New in EIP7732]
    parent_execution_payload_header: ExecutionPayloadHeader
    # [New in EIP7732]
    payload_attestations: List[PayloadAttestation, MAX_PAYLOAD_ATTESTATIONS]
```

#### `ExecutionPayloadHeader`

*Note*: The `ExecutionPayloadHeader` is modified to only contain the block hash
of the committed `ExecutionPayload` in addition to the builder's payment
information, gas limit and KZG commitments root to verify the inclusion proofs.

```python
class ExecutionPayloadHeader(Container):
    parent_block_hash: Hash32
    parent_block_root: Root
    block_hash: Hash32
    fee_recipient: ExecutionAddress
    gas_limit: uint64
    builder_index: ValidatorIndex
    slot: Slot
    value: Gwei
    blob_kzg_commitments_root: Root
```

#### `BeaconState`

*Note*: The `BeaconState` is modified to track the last withdrawals honored in
the CL. The `latest_execution_payload_header` is modified semantically to refer
to the header of the most recent execution payload envelope that was processed,
regardless of execution success or failure. This header contains only
pre-execution fields and represents the commitment made, not the execution
results. Another addition is to track the last committed block hash and the last
slot that was full, that is in which there were both consensus and execution
blocks included.

```python
class BeaconState(Container):
    genesis_time: uint64
    genesis_validators_root: Root
    slot: Slot
    fork: Fork
    latest_block_header: BeaconBlockHeader
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    state_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    historical_roots: List[Root, HISTORICAL_ROOTS_LIMIT]
    eth1_data: Eth1Data
    eth1_data_votes: List[Eth1Data, EPOCHS_PER_ETH1_VOTING_PERIOD * SLOTS_PER_EPOCH]
    eth1_deposit_index: uint64
    validators: List[Validator, VALIDATOR_REGISTRY_LIMIT]
    balances: List[Gwei, VALIDATOR_REGISTRY_LIMIT]
    randao_mixes: Vector[Bytes32, EPOCHS_PER_HISTORICAL_VECTOR]
    slashings: Vector[Gwei, EPOCHS_PER_SLASHINGS_VECTOR]
    previous_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    current_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    justification_bits: Bitvector[JUSTIFICATION_BITS_LENGTH]
    previous_justified_checkpoint: Checkpoint
    current_justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    inactivity_scores: List[uint64, VALIDATOR_REGISTRY_LIMIT]
    current_sync_committee: SyncCommittee
    next_sync_committee: SyncCommittee
    latest_execution_payload_header: ExecutionPayloadHeader
    next_withdrawal_index: WithdrawalIndex
    next_withdrawal_validator_index: ValidatorIndex
    historical_summaries: List[HistoricalSummary, HISTORICAL_ROOTS_LIMIT]
    deposit_requests_start_index: uint64
    deposit_balance_to_consume: Gwei
    exit_balance_to_consume: Gwei
    earliest_exit_epoch: Epoch
    consolidation_balance_to_consume: Gwei
    earliest_consolidation_epoch: Epoch
    pending_deposits: List[PendingDeposit, PENDING_DEPOSITS_LIMIT]
    pending_partial_withdrawals: List[PendingPartialWithdrawal, PENDING_PARTIAL_WITHDRAWALS_LIMIT]
    pending_consolidations: List[PendingConsolidation, PENDING_CONSOLIDATIONS_LIMIT]
    # [New in EIP7732]
    execution_payload_availability: Bitvector[SLOTS_PER_HISTORICAL_ROOT]
    # [New in EIP7732]
    latest_block_hash: Hash32
    # [New in EIP7732]
    latest_full_slot: Slot
    # [New in EIP7732]
    latest_withdrawals_root: Root
```

## Helper functions

### Math

#### New `bit_floor`

```python
def bit_floor(n: uint64) -> uint64:
    """
    if ``n`` is not zero, returns the largest power of `2` that is not greater than `n`.
    """
    if n == 0:
        return 0
    return uint64(1) << (n.bit_length() - 1)
```

### Misc

#### New `remove_flag`

```python
def remove_flag(flags: ParticipationFlags, flag_index: int) -> ParticipationFlags:
    flag = ParticipationFlags(2**flag_index)
    return flags & ~flag
```

### Predicates

#### New `is_attestation_same_slot`

```python
def is_attestation_same_slot(state: BeaconState, data: AttestationData) -> bool:
    """
    Checks if the attestation was for the block proposed at the attestation slot
    """
    if data.slot == 0:
        return True
    is_matching_blockroot = data.beacon_block_root == get_block_root_at_slot(state, Slot(data.slot))
    is_current_blockroot = data.beacon_block_root != get_block_root_at_slot(
        state, Slot(data.slot - 1)
    )
    return is_matching_blockroot and is_current_blockroot
```

#### New `is_valid_indexed_payload_attestation`

```python
def is_valid_indexed_payload_attestation(
    state: BeaconState, indexed_payload_attestation: IndexedPayloadAttestation
) -> bool:
    """
    Check if ``indexed_payload_attestation`` is not empty, has sorted and unique indices and has
    a valid aggregate signature.
    """
    # Verify indices are sorted and unique
    indices = indexed_payload_attestation.attesting_indices
    if len(indices) == 0 or not indices == sorted(set(indices)):
        return False

    # Verify aggregate signature
    pubkeys = [state.validators[i].pubkey for i in indices]
    domain = get_domain(state, DOMAIN_PTC_ATTESTER, None)
    signing_root = compute_signing_root(indexed_payload_attestation.data, domain)
    return bls.FastAggregateVerify(pubkeys, signing_root, indexed_payload_attestation.signature)
```

#### New `is_parent_block_full`

This function returns true if the last committed payload header was fulfilled
with a payload, this can only happen when both beacon block and payload were
present. This function must be called on a beacon state before processing the
execution payload header in the block.

```python
def is_parent_block_full(state: BeaconState) -> bool:
    return state.latest_execution_payload_header.block_hash == state.latest_block_hash
```

### Beacon State accessors

#### New `get_attestation_participation_flag_indices`

```python
def get_attestation_participation_flag_indices(
    state: BeaconState, data: AttestationData, inclusion_delay: uint64
) -> Sequence[int]:
    """
    Return the flag indices that are satisfied by an attestation.
    """
    if data.target.epoch == get_current_epoch(state):
        justified_checkpoint = state.current_justified_checkpoint
    else:
        justified_checkpoint = state.previous_justified_checkpoint

    # Matching roots
    is_matching_source = data.source == justified_checkpoint
    is_matching_target = is_matching_source and data.target.root == get_block_root(
        state, data.target.epoch
    )
    is_matching_blockroot = is_matching_target and data.beacon_block_root == get_block_root_at_slot(
        state, Slot(data.slot)
    )
    is_matching_payload = False
    if is_attestation_same_slot(state, data):
        assert data.index == 0
        is_matching_payload = True
    else:
        is_matching_payload = (
            data.index
            == state.execution_payload_availability[data.slot % SLOTS_PER_HISTORICAL_ROOT]
        )
    is_matching_head = is_matching_blockroot and is_matching_payload

    assert is_matching_source

    participation_flag_indices = []
    if is_matching_source and inclusion_delay <= integer_squareroot(SLOTS_PER_EPOCH):
        participation_flag_indices.append(TIMELY_SOURCE_FLAG_INDEX)
    if is_matching_target and inclusion_delay <= SLOTS_PER_EPOCH:
        participation_flag_indices.append(TIMELY_TARGET_FLAG_INDEX)
    if is_matching_head and inclusion_delay == MIN_ATTESTATION_INCLUSION_DELAY:
        participation_flag_indices.append(TIMELY_HEAD_FLAG_INDEX)

    return participation_flag_indices
```

#### New `get_ptc`

```python
def get_ptc(state: BeaconState, slot: Slot) -> Vector[ValidatorIndex, PTC_SIZE]:
    """
    Get the payload timeliness committee for the given ``slot``
    """
    epoch = compute_epoch_at_slot(slot)
    committees_per_slot = bit_floor(min(get_committee_count_per_slot(state, epoch), PTC_SIZE))
    members_per_committee = PTC_SIZE // committees_per_slot

    validator_indices: List[ValidatorIndex] = []
    for idx in range(committees_per_slot):
        beacon_committee = get_beacon_committee(state, slot, CommitteeIndex(idx))
        validator_indices += beacon_committee[:members_per_committee]
    return validator_indices
```

#### New `get_payload_attesting_indices`

```python
def get_payload_attesting_indices(
    state: BeaconState, slot: Slot, payload_attestation: PayloadAttestation
) -> Set[ValidatorIndex]:
    """
    Return the set of attesting indices corresponding to ``payload_attestation``.
    """
    ptc = get_ptc(state, slot)
    return set(index for i, index in enumerate(ptc) if payload_attestation.aggregation_bits[i])
```

#### New `get_indexed_payload_attestation`

```python
def get_indexed_payload_attestation(
    state: BeaconState, slot: Slot, payload_attestation: PayloadAttestation
) -> IndexedPayloadAttestation:
    """
    Return the indexed payload attestation corresponding to ``payload_attestation``.
    """
    attesting_indices = get_payload_attesting_indices(state, slot, payload_attestation)

    return IndexedPayloadAttestation(
        attesting_indices=sorted(attesting_indices),
        data=payload_attestation.data,
        signature=payload_attestation.signature,
    )
```

## Beacon chain state transition function

*Note*: state transition is fundamentally modified in EIP-7732. The full state
transition is broken in two parts, first importing a signed block and then
importing an execution payload.

The post-state corresponding to a pre-state `state` and a signed beacon block
`signed_block` is defined as `state_transition(state, signed_block)`. State
transitions that trigger an unhandled exception (e.g. a failed `assert` or an
out-of-range list access) are considered invalid. State transitions that cause a
`uint64` overflow or underflow are also considered invalid.

The post-state corresponding to a pre-state `state` and a signed execution
payload envelope `signed_envelope` is defined as
`process_execution_payload(state, signed_envelope)`. State transitions that
trigger an unhandled exception (e.g. a failed `assert` or an out-of-range list
access) are considered invalid. State transitions that cause an `uint64`
overflow or underflow are also considered invalid.

### Modified `process_slot`

*Note*: `process_slot` is modified to unset the payload availability bit.

```python
def process_slot(state: BeaconState) -> None:
    # Cache state root
    previous_state_root = hash_tree_root(state)
    state.state_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT] = previous_state_root
    # Cache latest block header state root
    if state.latest_block_header.state_root == Bytes32():
        state.latest_block_header.state_root = previous_state_root
    # Cache block root
    previous_block_root = hash_tree_root(state.latest_block_header)
    state.block_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT] = previous_block_root
    # [New in EIP7732]
    # Unset the next payload availability
    state.execution_payload_availability[(state.slot + 1) % SLOTS_PER_HISTORICAL_ROOT] = 0b0
```

### Epoch processing

### Block processing

*Note*: The function `process_block` is modified to call the new and updated
functions and removes the call to `process_execution_payload`.

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    process_block_header(state, block)
    # [Modified in EIP7732]
    process_withdrawals(state)
    # [Modified in EIP7732]
    # Removed `process_execution_payload`
    process_randao(state, block.body)
    process_eth1_data(state, block.body)
    # [Modified in EIP7732]
    process_operations(state, block.body)
    process_sync_aggregate(state, block.body.sync_aggregate)
```

#### Modified `process_withdrawals`

*Note*: This is modified to take only the state as parameter. Withdrawals are
deterministic given the beacon state, any execution payload that has the
corresponding block as parent beacon block is required to honor these
withdrawals in the execution layer. This function must be called before
`process_execution_payload` as this latter function affects validator balances.

```python
def process_withdrawals(state: BeaconState) -> None:
    # return early if the parent block was empty
    if not is_parent_block_full(state):
        return

    withdrawals, processed_partial_withdrawals_count = get_expected_withdrawals(state)
    withdrawals_list = List[Withdrawal, MAX_WITHDRAWALS_PER_PAYLOAD](withdrawals)
    state.latest_withdrawals_root = hash_tree_root(withdrawals_list)
    for withdrawal in withdrawals:
        decrease_balance(state, withdrawal.validator_index, withdrawal.amount)

    # Update pending partial withdrawals
    state.pending_partial_withdrawals = state.pending_partial_withdrawals[
        processed_partial_withdrawals_count:
    ]

    # Update the next withdrawal index if this block contained withdrawals
    if len(withdrawals) != 0:
        latest_withdrawal = withdrawals[-1]
        state.next_withdrawal_index = WithdrawalIndex(latest_withdrawal.index + 1)

    # Update the next validator index to start the next withdrawal sweep
    if len(withdrawals) == MAX_WITHDRAWALS_PER_PAYLOAD:
        # Next sweep starts after the latest withdrawal's validator index
        next_validator_index = ValidatorIndex(
            (withdrawals[-1].validator_index + 1) % len(state.validators)
        )
        state.next_withdrawal_validator_index = next_validator_index
    else:
        # Advance sweep by the max length of the sweep if there was not a full set of withdrawals
        next_index = state.next_withdrawal_validator_index + MAX_VALIDATORS_PER_WITHDRAWALS_SWEEP
        next_validator_index = ValidatorIndex(next_index % len(state.validators))
        state.next_withdrawal_validator_index = next_validator_index
```

#### Operations

##### Modified `process_operations`

*Note*: `process_operations` is modified to process PTC attestations and removes
calls to `process_deposit_request`, `process_withdrawal_request`, and
`process_consolidation_request`.

```python
def process_operations(state: BeaconState, body: BeaconBlockBody) -> None:
    # Verify that outstanding deposits are processed up to the maximum number of deposits
    assert len(body.deposits) == min(
        MAX_DEPOSITS, state.eth1_data.deposit_count - state.eth1_deposit_index
    )

    def for_ops(operations: Sequence[Any], fn: Callable[[BeaconState, Any], None]) -> None:
        for operation in operations:
            fn(state, operation)

    for_ops(body.proposer_slashings, process_proposer_slashing)
    for_ops(body.attester_slashings, process_attester_slashing)
    # [Modified in EIP7732]
    for_ops(body.attestations, process_attestation)
    for_ops(body.deposits, process_deposit)
    for_ops(body.voluntary_exits, process_voluntary_exit)
    for_ops(body.bls_to_execution_changes, process_bls_to_execution_change)
    # [Modified in EIP7732]
    # Removed `process_deposit_request`
    # [Modified in EIP7732]
    # Removed `process_withdrawal_request`
    # [Modified in EIP7732]
    # Removed `process_consolidation_request`
    # [New in EIP7732]
    for_ops(body.payload_attestations, process_payload_attestation)
```

##### Attestations

###### Modified `process_attestation`

*Note*: The function is modified to use the `index` field in the
`AttestationData` to signal the payload availability.

```python
def process_attestation(state: BeaconState, attestation: Attestation) -> None:
    data = attestation.data
    assert data.target.epoch in (get_previous_epoch(state), get_current_epoch(state))
    assert data.target.epoch == compute_epoch_at_slot(data.slot)
    assert data.slot + MIN_ATTESTATION_INCLUSION_DELAY <= state.slot

    # [Modified in EIP7732]
    assert data.index < 2
    committee_indices = get_committee_indices(attestation.committee_bits)
    committee_offset = 0
    for committee_index in committee_indices:
        assert committee_index < get_committee_count_per_slot(state, data.target.epoch)
        committee = get_beacon_committee(state, data.slot, committee_index)
        committee_attesters = set(
            attester_index
            for i, attester_index in enumerate(committee)
            if attestation.aggregation_bits[committee_offset + i]
        )
        assert len(committee_attesters) > 0
        committee_offset += len(committee)

    # Bitfield length matches total number of participants
    assert len(attestation.aggregation_bits) == committee_offset

    # Participation flag indices
    participation_flag_indices = get_attestation_participation_flag_indices(
        state, data, state.slot - data.slot
    )

    # Verify signature
    assert is_valid_indexed_attestation(state, get_indexed_attestation(state, attestation))

    # Update epoch participation flags
    if data.target.epoch == get_current_epoch(state):
        epoch_participation = state.current_epoch_participation
    else:
        epoch_participation = state.previous_epoch_participation

    proposer_reward_numerator = 0
    for index in get_attesting_indices(state, attestation):
        for flag_index, weight in enumerate(PARTICIPATION_FLAG_WEIGHTS):
            if flag_index in participation_flag_indices and not has_flag(
                epoch_participation[index], flag_index
            ):
                epoch_participation[index] = add_flag(epoch_participation[index], flag_index)
                proposer_reward_numerator += get_base_reward(state, index) * weight

    # Reward proposer
    proposer_reward_denominator = (
        (WEIGHT_DENOMINATOR - PROPOSER_WEIGHT) * WEIGHT_DENOMINATOR // PROPOSER_WEIGHT
    )
    proposer_reward = Gwei(proposer_reward_numerator // proposer_reward_denominator)
    increase_balance(state, get_beacon_proposer_index(state), proposer_reward)
```

##### Payload Attestations

###### New `process_payload_attestation`

```python
def process_payload_attestation(
    state: BeaconState, payload_attestation: PayloadAttestation
) -> None:
    # Check that the attestation is for the parent beacon block
    data = payload_attestation.data
    assert data.beacon_block_root == state.latest_block_header.parent_root
    # Check that the attestation is for the previous slot
    assert data.slot + 1 == state.slot

    # Verify signature
    indexed_payload_attestation = get_indexed_payload_attestation(
        state, data.slot, payload_attestation
    )
    assert is_valid_indexed_payload_attestation(state, indexed_payload_attestation)
```

#### Modified `is_merge_transition_complete`

`is_merge_transition_complete` is modified only for testing purposes to add the
blob kzg commitments root for an empty list

```python
def is_merge_transition_complete(state: BeaconState) -> bool:
    header = ExecutionPayloadHeader()
    kzgs = List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]()
    header.blob_kzg_commitments_root = kzgs.hash_tree_root()

    return state.latest_execution_payload_header != header
```

#### Modified `validate_merge_block`

`validate_merge_block` is modified to use the new
`signed_execution_payload_header` message in the Beacon Block Body

```python
def validate_merge_block(block: BeaconBlock) -> None:
    """
    Check the parent PoW block of execution payload is a valid terminal PoW block.

    Note: Unavailable PoW block(s) may later become available,
    and a client software MAY delay a call to ``validate_merge_block``
    until the PoW block(s) become available.
    """
    if TERMINAL_BLOCK_HASH != Hash32():
        # If `TERMINAL_BLOCK_HASH` is used as an override, the activation epoch must be reached.
        assert compute_epoch_at_slot(block.slot) >= TERMINAL_BLOCK_HASH_ACTIVATION_EPOCH
        assert block.body.parent_execution_payload_header.block_hash == TERMINAL_BLOCK_HASH
        return

    # [Modified in EIP7732]
    pow_block = get_pow_block(block.body.parent_execution_payload_header.block_hash)
    # Check if `pow_block` is available
    assert pow_block is not None
    pow_parent = get_pow_block(pow_block.parent_hash)
    # Check if `pow_parent` is available
    assert pow_parent is not None
    # Check if `pow_block` is a valid terminal PoW block
    assert is_valid_terminal_pow_block(pow_block, pow_parent)
```

### Execution payload processing

#### New `verify_execution_payload_envelope_signature`

```python
def verify_execution_payload_envelope_signature(
    state: BeaconState, signed_envelope: SignedExecutionPayloadEnvelope
) -> bool:
    proposer = state.validators[signed_envelope.message.proposer_index]
    signing_root = compute_signing_root(
        signed_envelope.message, get_domain(state, DOMAIN_BEACON_PROPOSER)
    )
    return bls.Verify(proposer.pubkey, signing_root, signed_envelope.signature)
```

#### New `process_execution_payload`

*Note*: `process_execution_payload` is now an independent check in state
transition. It is called when importing a signed execution payload envelope
created by the proposer of the current slot.

```python
def process_execution_payload(
    state: BeaconState,
    signed_envelope: SignedExecutionPayloadEnvelope,
    execution_engine: ExecutionEngine,
    verify: bool = True,
) -> None:
    # Verify signature
    if verify:
        assert verify_execution_payload_envelope_signature(state, signed_envelope)
    envelope = signed_envelope.message
    payload = envelope.payload
    # Cache latest block header state root
    previous_state_root = hash_tree_root(state)
    if state.latest_block_header.state_root == Root():
        state.latest_block_header.state_root = previous_state_root

    # Verify consistency with the beacon block
    assert envelope.beacon_block_root == hash_tree_root(state.latest_block_header)
    assert envelope.slot == state.slot

    # Verify the proposer index matches
    assert envelope.proposer_index == get_beacon_proposer_index(state)

    # Verify the withdrawals root
    assert hash_tree_root(payload.withdrawals) == state.latest_withdrawals_root
    # Verify consistency of the parent hash with respect to the previous execution payload
    assert payload.parent_hash == state.latest_block_hash
    # Verify prev_randao
    assert payload.prev_randao == get_randao_mix(state, get_current_epoch(state))
    # Verify timestamp
    assert payload.timestamp == compute_time_at_slot(state, state.slot)
    # Verify commitments are under limit
    assert len(envelope.blob_kzg_commitments) <= MAX_BLOBS_PER_BLOCK

    # Update latest execution payload header BEFORE execution validation
    # This ensures it's set even if execution fails, using only pre-execution fields
    state.latest_execution_payload_header = ExecutionPayloadHeader(
        parent_block_hash=payload.parent_hash,
        parent_block_root=state.latest_block_header.parent_root,
        block_hash=payload.block_hash,
        fee_recipient=payload.fee_recipient,
        gas_limit=payload.gas_limit,
        builder_index=envelope.proposer_index,  # Using proposer as builder in this simplified version
        slot=envelope.slot,
        value=Gwei(0),  # No payment mechanism in this simplified version
        blob_kzg_commitments_root=hash_tree_root(envelope.blob_kzg_commitments),
    )

    # Verify the execution payload is valid
    versioned_hashes = [
        kzg_commitment_to_versioned_hash(commitment) for commitment in envelope.blob_kzg_commitments
    ]
    requests = envelope.execution_requests
    assert execution_engine.verify_and_notify_new_payload(
        NewPayloadRequest(
            execution_payload=payload,
            versioned_hashes=versioned_hashes,
            parent_beacon_block_root=state.latest_block_header.parent_root,
            execution_requests=requests,
        )
    )

    def for_ops(operations: Sequence[Any], fn: Callable[[BeaconState, Any], None]) -> None:
        for operation in operations:
            fn(state, operation)

    for_ops(requests.deposits, process_deposit_request)
    for_ops(requests.withdrawals, process_withdrawal_request)
    for_ops(requests.consolidations, process_consolidation_request)

    # Cache the execution payload hash
    state.execution_payload_availability[state.slot % SLOTS_PER_HISTORICAL_ROOT] = 0b1
    state.latest_block_hash = payload.block_hash
    state.latest_full_slot = state.slot

    # Verify the state root
    if verify:
        assert envelope.state_root == hash_tree_root(state)
```
