# Lightspeed micropayments for Lightning Network

Dr. Maxim Orlovsky, Pandora Core AG

28 March 2020

## Abstract

This work proposes concept of fast and reliable micropayments protocol
("Lightspeed micropayments for Lightning Network") which allows millions of
transactions per channel without any additional network traffic and any
per-transaction signature generation. This requires generalization of the
Lightning network (such that parties may be able to add additional outputs to
the commitment transactions) and RGB.


## 1. Background

### 1.1. Setup

One of the main design goals for the Lightning Network was an idea of
micropayments: fast repeating payments for small amounts (sometimes <1 satoshi)
between two parties.

However, the current Lightning Network design prevents effective micropayments
due to the number of factors, which we will examine on a model setup where
Party 1 (Client) has to pay to Party 2 (Server) for each API call.

The setup of the configuration will require two Lightning Nodes; we will examine
the simples case when there exists a direct channel between them:

```
+------------+                                +------------+
| Client app | <----------------------------> | Server app |
+------------+                                +------------+
     |                                                 |
+-------------+                              +-------------+
| Client's LN | <--------------------------> | Server's LN |  
+-------------+                              +-------------+
```
Fig. 1. Setup of micropayments on arbitrary server API calls.


Possible payment workflow is the following:
* Server LN generates the invoice,
* passes it to server app,
* which sends it to the client app,
* client app instantiates payment from Client's LN node
* Client LN performs payment via Lightning channel, which requires at least
  three network requests: `update_add_htlc`, `commitment_signed` and
  `revoke_and_ack`
* After fulfilling the payment client app calls API call on Server and waits
  till
* the server app polls Server's LN for the payment fulfillment and notifies
  client that it is ready to provide it with the data.

### 1.2. Problem

So, the procedure takes at least:
* generation of multiple signatures: `2 * (pending_htlc_count * 2 + 2 + 1)` -
  two for the new HTLC, one for each of the existing HTLCs and one for the
  commitment transaction; on each side of the channel — i.e. at least
  **6 signatures**;
* **4 inter-process communications** (2 on each side) between application and
  Lightning Node;
* **3 network requests/replies** between Client and Server Lightning Nodes and
* **2 network requests** between Client and Server apps, including the actual
  API call.

As a result, each API call gets a significant delay required to perform all
network and interprocess communications and signatures. In fact, this renders
current Lightning Network implementation practically unusable for high-frequency
micropayments: they will require at least an order of magnitude more time to
perform than an original API call without an attached payment.

### 1.3. Issues that need to be fixed

1. Reduce number of signatures (decrease CPU load on both sides)
2. Reduce number of network communications (increase speed and reduce
   probability of protocol failure)
3. Decouple payment from API request, i.e. by providing some information
   in the API call that may prove/ensure Server app about successful payment
   w/o querying Lightning Node for the confirmation.


## 2. Initial solution with Generalized Lightning Network protocol

Generalized Lightning Network protocol (gLNP) is a
[concept](generalized_lightning.md) for establishing and managing arbitrary
channels (bi-directional, multiparty) with arbitrary extensions to transaction
structure that are negotiated during the channel setup.

### 2.1. Solution specification

#### 2.1.1. Channel setup

With gLNP we may extend commitment transaction by adding a new "funding" output
to which the Client will allocate some large multiple of per-API call payments
aside of the main channel funds. This allocation (**Lightspeed funding**, or
**LSF**) will be  controlled by script in the same manned as the initial channel
funding transaction.

Additionally to LSF, parties agree and sign another transaction spending LSF
output, named **micropayment transaction**. This transaction contains multiple
equal outputs, one output per payment for a single API call. Each of these
outputs are controlled by a script locked to either Client-generated hash
pre-image or Client's time-locked private key. It is important to note, that
hash-spending script branch does not require signature by a private key and
require only hash preimage, at this stage known only to the Client; at the same
time each of the outputs MUST have different pre-image.

```
Funding output:          Commitment tx:
+-----------------+      +--------------+
| 2-of-2 multisig | <--- | to_local     |
+-----------------+      +--------------+
                         | to_remote    |      Micropayment tx:
                         +--------------+      +--------------------------+
                         | **LSF**      | <--- | Hash1-or-pubkey_timelock |
                         +--------------+      +--------------------------+
                         | HTLC's       |      | Hash2-or-pubkey_timelock |
                         +--------------+      +--------------------------+
                                               | Hash3-or-pubkey_timelock |
                                               +--------------------------+
                                               | Hash4-or-pubkey_timelock |
                                               +--------------------------+
                                               | ...                      |
                                               +--------------------------+
```
Fig. 2. Version 1 of Lightspeed-enabled channel structure (based on pure gLNP).

#### 2.1.2. Workflow

With this setup Client app will have a list of `N` hash preimages, and Server
can be aware of the list of channel ids and corresponding hashes associated with
each client.

