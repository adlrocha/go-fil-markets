**[!!] This will be removed soon. Keeping it here for completion for now.**

# ~~Continous ZKCP~~ !! This is essentially zkcp-per-chunk
This is fundamentally wrong. Hash-lock transactions are always done in Filecoin through a payment
channel, so this is essentially the ZKCP-per-chunk.
~~Is the traditional ZKCP proposal for chunks leveraging payment channels and partial payments.
There is a voucher exchange with intermediate payments for each encryped chunk.
The provider needs to release the key to redeem the vouchers and settle
the payment channel. The client can't escape with the data sent (it is
encrypted).~~

~~**Implementation requirements:**~~

* Building the right proofs for the data to send them through data channel.
    - Proof for the whole data.
* Hash-lock transactions in Filecoin
* Upgradable payment channels to include verification through hash-lock.

**Cons**
This schemes opens the door to a lot of useless work if one of the parties aborts.
It can be secure but wasteful.
I don't think it offers any improvement to the traditional ZKCP. This merged
with the 1-bit ZKCP could make sense.~~



### Client
* To settle the payment channel, the provider needs to send k for redemption. 
Vouchers are hash-locked.

```mermaid
stateDiagram-v2
[*] --> DealStatusNew: ClientRetrieve
DealStatusNew --> DealStatusWaitAcceptance
DealStatusWaitAcceptance --> DealStatusAccepted
DealStatusAccepted --> DealStatusPaymentChannelCreating
DealStatusPaymentChannelCreating --> DealStatusChannelAllocatingLane
DealStatusChannelAllocatingLane --> DealStatusWaitingFirstBlock
DealStatusWaitingFirstBlock --> DealStatusOngoing

note right of DealStatusOngoing
Deal warmup finished.
Here is were the actual retrieval begins
end note

note right of DealStatusWaitingFirstBlock
The first block includes the proof to be verified
Proof(c, y, k) where c = Enc_k(data), y = sha(k) 
the stake is performed once the proof can be
verified.
end note


DealStatusOngoing --> DealStatusFundsNeeded
DealStatusFundsNeeded --> DealStatusSendFunds
DealStatusSendFunds --> DealStatusCheckFunds
DealStatusCheckFunds --> DealStatusOngoing
DealStatusOngoing --> DealStatusBlockComplete

note right of DealStatusSendFunds
Send voucher with partial payment
end note



DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusVerifyProofs
DealStatusVerifyProofs --> DealStatusUpdatePaymentCh
DealStatusUpdatePaymentCh --> DealStatusWaitForOnchainTx
DealStatusWaitForOnchainTx --> DealStatusCompleted
DealStatusCompleted --> [*]

note right of DealStatusVerifyProofs
Verifies Proof^-1(c, k, y) where
c = Enc_k(data), y = sha(k). If
proof correct update payment channel to make
vouchers redeemable.
end note

note right of DealStatusUpdatePaymentCh
Update the payment channel so it can only be
settled with the pre-image of k. Without this
update it can never be settled.
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
DealStatusGenerateProof --> DealStatusOngoing

DealStatusFundsNeeded --> DealStatusOngoing
DealStatusOngoing --> DealStatusFundsNeeded
note left of DealStatusFundsNeeded
Stays in this loop
until all encrypted blocks received
end note

DealStatusOngoing --> DealStatusBlockComplete
DealStatusBlockComplete --> DealStatusWaitChUpgrade
DealStatusWaitChUpgrade --> DealStatusCompleted
DealStatusCompleted --> [*]

note left of DealStatusGenerateProof
Generate Proof(c, k, y) where
c = Enc_k(data), y = sha(k). Send it
to client.
end note

note left of DealStatusCompleted
The channel settlement is performed
manually. The provider waits until the
channel is upgraded before releasing
the key. Not part of fsm.
end note
```