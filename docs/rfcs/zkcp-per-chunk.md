# State machine designs

## ZKCP-per-chunk
This assumes a ZKCP protocol with a proof-of-retrievability per chunk.
Is the traditional ZKCP proposal for chunks without the use of payment channels.

**Implementation requirements:**
* Building the right proofs for the data to send them through data channel.
    - Proof of chunks of data (potentially different sizes, or pre-agreed sizes).
* Hash-lock transactions in Filecoin


### Client
* To unlock the deposit, the provider sends `k` in the transaction
to validate the pre-image, and the client can get knowledge of the key
used to encrypt the data.

```mermaid
stateDiagram-v2
[*] --> DealStatusNew: ClientRetrieve
DealStatusNew --> DealStatusWaitAcceptance
DealStatusWaitAcceptance --> DealStatusAccepted
DealStatusAccepted --> DealStatusOpenDataChannel
DealStatusOpenDataChannel --> DealStatusOngoing

note right of DealStatusOngoing
Deal warmup finished.
Here is were the actual retrieval begins
end note

DealStatusOngoing --> DealStatusWaitingBlock
DealStatusWaitingBlock --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusStakePayment
DealStatusStakePayment --> DealStatusWaitForOnchainTx
DealStatusWaitForOnchainTx --> DealStatusOngoing

note right of DealStatusVerifyProofs
Verifies Proof^-1(c, k, y) where
c = Enc_k(data), y = sha(k)
end note
DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusFinalStakePayment
DealStatusFinalStakePayment --> DealStatusWaitForFinalOnchainTx
DealStatusWaitForFinalOnchainTx --> DealStatusCompleted
DealStatusCompleted --> [*]

note right of DealStatusStakePayment
Locks funds on-chain that can only
be unlocked commiting the pre-image
of sha(k) signed by client
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
DealStatusGenerateProof --> DealStatusWaitStake
DealStatusWaitStake --> DealStatusOngoing
note left of DealStatusWaitStake
Waits for client to stake funds on-chain
for the right pre-image before going on.
end note
note left of DealStatusGenerateProof
Generate Proof(c, k, y) where
c = Enc_k(data), y = sha(k)
end note
DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusEncrypt
DealStatusEncrypt --> DealStatusGenerateProof
DealStatusGenerateProof --> DealStatusWaitFinalStake
DealStatusWaitFinalStake --> DealStatusCompleted
DealStatusCompleted --> [*]

note left of DealStatusCompleted
The channel settlement is performed
manually. Not part of fsm.
end note
```

