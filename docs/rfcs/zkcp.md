# Baseline ZKCP
ZKCP for the full data. The payment channel is required to
perform the hash-lock transaction in Filecoin.
* Client creates payment channel with enough funds for the payment.
* Opens pull-data channel with the provider storing the content.
* The provider sends the data and the proofs.
* Client sends a hash-locked voucher with the funds for the payment.
* To redeem the voucher the provider needs to expose the key.

**Implementation requirements:**
* Building the right proofs for the data to send them through data channel.
    - Proof of the whole data (or chunks of data).
    - Proof of an encryption key.
* Hash-lock transactions in Filecoin

**Potential Optimizations:**
- Single proof-of-retreivability for the whole data (all data in a single chunk).
- Proofs for some of the chunks.
- Pre-computed encryptions and proofs.
    - There are pre-computed encryptions for the data with different keys.
    - The fair exchange is performed for the key that encrypted the encryption key and not the encryption key itself.
- Use payment channel to perform incremental payments which are settled by a fair exchange.

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

DealStatusOngoing --> DealStatusWaitingData
DealStatusWaitingData --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusSendFunds
DealStatusSendFunds --> DealStatusCheckFunds
DealStatusCheckFunds --> DealStatusCompleted

note right of DealStatusSendFunds
Send hash-locked voucher for provider.
Only redeemable if the key is revealed.
end note

note right of DealStatusVerifyProofs
Verifies Proof^-1(c, k, y) where
c = Enc_k(data), y = sha(k)
end note

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
DealStatusEncrypt --> DealStatusGenerateProof
DealStatusGenerateProof --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusRevealKey
DealStatusRevealKey --> DealStatusCompleted
note left of DealStatusFundsNeeded
Waits for client to send voucher
with the payment. 
end note
note left of DealStatusGenerateProof
Generate Proof(c, k, y) where
c = Enc_k(data), y = sha(k)
end note

note left of DealStatusRevealKey
Reveals key. Either redeeming the
voucher or directly sending it to
the client.
end note
DealStatusCompleted --> [*]

note left of DealStatusCompleted
The channel settlement is performed
manually. Not part of fsm.
end note
```
