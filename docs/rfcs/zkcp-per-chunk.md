# ZKCP-per-chunk
This assumes a ZKCP protocol with a proof-of-retrievability per chunk.
Is the traditional ZKCP proposal but running a fair-exchange for
each chunk instead of doing it for the whole file.
* This would allow the use of fixed-size chunks.

**Implementation requirements:**
* Building the right proofs for the data to send them through data channel.
    - Proof of chunks of data (potentially different sizes, or pre-agreed sizes).
* Hash-lock transactions in Filecoin


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
DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusEncrypt
DealStatusGenerateProof --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusRevealKey
DealStatusRevealKey --> DealStatusCompleted
DealStatusCompleted --> [*]

note left of DealStatusCompleted
The channel settlement is performed
manually. Not part of fsm.
end note
```

