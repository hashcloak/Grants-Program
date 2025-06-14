# zk‑ink Toolkit - Web3 Foundation Grant Application

* **Team Name:** HashCloak
* **Payment Address:** Fiat (details to be shared upon approval)
* **Level:** 3

---

## Project Overview

### Overview

`zk‑ink` is a developer‑friendly toolkit that unlocks verifiers for **two complementary zero‑knowledge tool‑chains**:

* **Circom / Noir workflow** – translate circuits written in Circom or Noir into compact **ink!** verifier smart‑contracts for **Groth16, Plonk, FFLonk, and Ultrahonk** (all share an R1CS‑ or Plonkish‑style constraint system).
* **Plonky3 workflow** – compile circuits written in Plonky3’s native Rust DSL (FRI‑based, STARK‑style constraints and **incompatible with Circom/Noir**) into equally‑efficient **ink!** Plonky3 verifiers.

> This dual‑path design lets Substrate builders pick the circuit language that best fits their needs while always ending with a production‑ready verifier on‑chain.

#### Why this particular mix of proving systems?

| System        | Setup                          | Proof size / verify cost     | Distinct strength               | Polkadot relevance                                                                                                  |
| ------------- | ------------------------------ | ---------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Groth16**   | Circuit‑specific trusted setup | \~192 B / single pairing     | Smallest proofs, battle‑tested  | Ideal for **privacy‑preserving fungible assets** on resource‑constrained parachains (e.g., Shielded DOT transfers). |
| **Plonk**     | Universal & updatable          | \~1 KB / \~2 pairings        | No per‑circuit ceremony         | Lowers barrier for many **parathreads** to add ZK gadgets without running a setup each time.                        |
| **FFLonk**    | Universal; FF‑friendly         | \~0.8 KB / 1 multi‑exp       | Field‑agnostic variant of Plonk | Serves **runtime modules** that already rely on Grumpkin or other exotic curves.                                    |
| **Ultrahonk** | Universal; recursion‑optimized | \~1.5 KB / hash‑based verify | Fast recursive aggregation      | Enables **roll‑up style parachains** and light clients that compress thousands of proofs into one.                  |
| **Plonky3**   | Transparent (no setup)         | \~2–5 KB / hashing only      | STARK‑like, Turbo‑recursive     | Perfect for **fast‑finality bridges** and **decentralised sequencer roll‑ups** where recursion depth matters.       |

Including all five gives Polkadot builders a menu that spans **low‑cost static circuits (Groth16)** to **massively‑recursive roll‑ups (Plonky3/Ultrahonk)**—crucial for heterogeneous parachain architectures.

### Project Details

#### 1. Mock‑ups / UX Sketches

`zk-ink` is a **CLI‑first** tool; the primary “UI” is the command‑line plus generated artifacts. A minimal interactive workflow is:

```bash
# install zk‑ink CLI (one‑time)
cargo install --git https://github.com/hashcloak/zk-ink --locked

# initialise a new project (Circom or Noir template)
zk-ink new my_circuit --template noir

# compile and prove the circuit
zk-ink build circuits/transfer.circom --prover plonk --target ink

# deploy the generated verifier contract to Rococo
cargo contract upload --suri "//Alice" --code target/transfer_verifier.wasm
```

The CLI prints a TUI progress bar and writes:

* `transfer_verifier.wasm` — ready‑to‑deploy ink! contract
* `transfer_verifier.json` — metadata/vk in JSON
* `prove.rs` — Rust helper to generate proofs programmatically
* `transfer_proof.json` — proof file containing field elements serialized in JSON (uniform across all backends)

A future VS Code extension will surface the same commands via an action palette, but it is **not** part of this grant’s scope.

#### 2. Data Models / API Specifications

| Artifact               | Format                                                           | Purpose                                                                                  |
| ---------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Verifying Key (vk)** | JSON file | Consumed by the verifier contract at deploy‑time.                                        |
| **Proof Object**       | JSON across all backends                               | Passed directly to `verify_proof`; contains serialized field elements and public inputs. |
| **Contract ABI**       | ink! metadata JSON (generated by `cargo-contract`)               | Enables any front‑end to call `verify_proof(bytes proof, bytes public_inputs) -> bool`.  |

A full Rust trait shared by all verifier crates:

```rust
pub trait InkVerifier {
    /// Returns `true` if the proof is valid for the given public inputs.
    fn verify_proof(proof: Vec<u8>, public_inputs: Vec<u8>) -> bool;
}
```

