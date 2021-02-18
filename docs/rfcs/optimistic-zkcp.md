# State machine designs

## Optimistic ZKCP
It can be seen like an "optimistic" zkcp-per-chunk with a judge to solve disputes. 
There is no need to pause the exchange between chunk and voucher exchanges.
* A payment channel to cover the retrieval costs is set up client and provider.
* The client encrypts the chunks of data with a different key, and generates a proof
of integrity and a proof of computation for each chunk.
* In exchange for each chunk, the client sends a hash-locked voucher. The provider then reveals the key for that chunk.There is no need for intermediate redemtpion of vouchers
or any on-chain messages.
* When the exchange finishes, the provider can redeeem the vouchers on-chain to access the payment.
* There is a way to recover funds in a fair way if there is a misbehavior by any of the parties
    * If provider sends wrong key or wrong proof, the client can go to the judge to make a dispute and request a slash (and recover the funds).
    * The only weak point is if the client sends the wrong voucher and runs with the data, there is no easy way to fix this except by using some kind of reputations systems.

**Implementation requirements:**
* Building the right proofs for the data to send them through data channel.
    - Proof for the whole data or data chunks.
* Retrieval actor to manage optimistic ZKCP stages

**Pros**
* Partial payments are possible, something that was hard to achieve in ZKCP without releasing
the key. Requires the generation of more proofs.

**Improvement proposals**
* Instead of having to set up a payment channel with every miner there could be a per-wallet
general-purpose deposit for fast retrievals (we would need to evaluate the risks of double-spending).



### Client

```mermaid
stateDiagram-v2
[*] --> DealStatusNew: ClientRetrieve
DealStatusNew --> DealStatusWaitAcceptance
DealStatusWaitAcceptance --> DealStatusAccepted
DealStatusAccepted --> DealStatusPaymentChannelCreating
DealStatusPaymentChannelCreating --> DealStatusChannelAllocatingLane
DealStatusChannelAllocatingLane --> DealStatusOngoing

note right of DealStatusOngoing
Deal warmup finished.
Here is were the actual retrieval begins
end note

DealStatusOngoing --> DealStatusWaitingBlock
DealStatusWaitingBlock --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusSendFunds
DealStatusSendFunds --> DealStatusCheckFunds
DealStatusCheckFunds --> DealStatusStartDispute: wrongKey
DealStatusStartDispute --> DealStatusSendProofs
DealStatusSendProofs --> DealStatusCompleted
DealStatusCheckFunds --> DealStatusOngoing

note right of DealStatusVerifyProofs
Verifies Proof^-1(c, k, y) where
c = Enc_k(data), y = sha(k)
end note
DealStatusOngoing --> DealStatusBlockComplete: FinalChunk?
DealStatusBlockComplete --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusSendFunds
DealStatusSendFunds --> DealStatusCheckFunds
DealStatusCheckFunds --> DealStatusCompleted: AllChunksRcvd?
DealStatusCompleted --> [*]

note right of DealStatusSendFunds
Send hash-locked voucher for provider.
Only redeemable if the key is revealed.
end note
```

### Provider
```mermaid
stateDiagram-v2
[*] --> DealStatusNew: RetrievalOrder
DealStatusNew --> DealStatusUnsealing
DealStatusUnsealing --> DealStatusUnsealed
DealStatusUnsealed --> DealStatusOngoing
DealStatusOngoing --> DealStatusEncrypt
DealStatusEncrypt --> DealStatusGenerateProof
DealStatusGenerateProof --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusRevealKey
DealStatusRevealKey --> DealStatusOngoing
note left of DealStatusFundsNeeded
Waits for client to send voucher
with the payment. 
end note
note left of DealStatusGenerateProof
Generate Proof(c, k, y) where
c = Enc_k(data), y = sha(k)
end note
DealStatusRevealKey --> DealStatusAbort: wrongVoucher!
DealStatusAbort --> DealStatusCompleted
DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusEncrypt
DealStatusGenerateProof --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusRevealKey
DealStatusRevealKey --> DealStatusCompleted
DealStatusCompleted --> [*]

note left of DealStatusAbort
If at one point the provider
sees a forged voucher it aborts the exchange.
end note
note left of DealStatusCompleted
The channel settlement is performed
manually. Not part of fsm.
end note
```
