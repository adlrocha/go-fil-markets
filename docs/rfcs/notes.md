# Design notes
Client metering proposals:

### Baseline ZKCP (with optimizations)
ZKCP for the full data without payment channels involved.

*   Client requests retrieval and opens a pull data channel with a provider. Client stakes the payment amount on-chain to start the channel.
*   Provider generates a key `k`, uses that key to encrypt the data, computes `sha(k)` and a ZK proof that verifies that the data was indeed encrypted with that key.
*   Provider sends the proof and starts sending the data over the pullChannel to the client. 
*   Upon reception the client validates the data received against the proof, and sends a voucher to perform the hash-lock transaction that allows the provider to access the payment funds if he released the right `k`.
*   Provider reveals `k` and the payment channel can be settled.
*   Instead of sending all the encrypted data, and the proofs all at once, in an alternative implementation of the protocol this could be done for the different chunks of the file. For each chunk, the provider needs to encrypt with a different key and create a new proof. There’s probably no need for additional on-chain interactions (as long as the provider doesn’t want to redeem half-way), as the client sends hash-lock vouchers for each chunk that can afterwards be redeemed seamlessly when the payment channel is settled.

*   __Implementation requirements:__
    *   Efficient ZK-Proof for data of different sizes.
    *   ZK-Proof of encryption key (for an optimization proposal, we can probably use constant size keys). 
    *   Hash-lock transactions (these are done through SecretPreimage in payment channels)
*   __Potential optimizations:__
    *   Precomputed data encryption and proofs. Precomputed versions of hot copies with different keys. For each client, the provider chooses a pre-computed version with a different key and the ZKCP protocol is run over the encryption key and not over the data.
    *   Run fair-exchange only for n of a total of m chunks the data is composed by.
    *   Use the payment channel to run a continuous fair-exchange for the different chunks (explained above).
*   __Limitations:__
    *   The ZK-proof for the data may be inefficient to compute.
    *   There is not an easy way to do an incremental exchange without a high overhead, which means that if a party aborts the amount of useless work committed may be large.

## 1-bit ZKCP 
Instead of having to encrypt chunks with different keys to run a  ZKCP-per-chunk to enable incremental payments, the provider encrypts the data with the same `k` of n bits. A fair-exchange for 1-bit of the key (i.e. a string of the key) is run after the exchange of every n/m chunks of encrypted data.

*   The client requests the retrieval creates a new payment channel staking the amount of the payment.
*   Provider generates a key `k` and uses that key to encrypt the data.
*   After every m/n of data the provider computes `sha(1bit-k + mask?)` and a ZK proof that verifies that the data was indeed encrypted with a key to which that bit belongs, and that the m/n chunks belong to that data.
*   Upon reception the client verifies the proof, and sends the right hash-lock voucher for the preimage of the sha. To access the payment the provider needs to release the bit of the key and redeem the voucher in the state-channel settlement.
*   This is repeated for the different sections of the key. If either the client or the provider aborts early the former can still brute force the key while the latter can redeem part of the payments. The price should be incremental to ensure that the penalty of aborting early is fair.

*   __Implementation requirements:__
    *   A Zk-Proof that shows that a bit string belongs to the key used to encrypt the data sent, and that the chunk sent belongs to that data. 
    *   Hash-lock transactions (these are done through SecretPreimage in payment channels)
*   __Limitations and open questions:__
    *   It is easy for any of the parties to delay or even pause the protocol at any time, complicating timely deliveries of data.
    *   Is it possible to design such a ZK-Proof?
    *   How can we design a fair penalty through the pricing function of the chunks?

## Optimistic ZKCP
*   On a payment channel:
    *   Agree on the query - both parties sign
    *   In case the query is simply retrieving a CID, then privacy can be maintained even for a dispute, by revealing only the hash of the query to the chain
    *   Agree on the prices
*   Buyer deposits enough coins for query execution + bandwidth + overhead
*   Seller executes query
    *   If seller aborts at this point, query times out and coins return to buyer
*   Seller sends result, in encrypted chunks
    *   If seller aborts at this point, buyer goes to on-chain dispute resolution
*   Buyer acknowledges receipt of encrypted chunks & pays per-chunk for bandwidth
    *   If buyer aborts at this point, seller is out of luck - reputation risk
    *   Possibly, buyer could also report the speed (timestamping the ACKs)
    *   If buyer starts reporting slower than it actually is, the seller will throttle to match
*   Seller reveals decryption key
    *   If seller aborts, buyer can go on-chain and claim a slash against the seller (and seller can reveal decryption key, with a proof, to “appeal” the slash)
*   Buyer verifies query satisfaction
*   Buyer acknowledges receipt of decryption key
*   If buyer aborts, seller goes to on-chain and makes a proof

*   __Implementation requirements:__
    *   Efficient ZK-Proof for data of different sizes
    *   Hash-lock transactions (these are done through SecretPreimage in payment channels)
    *   Dispute resolution Judge Actor. This could be implemented as a retrieval actor.
*   Limitations:

__Basic components to implement for all of the protocols:__
*   ZK-Proof for large amounts of data or chunks of different size.
*   ZK-Proof for a constant size string (key)
*   Hash-lock verification in payment channels.
*   Retrieval actor for disputes in optimistic-ZKCP.

__Next steps:__
*   Design proofs and implement a basic ZKCP protocol to understand performance and model simulations and formal verifications of protocols with the data collected.
*   Evaluate most promising from all the schemes and start implementing them.
*   ...