For each API call `i < N` Client adds information on `i`-th preimage to the call
parameters (or metadata, like a custom HTTP header). Server verifies that the
preimage corresponds to a known hash and if does provides Client with the
requested data.

This reduces the computational and network load for each API call:
* from >6 down to 0 signatures
* from 4 interprocess communications on Client and Server side down to 0
  communications
* from 3 network requests between Client and Server Lightning nodes down to 0
  network requests
* from 2 network communications between Client and Server app to a single
  API call.

I.e. the proposed scheme reduces **all** additional network and signature
generation load for Lightning micropayments, enabling them in a high-performant
manner.


### 2.2. Remaining problems

With all advantages the scheme still has some significant drawbacks:
1. In case of non-cooperative channel closing the Sever will have to publish
   micropayment transaction, which size can be enormous (tens of thousands
   outputs), and it may cost more than the money earned by the Server for
   API calls.
2. It can't efficiently work with subsatoshi payments.
3. It is limited in the number of payments in a single micropayment transaction
   to the maximum number of outputs that can fit in the block (tens of thousands).
   This may be still inefficient for high-frequency API calls, which may counts
   in millions per hour (for instance if we talk about car paying for each meter
   of the highway, or other IoT-5G use cases for micropayments).


## 3. Final solution with RGB

These problems may be solved if the payments will shift from transaction-output
satoshi-based model into a sealed state RGB token-based model.

Let's assume that the Server app owner issues some "service tokens" using RGB,
which have a fixed value (in terms of either it's services or satoshis), or
are equivalent in volume to the amount of API calls it may serve.

Client byes amount of tokens required for performing desired number of server
API calls on the market and allocates them to the Lightning channel opened with
the Server into the **LSF**  output. (NB: you may think about this as a
redeemable "pre-paid", since unlike most of "pre-paid" services in our case
unspent tokens can always be removed from the channel and sold on the market).

Now, instead of micropayment transaction spending LSF into a thousands of
outputs, in RGB-based version the micropayment transaction will have **a single
output** controlled by a script releasing funds either to Server's public key
(instantly) - or to Client's public key with some timelock. At the same time,
Client constructs a client-validated RGB data structure, that defined `N` seals
(and N can be millions or more) that assign a per-request amount of tokens to
the same single micropayment transaction output. It is important to know that
each of these outputs has some unknown secret value (stored in RGB metadata),
some entropy, that makes impossible to Server to guess the data for the
client-validated RGB proof without getting this secret from the client.

With each request Client provides Server with a single secret value, so
the Server will be able to generate corresponding RGB proof and use a part of
the tokens sealed to micropayment transaction output.


```
Funding output:          Commitment tx:
+-----------------+      +--------------+
| 2-of-2 multisig | <--- | to_local     |
+-----------------+      +--------------+
                         | to_remote    |      Micropayment tx:
                         +--------------+      +--------------------------+
                         | **LSF**      | <--- | 2-of-2+timelock multisig | <-+
                         +--------------+      +--------------------------+   |
                         | HTLC's       |                                   Single-use
                         +--------------+                                   seals
                                               Client-validated RGB data:     |
                                               +--------------------------+   |
                                               | Secret1-locked tokens    | --+
                                               +--------------------------+   |
                                               | Secret2-locked tokens    | --+
                                               +--------------------------+   |
                                               | Secret3-locked tokens    | --+
                                               +--------------------------+   |
                                               | Secret4-locked tokens    | --+
                                               +--------------------------+   |
                                               | ...                      | --+
                                               +--------------------------+
```
Fig. 3. Version 2 of Lightspeed-enabled channel structure (based on gLNP+RGB).


## 4. Security analysis and further work

The proposed final schema utilizing both gLNP and RGB-based tokens allows fast
and reliable micropayments ("Lightspeed") without counterparty risk within the
payment protocol. It still does have a counterparty risk of central token issuer
withholding full pre-paid amount, however this risk can be substantially reduced
in cases when the utility tokens used for API calls are free-tradeble on the
market and/or have a fixed value.

For non-cooperative channel closings the Server has ability to leave the Client
without any of the remaining tokens, however even in this case the Server will
not have a direct economical benefit: Client's tokens which were not already
paid to the server will be simply lost forever; at the same time Server owner
will risk it's reputation, so this kind of attack can be used for exit-scam only
and do not provide additional risk: exit-scam can be done in a more simple way
by Server seizuring all pre-payment funds and stopping from providing the
API services, which leaves us with the case of non-protocol counterparty risk
already described in the previous paragraph.

Interestingly the non-cooperative case of Client's token loss can be also
technically mitigated with more complex designs providing two symmetric outputs
for the micropayment transaction; however we believe that while it does increase
complexity of the scheme it does not in fact adds to safety due to the arguments
given above.

## License

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