#### 3. Technology Stack

* **Rust `no_std`** for verifier crates (ink! compiles to WASM‑32).
* **Arkworks** for BN254 & BLS12‑381 algebra, **Plonky3** native library for Goldilocks.
* **circom‑compat (arkworks‑rs)** for Rust‑native Circom proof generation.
* **noir‑rs (zkmopro/bb backend)** for compiling Noir circuits and generating proofs.
* **Cargo‑contract** & **ink! 5.0** for contracts.
* **Typescript** (Node >=18) for the SDK & CLI.
* **GitHub Actions** CI: unit tests + Rococo integration.

#### 4. Architecture & Components

Developers author circuits in their language of choice; **all subsequent steps are carried out through the single entry‑point `zk‑ink` CLI**, which internally wraps existing compilers/provers (no need to invoke them directly).

```
  Developer Circuit (Circom / Noir / Plonky3)
                     |
                     v
              +------------------+
              |    zk‑ink CLI    |  <— compile, prove, deploy
              +--------+---------+
                       |  leverages
       +---------------+----------------+
       | circom‑compat │ noir‑rs │ plonky3‑prove |
       +---------------+----------------+
                       |
       +---------------+----------------+
       |   Generated Artifacts          |
       |  • ink! verifier  (.wasm)      |
       |  • metadata/vk   (.json/.ron)  |
       |  • proof file     (.json)       |
       +---------------+----------------+
                       |
                       | ABI
                       v
                +--------------+
                | On‑chain ink! |
                |   Contract   |
                +--------------+
                       |
                       v
              +------------------+
              |  SDK (TS / Rust) |
              +------------------+
```

* **CLI commands**: `zk‑ink build` (compile + vk export), `zk‑ink prove` (generate proof), `zk‑ink deploy` (upload verifier), `zk‑ink verify` (local check).
* **Internal wrappers** keep up with upstream tool changes, shielding end‑users from version hell.
* The **SDK** consumes the same proof blob format, making dApp integration uniform across proving systems.

#### 5. Out‑of‑Scope / Limitations Out‑of‑Scope / Limitations

* **Front‑end GUIs** (VS Code extension, web dashboards) are NOT part of this grant.
* **Proof generation back‑ends** beyond Circom, Noir, and Plonky3 (e.g., Risc Zero) are future work.
* **Hosting / deployment** costs, **maintained dev‑ops**, and **tokenomics** are explicitly excluded per W3F guidelines.

### Ecosystem Fit

Below is a non‑exhaustive snapshot of Web3 Foundation grants touching privacy/ZK and how **zk‑ink** differentiates or complements them:

