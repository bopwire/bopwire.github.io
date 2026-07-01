# Bopwire — Technical White Paper

*A peer-to-peer music network where the listeners are the infrastructure, content
is verified by cryptographic fingerprint, and every play settles on a purpose-built
blockchain. This is a technical description of how the system works. It is not financial
advice and makes no claim about the monetary value of any token; the token is
described purely as the network's internal accounting and reward unit.*

---

**Contents**

1. The problem
2. The core idea
3. Network topology (who talks to whom)
4. Anatomy of a play (how data propagates, end-to-end)
5. Content identity — fingerprinting & canonical hashes
6. The transfer layer — multi-source swarm with verified pieces
7. The discovery layer — a private DHT
8. The relay layer — NAT traversal as a paid role
9. The chain — blocks, proofs, and the play lifecycle
10. Economics — the five reward lanes, supply, and burn
11. Wallets, transfers, and non-custodial escrow
12. Security model
13. Governance & moderation
14. Decentralization posture

---

## 1. The problems

The subscription model is a poor economic model for media distribution and consumption.

When Napster, Spotify and other technolgies were created, cryptocurrency did not exist.

When you charge a subscription for an all you can eat buffet, royalties etc aren't as good
as charging $20 for a physical copy. They can't be. One million artists at a subscription fee of
less than $20 practically robs artists anyway since that subscription fee divided by those million
artists is $0.00002. So, they also HAVE to do independent artists dirty, their royalties are going to
Taylor Swift instead.

Bopwire re-arranges that stack so the **people listening also supply the
delivery network**, the **payment is settled per play on a public ledger**, and no
single company owns the pipe.

In our research of cryptocurrencies with similar supply caps, even the small cap coins can
fetch $0.10 on the market. Even with .01 cents, the platform pays as much or more as
Apple **PER STREAM SERVED**. The general idea would be for the majority of music players to use
this kind of technology, so that every play is accounted for and paid for.

In addition, independent artists today practically have to *pay* to get heard. There is a lot of
debt in the sense of new music and listeners, and there's so much of it that it has become *spam*
to the listener.

To combat this problem, Bopwire pays the *listener* for plays under 10,000. We have asked this question:
Should someone who discovers something great and plays it a lot be paid to organize the quality? We
beleive the answer is actually yes, but artists shouldn't have to pay for those listens.

---

## 2. The core idea

Three mechanisms are wired into one loop:

1. **Listeners are the CDN.** When you have a song, your device *seeds* it to
   other listeners — like BitTorrent. Bandwidth scales with the audience instead of
   with a server bill.
2. **Everyone who did work gets paid per play, on-chain.** Not just the artist —
   the **peer that uploaded the bytes** (the *seeder*) and the **relay that carried
   them** are each credited a token every time a song is genuinely listened to.
   Those that put in work to listen to new and undiscovered music also get paid.
3. **Content is identified by what it sounds like.** An audio *fingerprint*
   collapses every upload of the same song into one canonical entry, so the swarm
   pools all copies and the royalty always lands on one artist — regardless of who
   uploaded which file.

The result is a self-bootstrapping delivery network where the users *are* the
infrastructure and are compensated for it, settled on a ledger no one privately
controls.

---

## 3. Network topology (who talks to whom)

Four roles exist. A single device can play several of them at once (a phone is a
listener, a seeder, and a wallet simultaneously).

![Network topology — a full node above mini-node relays above two peer players](d-topology.svg)

**In plain terms.** Listeners (players) don't usually connect to each other
directly — they connect to **mini-nodes**, which are relays sitting on reachable
public IP addresses. The mini-node passes audio between two listeners (one seeding,
one downloading). The **full node** is the bookkeeper and rule-keeper: it knows who
holds what, runs the rules that decide a "real" play, and writes the blockchain.

**Under the hood.**
- Transport is **librats**, a custom C++ peer-to-peer library.
- Identity is unified: one key is your network identity *and* your wallet.
- Mini-nodes exist because most listeners are behind NAT and can't accept inbound
  connections. It's also a deliberate economic choice (see §8).

---

## 4. Anatomy of a play (how data propagates, end-to-end)

This is the central data-flow. Follow a single song from "tap play" to "everyone
paid."

![Anatomy of a play — a sequence diagram across listener, full node, mini-node and seeder: discover, fetch, account, settle](d-anatomy.svg)

**Step by step.**

- **① Discover.** The listener asks the full node *who has this content_hash*. The
  node answers with a list of online peers that hold it **plus the song's piece
  manifest** (the list of per-piece checksums — see §6). One round-trip; the
  manifest means the listener already knows the file's exact size and shape.
