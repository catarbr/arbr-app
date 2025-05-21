# Arbr Technology & Architecture — **v0.92 (Engineering Review Draft)**

*Modular, privacy-first Human Fulfillment System™ with dual-mode (Natural-Language ↔ Touch) interaction*

> **What changed from v0.91?**
>
> * Added **Local Event Bus** for on-device messaging; SNS/SQS now cloud-only.
> * Optimised **NL stack** (quantised/ distilled models, RAM target ≤170 MB).
> * **Lazy / on-demand explainability** pipeline.
> * Hardened **Cloud-Boost tunnel** with E2E XChaCha20-Poly1305 + audit hash-chain.
> * Documented crypto standards, event-schema versioning, Windsurf guardrails.
> * Updated open items & DoD.

---

## 0 . Purpose of this Document

Locks the architecture for the 1.0 cut-off **with clarifications demanded by the Grok review**.
Any deviation requires an approved ADR in `/adr/`.

---

## 1 . Experience Model (UI × NL)

| Layer                    | Goal            | Primary   | Secondary    | Fallback   |
| ------------------------ | --------------- | --------- | ------------ | ---------- |
| **Micro-actions**        | Instant control | Touch     | One-shot NL  | Touch      |
| **Reflection & Insight** | Rich empathy    | NL conv.  | Visual cards | UI summary |
| **Setup & Permissions**  | Clarity & trust | Guided NL | Stepper UI   | UI         |

*All behaviour flows through the **Intent API** (§ 4.1).*

---

## 2 . Architectural Principles

1. **Local-First by Default** — data leaves device only via zero-knowledge channels.
2. **Domain-Driven Modularity** — Identity, Personal, Relational, Fulfilment, Guardian, Trust.
3. **Dual Event Spine**

   * **Local Event Bus (LEB)** — ultra-light in-process Pub/Sub for on-device comms.
   * **SNS/SQS** — cloud & sync events only.
4. **Dual-Mode Interaction as First-Class** — every capability addressable via Intent.
5. **Lazy, Verifiable Explainability** — generated on demand, cached, hash-linked to actions.
6. **Zero-Regret Privacy** — if optimisation vs privacy, privacy wins.

---

## 3 . High-Level Component Map

```
┌ Presentation ──► Interaction Orchestrator ──► Domain Contexts ──► Shared Services
│  (React UI & NL)    (Intent Router, Dialog)      (DDD)         (Encrypted DB, ML, Sync)
│                         ▲│                              
│            Local Event Bus│                              
└──────────────────────────┘                              
```

*Cloud events (sync, push) exit device via SNS/SQS.*

---

## 4 . Technical Blueprint

### 4.1 Intent API

* `verb` • `payload` • `source` • `trace_id` • `schema_ver`
* **Schema versioning**: `semver` field; breaking changes bump major.
* Spec lives in `schema/intent.proto`; LEB + cloud share same envelope.

### 4.2 Natural-Language Stack

| Function       | Tech                          | Footprint | Notes                                                                                                                                             |
| -------------- | ----------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| STT            | Whisper-tiny-int8 (quantised) | 23 MB     | ≤80 ms latency                                                                                                                                    |
| Intent         | MiniBERT-Q8 + rule lattice    | 12 MB     | 25 ms median                                                                                                                                      |
| Response       | DistilGPT-Lite-int8           | 38 MB     | 45 ms median                                                                                                                                      |
| Total RAM goal | **≤170 MB** on iPhone SE 2    |           |                                                                                                                                                   |
| Cloud Boost    | Bedrock/SageMaker LLM         | opt-in    | **End-to-end E2E** XChaCha20-Poly1305; client-side ephemeral keys, server stateless; hash-chain audit stored locally & in Transparency Dashboard. |

### 4.3 Touch UI Stack

* React Native 0.74 • Tamagui 1.x • React Navigation 7 (lazy).
* Frame budget < 16 ms (iPhone 12).

### 4.4 Data & Sync