| Past Grant                                                                | Focus & Deliverable                                                                                                                              | Gap Filled / Relation to `zk‑ink`                                                                                                                     |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- | 
| **Nullifier Prime** – Compliant & Programmable Privacy 【turn1view0†L1-L6】 | EVM‑fork parachain with automatic shielding/unshielding & wallet UX.                                                                             | Needs efficient on‑chain verifiers; `zk‑ink` can supply Groth16/Plonk verifiers for its shielded pools.                                               |                                  |                                                                   |
| **Zkverse** – zkPallet suite 【turn1view1†L1-L6】                           | Generic pallets integrating bellman & Plonk libraries; dev‑oriented examples.                                                                    | `zk‑ink` offers ink! **contracts** rather than pallets and adds Ultrahonk & Plonky3 plus a language‑agnostic SDK.                                     |                                  |                                                                   |
| **ZK‑Snarks Tutorial** 【turn1view2†L1-L5】                                 | Educational blog + sample Groth16 verifier pallet.                                                                                               | Tutorial only; our project is production‑grade tooling with five proving systems.                                                                     |                                  |                                                                   |
| **zk‑Plonk pallet** 【turn2view0†L1-L4】                                    | Single pallet implementing Plonk.                                                                                                                | `zk‑ink` supports **four additional provers**, ink! WASM targets, and turnkey CLI.                                                                    |                                  |                                                                   |
| **Ruby Protocol** – Functional encryption 【turn2view1†L2-L7】              | Middleware for encrypted data monetization (uses FE, not SNARKs).                                                                                | Could leverage `zk‑ink` verifiers for zero‑knowledge policy proofs.                                                                                   |                                  |                                                                   |
| **pallet‑MACI** 【turn2view2†L0-L4】                                        | Anti‑collusion quadratic funding pallet.                                                                                                         | MACI on Polkadot will need Plonk‑style verifiers—`zk‑ink` provides drop‑in contracts + SDK.                                                           |                                  |                                                                   |
| **Webb Mixer** 【turn3view0†L1-L5】                                         | Bulletproofs‑based mixer pallets.                                                                                                                | We extend options with Groth16/Plonk mixers and recursive proofs via Ultrahonk.                                                                       |                                  |                                                                   |
| **ZeroPool** 【turn3view1†L1-L4】                                           | Optimistic rollup–style anonymity pool.                                                                                                          | Could migrate verifier to ink! through our SDK for better composability with parachains.                                                              |                                  |                                                                   |
| **Evanesco Network** 【turn3view2†L2-L7】                                   | Privacy DeFi parachain layer.                                                                                                                    | Needs multi‑prover support for different dApps; `zk‑ink` offers lightweight verifier contracts.                                                       |                                  |                                                                   |
| **ZK Rollups pallet** 【turn4view0†L2-L7】                                  | Layer‑2 scalability pallet targeting Plasm.                                                                                                      | Recursive aggregation via Ultrahonk/Plonky3 in `zk‑ink` can compress rollup proofs.                                                                   |                                  |                                                                   |
| **Manta Network** – Privacy DEX/Payments 【turn4view1†L2-L4】               | zkSNARK‑based shielded payments & AMM DEX.                                                                                                       | Our multi‑prover verifiers give Manta flexibility to iterate on proof systems without rewriting contracts.                                            |                                  |                                                                   |
| **KILT Anonymous Credentials** (speculative)                              | Credential issuance with selective disclosure proofs.                                                                                            | Could embed lightweight Groth16/Plonk verifiers from `zk‑ink`.                                                                                        |                                  |                                                                   |
| **Aleph Zero** – Liminal privacy chain                                    | Substrate L1 using ink! contracts and zk‑SNARK–based asset mixer (Liminal).                                                                      | `zk‑ink` can standardize multi‑prover verifiers (Groth16→Plonky3) for Aleph Zero’s mixers and recursive bridges, eliminating bespoke implementations. |                                  |                                                                   |
| **zkMega** – Zero‑knowledge tool‑set (Patract Labs)                       | Early Groth16‑based verifier library & modified ink! for ZK Rollups ([github.com](https://github.com/patractlabs/zkmega?utm_source=chatgpt.com)) | Laid groundwork but limited to Groth16; `zk‑ink` delivers multi‑prover (Plonk→Plonky3) support and a polished SDK.                                    |                                  |                                                                   |
| **Zerochain** (speculative)                                               | Privacy‑first parachain concept.                                                                                                                 | Toolkit accelerates dev speed by abstracting verifier generation.\*\* (speculative)                                                                   | Privacy‑first parachain concept. | Toolkit accelerates dev speed by abstracting verifier generation. |

**Contrast in one sentence:** Whereas prior grants mostly deliver **single‑purpose pallets, dApps, or tutorials**, **zk‑ink** is core **tooling infrastructure**—a reusable SDK and suite of ink! verifier crates spanning **five modern proving systems**, closing the integration gap for every future privacy‑oriented parachain or contract on Polkadot.

---

## Team

| Name             | Role                         |
| ---------------- | ---------------------------- |
| **Manish Kumar** | Cryptography Engineer (lead) | 
| **Alex Liao**    | Cryptography Engineer        |
| **Teresa Li**    | Project Manager              | 

**Legal structure:** HashCloak Inc., 34 Minowan Miikan Lane, Toronto, Canada.

**Contact Name:** Pankaj Persaud - Operations Manager

**Contact Email:** pankaj@hashcloak.com

**Team website:** hashcloak.com

### Team’s experience

HashCloak Inc. is a cryptography R&D lab and consultancy helping teams build secure, scalable, and privacy-preserving blockchain infrastructure. From advancing privacy tools to strengthening cryptographic security and optimizing blockchain performance, we bridge research and real-world deployment. Our expertise spans protocol design, secure computation, and cryptographic audits, ensuring teams can confidently integrate cutting-edge cryptography into their systems.

We've worked across different ecosystems to build core cryptography and integrate ZK tooling. Some of our recent work includes:
- Implemented proof verifiers in Sway for the Fuel ecosystem: https://github.com/hashcloak/sway-zkp-verifiers
    - We wrote a complementary tutorial: https://github.com/hashcloak/sway_zkp_tutorial
- Implemented privacy-preserving logistic regression using Noir and Co-noir: https://github.com/hashcloak/noir-mpc-ml
- Implemented BLS signatures and the ED25519 signature scheme along with their underlying primitives in Sway for the Fuel ecosystem: https://github.com/hashcloak/fuel-crypto
- Designed and implemented a pairing-friendly curve that uses the Starknet curve as the basefield: https://github.com/hashcloak/starkjub
- 
An overview of our publicly available work is at https://github.com/hashcloak.

---

## Development Roadmap

### Overview

* **Total estimated duration:** 12 weeks
* **Full‑Time Equivalent (FTE):** 2.2
* **Total costs:** **62 000 USD**


### Milestone 1 – Groth16 Verifier & CLI Skeleton

* **Estimated duration:** 3 weeks
* **FTE:** 2.2
* **Costs:** **17 000 USD**

| No | Deliverable            | Specification                                         |
| -- | ---------------------- | ----------------------------------------------------- |
| 0a | License                | Apache‑2.0                                            |
| 0b | Documentation          | Inline Rust docs + quick‑start guide                  |
| 0c | Testing                | ≥ 90 % unit‑test coverage; basic ink! integration     |
| 1  | `ink_zk_groth16` crate | Pairings on BN254 & BLS12‑381, `verify_proof()` ABI   |
| 2  | CLI scaffold           | `zk‑ink build` & `zk‑ink prove` for Groth16           |
| 3  | Benchmarks             | Gas report: Groth16 verifier execution on ink! |

### Milestone 2 – Plonk & FFLonk + Unified ABI

* **Estimated duration:** 3 weeks
* **FTE:** 2.2
* **Costs:** **18 000 USD**

| No | Deliverable           | Specification                                                         |
| -- | --------------------- | --------------------------------------------------------------------- |
| 0a | `ink_zk_plonk` crate  | Plonk (BN254 & BLS12‑381)                                             |
| 0b | `ink_zk_fflonk` crate | FFLonk (BN254 & BLS12‑381)                                            |
| 0c | Testing               | Unit & integration tests for Plonk/FFLonk verifiers; CI pass required |
| 1  | Unified ABI           | Same `verify_proof()` signature across crates                         |
| 2  | CLI upgrade           | Support `--prover plonk` flag                                         |
| 3  | Benchmarks            | Gas report: Plonk and FFLonk verifier execution on ink!        |

### Milestone 3 – Plonky3 & Ultrahonk

* **Estimated duration:** 3 weeks
* **FTE:** 2.2
* **Costs:** **19 000 USD**

| No | Deliverable              | Specification                                                                |
| -- | ------------------------ | ---------------------------------------------------------------------------- |
| 0a | `ink_zk_plonky3` crate   | Native Plonky3 field support (Goldilocks or others)                          |
| 0b | `ink_zk_ultrahonk` crate | Ultrahonk (BN254 only) – Halo‑style recursion stub                           |
| 0c | Testing                  | Unit & integration tests for Plonky3 & Ultrahonk verifiers; CI pass required |
| 1  | CLI upgrade              | Support `--prover plonky3` & `ultrahonk`                                     |
| 2  | Benchmarks               | Gas/weight report: Plonky3 and Ultrahonk verifier execution on ink!          |

### Milestone 4 – SDK & Documentation

* **Estimated duration:** 3 weeks
* **FTE:** 2.2
* **Costs:** **8 000 USD**

| No | Deliverable           | Specification                                                          |
| -- | --------------------- | ---------------------------------------------------------------------- |
| 0a | Typescript & Rust SDK | Helper functions for proof generation & verification                   |
| 0b | Documentation site    | mdBook-based docs.rs site with API reference and getting‑started guide |
| 0c | Testing               | End-to-end tests covering SDK interactions with all verifier contracts |

---

## Future Plans

In terms of features, 
* Add STARK‑based verifiers (RiscZero‑style) and recursive aggregation.
* Publish `ink!` privacy primitives library and maintain long‑term support.

As for sustainability, we have prospective clients who are interested in building in the Polkadot/Kusama ecosystem. Through our work with those clients, we aim to maintain and upgrade this tool. If this project is a good fit for other funding in the Polkadot/Kusama ecosystem, then we are open to applying for extra funding through those venues as well. 

## Additional Information

We first learned about the Web3 Foundation Grants Program through a personal recommendation from a grants‑committee member several years ago; after iterating on ideas, we now have a conviction‑driven proposal that we are eager to build for the Polkadot community.