- **② Fetch.** The listener requests a *range of pieces* with `swarm.fetch`. The
  request travels to a seeder **through a mini-node** (`relay.forward`); the seeder
  streams the bytes back as **binary frames** through the same relay. The seeder
  paces itself to ≤4 Mbit/s per stream. The listener verifies each 256 KB piece
  against the manifest's SHA-256 as it arrives; a corrupt or lying piece fails its
  hash and is refetched from a different seeder.
- **③ Account.** Independently, the player opens a **session** with the full node,
  sends a position heartbeat every 5 seconds while actually playing, and on finish
  reports `session.complete` — including the **addresses of the seeder and relay**
  that served this play.
- **④ Settle.** The node checks the play is genuine (§9), signs a *PlayProof*, and
  mints the five reward lanes into the next block. Balances update.

**Why two parallel tracks?** The **audio** path (①②) and the **accounting** path
(③④) are deliberately decoupled. Audio flows peer-to-peer through relays; payment
flows through the full node's session pipeline. They meet only at the end, when the
session reports who served the bytes.

---

## 5. Content identity — fingerprinting & canonical hashes

**In plain terms.** Two people can upload "the same song" as different files
(different bitrate, different encoder, a re-rip). Bopwire recognizes them as the
*same song* by listening to them, not by comparing file bytes. That way the swarm
pools every copy together and the artist earns no matter which copy played.

**Under the hood.**
- Every song carries two hashes: a **`content_hash`** (SHA-256 of the exact audio
  bytes — addresses *this file*) and a **`fingerprint_hash`** (SHA-256 of a
  compressed **Chromaprint** acoustic fingerprint — addresses *this recording*).
- On ingest the player computes both and submits via `fingerprint.submit`. The full
  node matches against the chain three ways: exact `fingerprint_hash`, exact
  `content_hash`, then a **fuzzy Chromaprint similarity** search (bucketed index,
  similarity ≥ threshold) so different encodings of one recording **canonicalize**
  to a single on-chain song entry.
- The canonical entry lives in a block's `SongSection` (artist address, royalty
  splits, metadata, duration). Variants register against it instead of duplicating.

---

## 6. The transfer layer — multi-source swarm with verified pieces

This is "Swarm Transfer v2," the layer that actually moves the audio.

**In plain terms.** A song is cut into fixed 256 KB pieces. A download can pull
different pieces from *several seeders at once*, so your speed is the sum of their
upload speeds, not any one peer's. Every piece arrives with a checksum the uploader
*can't fake*, so pulling from strangers is safe — a bad piece is detected instantly
and refetched elsewhere.

![Multi-source swarm — three seeders each serving a piece range through a relay to one downloader, every piece checked against the manifest](d-swarm.svg)

**Under the hood.**
- **Pieces:** 256 KB. **Ranges:** a `swarm.fetch{piece_start,count}` pulls up to 16
  pieces (4 MB) per request, so one round-trip amortizes over many pieces.
- **Manifest:** generated at ingest — `sha256` of each piece — and served by the
  full node alongside the peer list. Per-piece verification is what makes
  *untrusted* multi-source safe: a bad piece fails its own hash (you know *which*
  one) instead of only discovering corruption after assembling the whole file.
  The whole-file `content_hash` is the final backstop.
- **Wire format:** binary, not base64 — pieces stream as `F`-frames
  `[stream_id | seq | eof | payload]` through the relay's binary channel (~33%
  less overhead than a JSON/base64 reply).
- **Flow control:** each seeder paces its own upload to a **4 Mbit/s** per-stream
  cap; a downloader opens **one flow per seeder** (a process-global per-seeder
  window) and aggregates across many — so total speed scales with swarm size while
  no single seeder is overloaded. Streaming biases to the next-needed pieces from
  the fastest seeders; bulk download fans wide. An **endgame** mode duplicate-fetches
  the final pieces so one slow seeder can't stall completion.

---

## 7. The discovery layer — a private DHT

**In plain terms.** Beyond asking the full node "who has this song," peers also keep
a distributed phone-book (a DHT) so discovery survives even if a node is busy. The
important property: this phone-book is **private** — it only talks to other
Bopwire nodes, never the public BitTorrent network.

**Under the hood.**
- The DHT speaks the standard Kademlia/KRPC wire format **but every packet carries a
  private network tag** (a top-level `"mc":"mcnet1"` key). Any inbound packet missing
  the tag is dropped at decode, so the public BitTorrent mainline can never enter the
  routing table and Bopwire nodes never answer it.

---

## 8. The relay layer — NAT traversal as a paid role