| Layer    | Tech                                      | Notes                            |
| -------- | ----------------------------------------- | -------------------------------- |
| Local DB | SQLite + SQLCipher (AES-256-GCM)          | Graph schema `neomorph.db`       |
| Sync     | Opt-in zero-knowledge S3 + Lambda         | Incremental, compressed          |
| Realtime | CRDT (Yjs) behind `pods-sync` + WebSocket | Conflict-free; local merge first |

### 4.5 Security & Trust

| Concern          | Solution                                                                         |
| ---------------- | -------------------------------------------------------------------------------- |
| At-rest crypto   | AES-256-GCM, keys in Secure Enclave                                              |
| In-transit       | TLS 1.3 + XChaCha20 overlay for Cloud-Boost                                      |
| Key backup       | iCloud Keychain (opt-in)                                                         |
| Explainability   | Lazy generator → caches JSON blob; `/explain-decision` returns ≤100 ms if cached |
| Audit            | EventStore (hash-chain) surfaced in Dashboard                                    |
| Analytics toggle | Single switch disables Matomo & event export                                     |

---

## 5 . DevEx & CI/CD ( **Windsurf additions** )

| Stage                | Tooling                                     | Apple-Silicon Ready | Windsurf Impact                                                  |
| -------------------- | ------------------------------------------- | ------------------- | ---------------------------------------------------------------- |
| Editor / AI          | **Windsurf IDE 1.x**                        | —                   | `.windsurf.json` enforces local-only AI; cloud telemetry blocked |
| Extension whitelist  | ESLint, Prettier, Tamagui, NX, Jest, Docker | —                   | Monthly audit; script `audit-windsurf-ext.sh`                    |
| Pre-commit           | Husky hook `check-windsurf-telemetry.sh`    | n/a                 | Fails commit if cloud assist toggled                             |
| Docker dev-container | `arbr-dev` multi-arch                       | ✅                   | One-click open-in-container verified                             |
| CI runners           | GitHub Actions macOS-M1 & Linux             | ✅                   | heavy ML tests shifted to CI                                     |

---

## 6 . Domain Contexts & Event Flow

Local intents → **LEB** → Domain service → optional cloud event (SNS/SQS) → sync/ML → result via LEB → Orchestrator → UI/NL.

---

## 7 . Scalability & Performance Targets

| Metric        | Target                             |
| ------------- | ---------------------------------- |
| Cold start    | ≤ 2.8 s on iPhone SE 2             |
| Idle RAM      | ≤ 200 MB                           |
| Battery drain | < 2 % per 30 min active NL session |
| Trust-score   | ≥ 0.98 (usability study)           |

---

## 8 . Privacy & Ethics — Implementation Specifics

| Item              | Detail                                                               |
| ----------------- | -------------------------------------------------------------------- |
| Encryption stds   | AES-256-GCM (at rest), XChaCha20-Poly1305 (E2E tunnel), Argon2id KDF |
| Cloud-Boost audit | SHA-256 hash-chain; proof displayed in Dashboard                     |
| Data export       | Portable encrypted JSON + schema\_ver                                |
| CRDT conflicts    | Resolved locally; last-writer-wins only for non-semantic fields      |

---

## 9 . Open Items to 1.0

| # | Topic                                        | Owner     | Date   |
| - | -------------------------------------------- | --------- | ------ |
| 1 | Verify RAM on iPhone SE 2 after quantisation | ML Guild  | 15 Aug |
| 2 | Local Event Bus debug tooling                | Platform  | 22 Aug |
| 3 | Pod-sync CRDT edge-case test plan            | Collab WG | 30 Aug |
| 4 | Cloud-Boost tunnel third-party audit         | Security  | 05 Sep |
| 5 | Trust Manifesto legal sign-off               | Ethics    | 10 Sep |

---

## 10 . Definition of Done (1.0)

* ≥ 95 % intents usable via both NL & Touch
* Performance, RAM, battery targets met on baseline devices
* Privacy & security audits passed; Cloud-Boost audit published
* All ADRs for open items merged or closed

---

*Questions → start a thread & tag #arch-core.  All changes require an ADR.*