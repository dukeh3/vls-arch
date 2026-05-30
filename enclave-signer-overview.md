# Lightning Signer in a Secure Enclave

## What This Is

A Lightning Network signing service that runs inside a hardware-protected
vault. The signing keys -- the secrets that authorize moving Bitcoin --
never exist outside this vault. Not even the server operator can extract
them.

## The Problem

When you run a Lightning node, the software that holds your keys typically
runs on the same machine as everything else. If that machine is
compromised, an attacker gets your keys and can steal your funds.

The standard defense is a "remote signer" -- move the keys to a separate
machine that only signs what it's asked to sign, and validates every
request against a set of rules. But what if someone compromises that
separate machine too?

## The Solution

We run the signer inside an AMD SEV-SNP secure enclave. This is a special
mode of modern AMD server processors where the CPU encrypts all of the
program's memory with a key that only the hardware knows. The operating
system, the hypervisor, even someone with physical access to the server
cannot read the signer's memory.

## How the Signer Gets Its Keys

The signer does not come pre-loaded with secrets. Instead, a separate
component called the **initiator** provisions the signer after verifying
that it is running inside a genuine enclave.

The process works like this:

1. The enclave boots with only one piece of pre-installed information: the
   initiator's public identity.

2. The signer generates a temporary key and asks the AMD processor to
   certify it. The processor produces an **attestation report** -- a
   cryptographic statement, signed by the chip itself, that says: "this
   specific program is running inside an encrypted VM on this specific AMD
   processor, and this temporary key was generated inside it."

3. The signer sends this attestation to the initiator over a message relay.

4. The initiator checks the attestation. It verifies the processor's
   signature chain all the way back to AMD's root certificates. It
   confirms the enclave has debugging disabled. It checks that the
   temporary key matches what the processor certified.

5. Only after all checks pass does the initiator send the signer its
   secrets: the master seed (from which all Lightning channel keys are
   derived) and the identities needed to communicate with the rest of the
   system.

6. The signer receives these secrets, initializes, and begins signing
   Lightning transactions.

The master seed only ever travels encrypted to a key that the AMD
processor has certified is inside the enclave. Once inside, it never
leaves.

## How It Communicates

The signer has no open ports. It makes a single outbound connection to a
Nostr relay -- a simple message broker. The Lightning node's proxy sends
signing requests to the relay, the signer picks them up, validates them,
signs if appropriate, and posts the response back. All messages are
encrypted between the two parties.

This means the signer can run anywhere with network access to the relay.
It doesn't need to be on the same network as the Lightning node. It
doesn't even need a public IP address.

## The Components

Bob operates a Lightning node with a remote signer in an enclave.
Alice is a regular Lightning user who wants to pay Bob.

```
Bob's Infrastructure
--------------------

Initiator (trusted setup component)
  Holds the master seed
  Verifies enclave attestation
  Provisions the signer with secrets
  Runs once at startup, then watches for reboots

Signer (inside AMD SEV-SNP encrypted VM)
  Receives seed from initiator after attestation
  Validates and signs Lightning transactions
  Enforces spending policy rules

Lightning node (lnrod + proxy)
  Manages channels and routes payments
  Sends signing requests to signer via relay

Nostr relay (message broker)
  Passes encrypted messages between all components
  Stateless -- just forwards messages

bitcoind
  Provides access to the Bitcoin blockchain


Alice's Side
------------

ldk-controller
  Opens channels to Bob's Lightning node
  Sends and receives payments
  Does not know or care about Bob's signer setup
```

## What Alice Sees

Alice sees a normal Lightning node. She opens a channel, sends a payment,
receives a payment, and closes the channel. She has no idea that every
signing operation on Bob's side went through a hardware-encrypted vault
via an encrypted relay. From her perspective, Bob is just another node on
the network.

## What the Test Proves

The end-to-end test runs a complete Lightning channel lifecycle through
the enclave signer:

1. The initiator verifies the enclave and provisions the signer
2. Open a payment channel (2,000,000 sat)
3. Send a payment from Alice to Bob
4. Send a payment from Bob to Alice
5. Close the channel cooperatively
6. Verify final balances are correct

Every signing operation in this flow -- funding transaction, commitment
updates, HTLC handling, closing transaction -- goes through the signer
inside the secure enclave, provisioned by the initiator, communicating
via the Nostr relay.

## Why This Matters

Traditional Lightning node security relies on the operator securing their
server. This setup removes the operator from the trust model entirely:

- The **hardware** guarantees memory isolation (AMD SEV-SNP)
- The **attestation** proves the signer is running in real hardware
- The **initiator** only provisions verified enclaves
- The **signer** enforces policy rules on every transaction
- The **relay** means no network ports need to be opened

Even if an attacker gains root access to the host server, they cannot
read the signer's memory, extract the keys, or forge an attestation.
