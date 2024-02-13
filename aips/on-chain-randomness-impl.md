---
aip: (this is determined by the AIP Manager, leave it empty when drafting)
title: Implementation of instant on-chain randomness
author: Alin Tomescu (alin@aptoslabs), Zhuolun "Daniel" Xiang <daniel@aptoslabs.com>
discussions-to (*optional): <a url pointing to the official discussion thread>
Status: Draft
last-call-end-date (*optional): <mm/dd/yyyy the last date to leave feedbacks and reviews>
type: Standard (Core, Networking, Framework)
created: 02/12/2024
updated (*optional): <mm/dd/yyyy>
requires (*optional): <AIP number(s)>
---

# AIP-X - Implementation of instant on-chain randomness

## Summary

 > Summarize in 3-5 sentences what is the problem we’re solving for and how are we solving for it

This AIP describes the implementation of the randomness API from AIP-41[^aip-41], focusing on the cryptographic constructions and their efficient integration into our consensus protocol. Specifically, we describe how we use a weighted distributed key generation (DKG) protocol in a PoS setting to 

### Goals

 > What are the goals and what is in scope? Any metrics?
 > Discuss the business impact and business value this change would impact.

The goal is to provide Move smart contracts with access to (1) **instant**[^aip-41], (2) **unbiasable** and (3) **unpredictable** randomness w.r.t. to any minority stake of the validators.

At the same time, we must do this without imposing a high penalty on the blockchain’s latency or throughput.

### Out of Scope

 > What are we committing to not doing and why are they scoped out?

...

## Motivation

 > Describe the impetus for this change. What does it accomplish?
 > What might occur if we do not accept this proposal?