**In plain terms.** Most phones and home machines can't accept incoming
connections (they're behind NAT/firewalls). The **mini-node** is a relay on a
reachable address that bridges two such peers. Crucially, relaying is a **paid job**
— a mini-node earns a token every time it carries a stream or download — so there's an incentive
to run the infrastructure that keeps the network reachable.

**Under the hood.**
- A seeder's bytes flow `seeder → mini-node → downloader`. The relay forwards opaque
  `F`-frames (`relay.forward` for control, `relay-bin` for audio) and is
  content-agnostic — it doesn't parse the audio, it just bridges.
- The relay is also a deliberate economic anchor: routing the data path through a
  paid relay is what funds the reachable-IP layer. (Direct peer connections + ICE
  hole-punching exist in the stack but are off by default.)

---

## 9. The chain — blocks, proofs, and the play lifecycle

**In plain terms.** The blockchain is the shared, tamper-evident record of *what was
played and who got paid*. A play only becomes money if it passes anti-cheat rules —
you can't loop a 5-second clip or fake 10,000 plays to print tokens.

**The session lifecycle.** A play is a three-message conversation with the full node:

![Session lifecycle — start, then heartbeat while playing, then complete, then the anti-cheat gates and mint](d-session.svg)

**The anti-cheat gates (run at `session.complete`).**
- **Coverage gate:** the union of *distinct* position ranges actually heard must
  cover **≥50%** of the song's registered duration. Re-listening the same 5 seconds
  collapses to 5 seconds of coverage — you can't loop a clip to qualify.
- **Density gate:** heartbeats must arrive at a plausible rate (≥1 per ~10s of wall
  time), so you can't fast-forward and "complete" instantly.
- **Replay protection:** each `session_id` is single-use, persisted on-chain; a
  reused session is rejected.

**The proof.** A passing play produces a signed **`PlayProof`** — session id,
content hash, block hash, artist, listener, serving-node id, timestamps, duration,
heartbeat count, and (v2) the **seeder** and **mini-node** addresses — signed by the
full node's key. The node embeds it in a **`MintTx`** with the computed reward
outputs and applies it to the next block.

**Under the hood.**
- Transactions: `TRANSFER`, `MINT` (the play reward), `RELAY_REWARD` (legacy),
  `MODERATOR_OP`, `MODERATOR_PROPOSAL`, `USERNAME_REGISTER`, `SLASH`. State is
  LevelDB; pending txs sit in a mempool and a candidate-block builder drains them.
- `PlayProof` is **versioned by length** for forward-compatibility: legacy proofs
  deserialize without the seeder/mini fields and keep validating, so adding the new
  reward lanes required **no chain reset**.
- The chain is currently produced/validated by a single full node (a permissioned
  validator); light-client players don't validate blocks, they trust the node's
  signed results. Multi-validator consensus is a forward step (see §14).

---

## 10. Economics — the five reward lanes, supply, and burn

**In plain terms.** Every genuine play mints tokens to **everyone who made it
happen** — not just the artist. The token is the network's internal unit of account
and reward; it is consumed and earned by participation.

![The five reward lanes minted per qualifying play, and the post-10,000-play burn](d-rewards.svg)

Listening to popular artists spends tokens, and those tokens are burned at a variable
rate — taken out of circulation for good. What survives the burn flows the other way:
to the artists who made the thing and the seeders who keep it alive on the network. So
the economy runs as a loop, not a pump — value is drawn in at one end by attention and
paid out at the other to the people doing the actual work, with the burn sitting in
between as the throttle.

We have put some cybernetics ideas into this as well. When the network runs hot — more
plays, more transfers — more is burned, and supply tightens against its own activity;
when it runs cool, the pressure eases. There's no hand on the dial and no central bank.
So here, the burn is the *governor*: a negative feedback loop that holds the token near
equilibrium the way a steam engine's governor holds its speed. The upshot is that the
currency can't inflate itself into noise — the more it's used, the more disciplined it
becomes, and value keeps settling on the parts of the system that create and sustain
rather than the parts that merely consume.

To quote Stafford Beer:

> *"According to the science of cybernetics, which deals with the topic of control in
> every kind of system (mechanical, electronic, biological, human, economic, and so on),
> there is a natural law that governs the capacity of a control system to work. It says
> that the control must be capable of generating as much 'variety' as the situation to
> be controlled."*

In that sense, the burn is built in accordance with Ashby's Law (which is what Beer is
referencing here) — the requisite (negative-feedback) variety matched to the state of
the system that would otherwise inflate.

**Under the hood.**
- Precision: `1 token = 1e8` internal units (like satoshis). Each lane is exactly
  `1.00000000`.
- **Royalty splits:** the artist lane is divided by on-chain basis-points among
  collaborators/labels (validated to sum ≤ 100%; any remainder routes to an
  unclaimed escrow).
