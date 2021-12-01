# Optimistic Sync -- Honest Validator

**Notice**: This document is a work-in-progress for researchers and implementers.

## Introduction

This document contains a specification for the *optimistic sync* variation of
the post-merge beacon chain. It extends the (merge
specifications)[specs/merge].

## Validator assignments

If the `head` block does not have a `"VALID"` execution payload (i.e., `assert
hash_tree_root(head) in store.valid_payload_block_hashes` where `store` is as
used in [fork choice](./fork_choice.md#Store)), then validators MUST refuse
to sign the following messages:

- `SignedBeaconBlock`
- `Attestation`
- `SignedAggregatedAndProof`
- `SyncCommitteeMessage`
- `SyncCommitteeContribution`
- `ContributionAndProof`
- `SignedContributionAndProof`