As already described in [AIP-41](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md#motivation), the need for on-chain randomness arises in many applications, such as decentralized games, raffles, randomized NFTs, randomized airdrops, and more. Plus, many blockchain applications will benefit from randomness in terms of fairness, security, and functionality.

## Impact

 > Which audiences are impacted by this change? What type of action does the audience need to take?

On-chain randomness interacts with & impacts our consensus protocol. The two protocols must be carefully co-designed to maintain performance, especially in terms of latency.

## Alternative solutions

 > Explain why you submitted this proposal specifically over alternative solutions. Why is this the best possible outcome?

**VDFs.** Current techniques based on verifiable delay functions (VDFs) introduce high latencies in the worst paths (TODO: cite bicorn). This does not meet the [goals](#Goals) described above.

**Interactive VSS and VRFs**. A different design would use a DKG based on interactive VSS to deal a secret amongst our validators. Unfortunately, this design requires that validators be 

**Other insecure approaches.**

## Specification

 > How will we solve the problem? Describe in detail precisely how this proposal should be implemented. Include proposed design principles that should be followed in implementing this feature. Make the proposal specific enough to allow others to build upon it and perhaps even derive competing implementations.

### Stake Rounding

Aptos Roll relies on rounding the stakes of each validator into a smaller *weight.* This is very good for achieving practical performance but has implications on secrecy and availability. Specifically, any rounding scheme will lead to **rounding errors**: i.e., some players might receive more secret shares than they deserve and other players might receive less. As a result, this effectively turns the threshold secret sharing scheme into a *ramp secret sharing scheme,* which has a different secrecy threshold and a different reconstruction threshold. Typically, the reconstruction threshold is higher. As a consequence, for liveness, the mechanism needs to ensure any 66% or more of the stake can reconstruct while secrecy holds against any 33% or less of the stake. To minimize the impact of MEV attacks, we chose to make the secrecy threshold even higher, setting it to 50% while still keeping the reconstruction threshold below 66%, to handle a 33% adversary.

Below we give a brief description of the rounding interface and its guarantees. More technical details of the rounding algorithm can be found in our [paper](https://eprint.iacr.org/2024/198).

#### Rounding interface

- **Inputs**:
  - `validator_stakes`: The stake distribution of the validators.
  - `secrecy_threshold_in_stake_ratio`: Any subset of validators with stake ratio ≤ this value cannot reveal the secret (randomness). Aptos uses 50% for this value in production.
  - `reconstruct_threshold_in_stake_ratio`: Any subset of validators with stake ratio ≥ this value can always reveal the secret (randomness). Aptos uses 66% for this value in production.
- **Output**:
  - `validator_weights`: The weight distribution assigned to the validators after rounding.
  - `reconstruct_threshold_in_weights`: Any subset of validators whose weights sum is ≥ this value can always reveal the secret (randomness).

#### Guarantees

When validators are assigned with weights according to `validator_weights`, any subset of validators that have a weight sum `≥ reconstruct_threshold_in_weights` must have stake ratio in the range `(secrecy_threshold_in_stake_ratio, reconstruct_threshold_in_stake_ratio]`. 

#### Rationale

Since in production `secrecy_threshold_in_stake_ratio=0.5`  and `reconstruct_threshold_in_stake_ratio=0.66`, the subset of validators that can reveal the secret (randomness) must have a stake ratio in `(0.5, 0.66]`. Due to the Proof-of-Stake assumption where the adversary owns at most `1/3` of the stake, the adversary cannot reveal the secret by itself (since `1/3<0.5`), and the honest validators can always reveal the secret by themselves (since `2/3>0.66`).

### System design

On-chain randomness introduces several changes to the current Aptos blockchain system, including running a wDKG for every epoch change, modifying the existing reconfiguration process, and generating a randomness seed for every block. This section describes the above system changes. More technical details of the design can be found in our [paper](https://eprint.iacr.org/2024/198).

#### Background on Aptos Blockchain

Aptos is a proof-of-stake (PoS) blockchain with a consensus algorithm that operates in periodic two-hour intervals known as *epochs*. The set of validators and their stake distribution remain fixed within each epoch, and can change across epoch boundaries. The validators of the next epoch do not come online until the new epoch starts.

The blockchain also decouples consensus (i.e., currently a BFT consensus protocol named [Jolteon](https://arxiv.org/abs/2106.10362)) from execution (i.e., an optimistic concurrency control execution engine named [BlockSTM](https://medium.com/aptoslabs/block-stm-how-we-execute-over-160k-transactions-per-second-on-the-aptos-blockchain-3b003657e4ba)), where each block is first finalized by consensus and then executed to update the blockchain state.

*This consensus-execution decoupling is especially important for on-chain randomness, because it allows the network to first commit to an ordering of transactions before computing and revealing randomness later on, which ensures the randomness is unbiasable and unpredictable.*

#### Weighted Distributed Key Generation Using wPVSS

The goal of wDKG is to have validators jointly generate a *shared secret*, which will be used to compute the randomness seed for every block. Aptos Roll runs a *non-interactive* wDKG *before* every epoch change. The design rationale is the following.

- Why use weighted DKG instead of unweighted DKG? As previously mentioned, the security of the on-chain randomness assumes proof-of-stake, therefore the DKG needs to incorporate the weights of the validators when generating the shared secret for randomness generation.
- Why running wDKG before epoch change instead of after? If the wDKG starts after the epoch change, then the validators do not have the shared secret setup for the randomness generation when epoch starts. Until the wDKG finishes, validators cannot process any transaction that requires randomness. On the other hand, when running wDKG before the epoch change, validators can generate randomness and process transactions immediately after entering the new epoch. 
- Why use non-interactive wDKG? The new validators of the next epoch may be offline until the new epoch starts. This means an interactive wDKG, which requires the new validators to interact with the current validators, is not feasible. In contrast, the non-interactive wDKG does not require the new validators to be online, and can use the blockchain as the broadcast channel among old and new validators when epoch change.

Here is the description of the non-interactive wDKG designed and implemented for Aptos Roll.

- When the current epoch reaches predefined limit (2 hours) or a governance proposal requires a reconfiguration, each validator will act as a *dealer*: it first computes the weight distribution (`validator_weights`) and the weight threshold (`reconstruct_threshold_in_weights`) of the next-epoch validators via rounding, and then computes a *weighted publicly-verifiable secret sharing (wPVSS) transcript* based on the weight distribution and threshold, and sends it to all validators via a reliable multicast.
- Once each validator receives a secure-subset of *valid transcripts* from 66% of the stake, it aggregates the valid transcript into a single final *aggregated transcript* and passes it on to consensus. More specifically, the validator will propose the aggregated wPVSS transcript via a `ValidatorTransaction`. Upon committing on the first `ValidatorTransaction` that contains a valid aggregated wPVSS transcript, validators finish the wDKG and start the new epoch.
- Finally, when the new epoch begins and the new validators are online, they can decrypt their shares of the secret from the aggregated transcript. At this point, the new validators will be ready to produce randomness for every block by evaluating a wVRF in a secret-shared manner, as we explain in a later section.

#### Reconfiguration

The reconfiguration is a procedure to

- Increment the on-chain epoch counter.
- Compute the set of validators in the new epoch and their stake distribution, namely `ValidatorSet`.
- Emit a `NewEpochEvent` to validators to restart some components and enter the new epoch.

The current reconfiguration does the above steps in one [Move function call](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/reconfiguration.move#L96). 

For on-chain randomness, validators need to perform wDKG before the new epoch starts. The computation of wDKG takes `ValidatorSet` of the next-epoch as input, so that wDKG can setup shared secret among next-epoch validators according to their stake distribution and is secure under the proof-of-stake assumption. Therefore, the new reconfiguration procedure for the on-chain randomness, referred to as ***async reconfiguration***, is a phase that includes the following steps. 

- To start the async reconfiguration, validators will compute the `ValidatorSet` of the next epoch, and start the wDKG.
- Validators run wDKG as described in the above section.
- When wDKG finishes, the validators finish async reconfiguration by performing the tasks of epoch change and entering the new epoch.

One implication of the design is that, any request to change `ValidatorSet` during the wDKG phase will be rejected. 

There is one more complication about the async reconfiguration due to *on-chain configuration* changes. On-chain configurations are special on-chain resources that control system behaviors (e.g., `ValidatorSet`, `ConsensusConfig`, `Features`). Some of them, when being updated by a governance proposal transaction, require an instant reconfiguration in the same transaction, to guarantee consistency among validators even under crashes. However, async reconfiguration cannot be instant (requires wDKG to finish), and the on-chain configuration changes cannot be applied before the new epoch starts. The solution is, the validators buffer the on-chain configuration changes when the transaction is executed, run the wDKG, and then apply the buffered on-chain configuration changes when the new epoch starts. All on-chain configuration changes during the wDKG will be buffered and applied together when new epoch starts. 

#### Randomness Generation Using wVRFs

In each epoch, the validators will collaborate to produce randomness for *every* block finalized by consensus, by evaluating a *weighted verifiable random function (wVRF)* using shared secret established by wDKG. 

- When a block achieves consensus finalization, each validator will use its decrypted share from the aggregated wPVSS transcript to produce its wVRF *share* for that block by evaluating *wVRF* on block-specific unbiasable message (e.g., epoch number and round number), and reliably-multicast this share.
- Upon collecting wVRF shares that exceeds the reconstruction weight threshold (`reconstruct_threshold_in_weights` by the stake rounding algorithm), every validator will combine them and derive the same final wVRF *evaluation*, which essentically equals to the shared secret evaluated on the block-specific unbiasable message. Lastly, validators attach wVRF *evaluation* as the block seed and send the block for execution.

Importantly, only 50% or more of the stake can compute the wVRF evaluation, which ensures **unpredictability**. Furthermore, the uniqueness property of wVRFs together with the secrecy of the wPVSS scheme ensure **unbiasability** against adversaries controlling less than 50% of the stake.

## Reference Implementation

 > This is an optional yet highly encouraged section where you may include an example of what you are seeking in this proposal. This can be in the form of code, diagrams, or even plain text. Ideally, we have a link to a living repository of code exemplifying the standard, or, for simpler cases, inline code.

...

## Testing (Optional)

 > - What is the testing plan? (other than load testing, all tests should be part of the implementation details and won’t need to be called out)
 > - When can we expect the results?
 > - What are the test results and are they what we expected? If not, explain the gap.

Unit tests, smoke tests, forge tests.

Deploy a test network called `randomnet`.

Further stress testing in `previewnet`.

## Risks and Drawbacks

 > - Express here the potential negative ramifications of taking on this proposal. What are the hazards?
 > - Any backwards compatibility issues we should be aware of?
 > - If there are issues, how can we mitigate or resolve them?

**Validators fail to finish DKG due to bugs.** To mitigate against this, validators will eventually reconfigure and change epoch after a large timeout. The new epoch will not have randomness available. This timeout-based mitigation will be disabled in future releases, since such bugs should never occur.

**Validators fail to generate randomness**. If this happens, the blockchain will halt as every block needs randomness to proceed for execution.

**MEV attacks**. There is a risk of validators colluding together and leak the VRF secret key once the stake ratio of the malicious validators exceeds the secrecy threshold (50%). If this happens, the colluding validators can predict all randomness in the current epoch.

**Performance impact.** The randomness generation phase adds latency overhead in the block commit latency. More details can be found in our [paper](https://eprint.iacr.org/2024/198).

**Rounding errors.** With increasing the number of validators and/or sufficiently-bad stake distributions, the number of total secret shares outputted by our rounding algorithm can become larger and larger, degrading the performance of the wDKG phase and the per-block wVRF aggregation. **TODO:** address

**Reconfiguration impact**. The wDKG phase will affect the reconfiguration process. For periodic reconfigurations that happen every two hours, the validators need to finish the wDKG before the next epoch starts. For governance proposals that require reconfiguration, the changes by the governance proposal will be buffered on-chain, and applied after the wDKG finishes and validators enter the new epoch. During the wDKG period, changes to the `ValidatorSet` will be rejected. Any future new on-chain configs need to follow the same procedure.

**Shared secret being a group element has limited applications**. One limitation of the current implementation is that the shared secret of the wDKG is a group element instead of the field element. This means the shared secret cannot be used for other applications such as threshold decryption.

## Future Potential

 > Think through the evolution of this proposal well into the future. How do you see this playing out? What would this proposal result in one year? In five years?

...

## Timeline

### Suggested implementation timeline

 > Describe how long you expect the implementation effort to take, perhaps splitting it up into stages or milestones.

...

### Suggested developer platform support timeline

 > Describe the plan to have SDK, API, CLI, Indexer support for this feature is applicable. 

...

### Suggested deployment timeline

 > Indicate a future release version as a *rough* estimate for when the community should expect to see this deployed on our three networks (e.g., release 1.7).
 > You are responsible for updating this AIP with a better estimate, if any, after the AIP passes the gatekeeper’s design review.
 > 
 > - On devnet?
 > - On testnet?
 > - On mainnet?

Targeting release v1.11

## Security Considerations

 > - Does this result in a change of security assumptions or our threat model?
 > - Any potential scams? What are the mitigation strategies?
 > - Any security implications/considerations?
 > - Any security design docs or auditing materials that can be shared?

Test-and-abort attach preventions.

Under-gasing attack preventions.

**TODO:** anything else

## Open Questions (Optional)

 > Q&A here, some of them can have answers some of those questions can be things we have not figured out but we should

## References

[^aip-41]: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md