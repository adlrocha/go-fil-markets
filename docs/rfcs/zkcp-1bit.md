# State machine designs

## ZKCP-1bit
Instead of having to use a different key for each chunk like in
a traditional chunk-based ZKCP, all the data is encrypted with the same `k` of
`n bits`. There is a payment and a fair-exchange for 1 bit of the key every `m/n`.
It is a mix between traditional ZKCP-per-chunk and continous-ZKCP.

There is no need for a payment channel.

**Implementation requirements:**
* Building the right proofs for the data to send them through data channel.
    - Proof for the whole data.
    - Proof that the bit for the data belongs to the key
* Hash-lock transactions in Filecoin

**Pros**
Partial payments are possible, something that was hard to achieve in ZKCP without releasing
the key. Requires the generation of more proofs.



### Client
```mermaid
stateDiagram-v2
[*] --> DealStatusNew: ClientRetrieve
DealStatusNew --> DealStatusWaitAcceptance
DealStatusWaitAcceptance --> DealStatusAccepted
DealStatusAccepted --> DealStatusOpenDataChannel
DealStatusOpenDataChannel --> DealStatusOngoing

DealStatusOngoing --> DealStatus1BitExchange
DealStatus1BitExchange --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusStakePayment
DealStatusStakePayment --> DealStatusWaitForOnchainTx
DealStatusWaitForOnchainTx --> DealStatusOngoing

note right of DealStatusVerifyProofs
Verifies that the bit being exchanged was used
in the key that encrypts the data
Proof(c, y, k) where c = Enc_k(data), y = sha(k) 
end note

DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStateFinalBitExchange
DealStateFinalBitExchange --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusStakePayment
DealStatusStakePayment --> DealStatusWaitForFinalOnchainTx
DealStatusWaitForFinalOnchainTx --> DealStatusCompleted
DealStatusCompleted --> [*]

```

### Provider
```mermaid
stateDiagram-v2
[*] --> DealStatusNew: RetrievalOrder
DealStatusNew --> DealStatusUnsealing
DealStatusUnsealing --> DealStatusUnsealed
DealStatusUnsealed --> DealStatusOngoing
DealStatusOngoing --> DealStatusEncrypt
DealStatusEncrypt --> DealStatusOngoing

DealStatusOngoing --> DealStatus1BitProof
DealStatus1BitProof --> DealStatusGenerateProof
DealStatusGenerateProof --> DealStatusWaitStake
DealStatusWaitStake --> DealStatusOngoing


DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStateFinalBitExchange
DealStatusFinalBitExchange --> DealStatusGenerateProof
DealStatusGenerateProof --> DealStatusWaitStake
DealStatusWaitStake --> DealStatusCompleted
DealStatusCompleted --> [*]

note left of DealStatusGenerateProof
Generate Proof(c, k, y) where
c = Enc_k(data), y = sha(k). Send it
to client.
end note

```