- **Supply control.** `SUPPLY_FLOOR = 1,000,000,000` tokens and `SUPPLY_CAP =
  2,000,000,000`. Below the floor there is **no burn** ("give the network away"
  bootstrap). Between floor and cap a per-play **listener burn** scales
  *cubically* — roughly: 1.0 B supply ⇒ 0 burned, 1.5 B ⇒ ~125, 1.8 B ⇒ ~512 — so
  usage gets deflationary pressure as the chain approaches the cap, and minting
  refuses past it.
- **The mini-node lane is per-stream, not per-byte.**

---

## 11. Wallets, transfers, and non-custodial escrow

**In plain terms.** Your wallet is a key on your device. Sending tokens is a message
*you* sign — the network never moves your money without your signature, and the
nodes never hold your private key. Artists' pending earnings sit in **escrow held by
the blockchain itself**, not by any operator's wallet.

**Under the hood.**
- **Transfers** are `TransferTx{from, to, amount, nonce, from_pubkey, signature}`.
  The signer covers `chain_id | from | to | amount | nonce | from_pubkey` with
  secp256k1; the chain verifies the signature, cross-checks
  `address_from_pubkey == from`, enforces a per-sender **nonce** (replay
  protection), and checks balance. The private key never leaves the device; a node
  only *validates and relays* a signed transaction (the `eth_sendRawTransaction`
  model). `MC_CHAIN_ID = 19779` is mixed into the signature so it can't replay on
  another chain.
- **Escrow is non-custodial.** A pre-threshold artist reward is minted to a
  **deterministic `escrow_address(artist)`** — funds the *chain* holds, not an
  operator. Release is a `RELEASE_ESCROW` proposal approved by a **moderator quorum**
  (majority vote), and it can only pay out **to the artist the escrow was created
  for**. No single party custodies the funds; a signer **cannot divert them to
  themselves**. The trusted part is *identity verification* (is this really the
  artist), not *custody* — a narrower role than holding a pot of money.

---

## 12. Security model

| Surface | Mechanism |
|---|---|
| **Value transfer** | secp256k1 signature + inline pubkey cross-checked to `from` address + per-sender nonce (anti-replay). |
| **Play minting** | Node-signed `PlayProof`; single-use `session_id`; 50% coverage + heartbeat-density gates. |
| **Piece integrity** | Per-256 KB SHA-256 manifest verified on arrival; whole-file `content_hash` backstop. |
| **Network isolation** | Private DHT tag drops all non-Bopwire packets; private bootstrap only. |
| **Escrow** | Funds chain-held at a deterministic address; destination-bound; quorum-released; not operator-custodied. |
| **Sybil / cheating** | Permissioned, vouched moderators; cubic burn past the play threshold; replay-proof session/nonce sets. |
| **Identity** | One secp256k1 key = peer-id = wallet address (single identity across network and ledger). |

---

## 13. Governance & moderation

**In plain terms.** A small, accountable set of moderators handles the things a
public vote *can't* safely do — verifying an artist's identity to release escrow,
and acting on takedown (DMCA) requests. New moderators must be *vouched for*, because
a flood of fake moderators approving fake identity claims would be an attack.

**Under the hood.**
- A three-tier role model on-chain: `FOUNDER` (bootstrap authority, can grant/revoke
  moderators), `OP` (proposes and votes on actions), `VOICE` (observer). Multi-step
  actions (hide content, release escrow) go through `ProposalTx` and execute on
  **OP-majority quorum** in-block — no single operator acts alone.
- **Takedown without cutting the artist:** a "hidden" track is delisted from
  discovery but its on-chain royalty entry persists — so a DMCA-complied song can
  still settle the artist's earnings even while it's removed from the catalogue.
- The shape is a **maintainer hierarchy** (think Linux/BDFL): a steward over an open
  base, intended to widen its operator and moderator set as the network grows.

---

## 14. Decentralization posture

Bopwire is **decentralized where it can be and centralized only where a public
process would be unsafe** — and it keeps those layers separate on purpose:

- **Decentralized:** content distribution (the swarm), discovery (private DHT +
  SwarmIndex), value transfer (self-signed transactions), and token issuance (fixed,
  algorithmic, minted by genuine plays — no pre-mine, no sale, no protocol fee to any
  central party).
- **Centralized by necessity:** identity verification and escrow *release*, and
  content takedown. These can't be a public vote without becoming an attack surface
  (fake-identity approval, or a mob withholding an artist's money). This is an
  **authorization** role — the chain holds the funds; moderators only *attest* and
  *can't divert* — not a custodial one.
- **Stewardship:** development and root authority currently rest with a founder
  operating as a node on the *same terms as any operator* (no special token reward,
  no fee, no allocation).

The design goal is a network whose **value is created by its participants and
recorded on a ledger no one privately owns**, with the unavoidable trust (identity,
takedown) confined to a narrow, accountable, non-custodial role.
