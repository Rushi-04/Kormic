# Kormic System — Complete Developer Guide & Architecture

Welcome to the Kormic System. This architecture implements a **Zero-Trust, Pedigree-based security model for AI agents**. 

As AI agents become more autonomous, standard security practices (like giving an agent a permanent API key or a static UUID) are no longer safe. Kormic ensures that hijacked AI agents cannot exceed their blast radius, steal credentials, or hide their tracks, while allowing infrastructure to instantly globally revoke them if they misbehave.

This document breaks down the entire system from Phase 1 to Phase 2.5, explaining exactly **What** we built, **How** it works, **Why** we built it, and **Why alternative methods were rejected**.

---

## Phase 1: Core Pedigree & Immutable Identity

### The Problem
Traditional software uses standard usernames, passwords, or UUIDs (e.g., `user-1234`). But if an autonomous AI agent starts buying 10,000 stocks a second, a UUID doesn't tell us *who legally owns that agent* or *what it was designed to do*.

### What We Implemented: Structured Identity & Birth Records
We created a cryptographic **Birth Record** for every agent. The agent's ID looks like this:
`KMC.CMP.agent_name.0001.<real_id_hash>`

### How We Implemented It
When an agent is registered via the `AgentManager`, it is assigned a `Pedigree`. This pedigree contains the Birth Record. The Central Authority mathematically signs this birth record using **Post-Quantum ML-DSA-44 cryptography** to ensure it can never be forged, even by future supercomputers.

### Why We Implemented It
To legally and cryptographically bind an AI agent to a real-world entity (like a corporation's DUNS number). If the agent goes rogue, we know exactly who is responsible.

### Why Not Other Ways?
* **Why not JSON Web Tokens (JWTs)?** JWTs are easily stolen and lack long-term historical context. 
* **Why not standard Database Rows?** Database rows can be silently altered by a rogue admin. A cryptographically signed Birth Record is mathematically immutable.

---

## Phase 1.5: Tamper-Evident History & FAST Verification

### The Problem
Agents perform thousands of actions. If an agent gets hacked, the hacker's first move is to delete the logs to hide their tracks. Furthermore, verifying a massive history log every time an agent wants to act is too slow.

### What We Implemented: The History Chain
A cryptographically chained history log, mathematically summarized into a single 64-character hash called the `running_head`.

### How We Implemented It
Every time the agent performs an action (e.g., "Sent email to Bob"), we hash that action *together with the hash of the previous action*. This creates an unbroken chain. To verify the agent, Kormic uses **FAST Verification (O(1))** which simply checks if the current `running_head` matches the expected state, without needing to read the whole history.

### Why We Implemented It
To guarantee that no action can be secretly deleted or altered. If a hacker deletes a log from the middle of the chain, the final `running_head` hash will completely change, instantly exposing the tampering.

### Why Not Other Ways?
* **Why not just use standard server logs (Splunk/Datadog)?** Traditional logs are append-only by convention, but a compromised server allows hackers to edit them. Our History Chain makes tampering mathematically impossible to hide.

---

## Phase 2: Global Scale & Catastrophic Recovery

### The Problem
If the main Kormic database is destroyed in a fire, all agent histories are lost. Furthermore, if an agent in India tries to do something, checking its signature against a database in New York introduces massive latency.

### What We Implemented: Recovery Twins & Bloom Filters
1. **Encrypted Recovery Twins (Backups):** Snapshot backups of the entire system.
2. **Shamir Secret Sharing:** A method to split the backup password into 5 physical pieces.
3. **Regional Replicas:** Edge servers that sync a "Revocation List".

### How We Implemented It
We lock the system backups using military-grade **AES-256-GCM** encryption. Instead of giving one person the master password, we split the password into a polynomial equation (Shamir's Secret Sharing) given to 5 executives. To restore the database, 3 of the 5 executives must combine their pieces.
For speed, we push a highly compressed bit-array (a **Scalable Bloom Filter**) to Regional Replicas.

### Why We Implemented It
To survive a total catastrophic server loss without relying on a single point of failure (no single rogue employee can steal the backup). Bloom filters allow edge-servers to instantly check if an agent is globally revoked in less than a millisecond.

### Why Not Other Ways?
* **Why not standard AWS RDS Backups?** Standard backups rely on trusting the Cloud Provider and the IT Admin. Kormic is Zero-Trust.
* **Why not standard database replication?** Full database replication across the globe is expensive and slow. Bloom filters use almost zero memory and allow lightning-fast revocation checks.

---

## Phase 2.5: Runtime Countermeasures

### The Problem
We looked at real-world breaches:
1. **The Dialogflow Incident:** Multiple agents shared a server. A hacker broke into one agent, and used it to read the private conversations of *other* agents on the same server.
2. **The Replit Incident:** A standard developer agent accidentally ran a destructive script that deleted production databases because it had "ambient" (always-on) permissions.

### What We Implemented: Sandbox, Challenge-Response, and Drift HALT
1. **Capability Manifests (C1):** A strict list of exact tools and network addresses the agent is allowed to touch.
2. **FAST Challenge-Response (Gap 1):** Agents are born with a unique cryptographic Private Key.
3. **Anti-Replay & Fail-Closed Logic:** Hardened verification gates block stolen tokens and keyless agents.
4. **No Ambient Authority (C2):** Agents do not get standing API keys. 
5. **Drift HALT Wiring:** Automatic instant-kill switches.

### How We Implemented It
* **The Sandbox:** Wraps the agent's code. If the agent asks to use a tool not in its Manifest, the Sandbox throws a `PermissionError`.
* **Challenge-Response:** When an agent boots up, the Sandbox gives it a random math puzzle. The agent must solve it using its Private Key. If a hacker steals a session token but doesn't have the key, they fail the challenge.
* **Anti-Replay & Fail-Closed:** The Verifier engine issues a fresh 5-minute single-use nonce challenge and maintains a memory-safe cache of spent nonces to physically block stolen token replays. The system mathematically "fails closed," meaning an empty or missing signature instantly blocks the credential instead of accidentally passing.
* **Credential Root:** If an agent wants to do a destructive action (e.g., `refund:money`), it must ask Kormic for a 5-minute scoped token. Kormic only issues it if the manifest explicitly flags it as an `irreversible_scope`.
* **Drift Wiring:** If the Sandbox blocks an illegal action, the `SessionController` immediately logs a `MANIFEST_BREACH` and tells the `BehaviorMonitor` to issue a `HALT`. The agent is globally revoked instantly.

### Why We Implemented It
To ensure that even if an agent's "brain" (LLM) is completely hijacked by a prompt injection, the physical infrastructure mathematically refuses to let it reach outside its designated blast radius. 

### Why Not Other Ways?
* **Why not rely on Cloud IAM Roles (AWS/GCP)?** Cloud IAM roles are "ambient". Once a server boots up, it has those permissions forever. If a hacker gets in, they inherit everything. The architecture forces granular, 5-minute cryptographic verifications per action.

---

## Phase 3: Crown Jewels (Security Lead)

### The Problem
If the master encryption keys are kept in software, a hacker who gains root access to the physical server could theoretically extract the Master Key and destroy the system. Furthermore, no single human should ever have the power to destroy the entire AI fleet.

### What We Implemented: Hardware Key Isolation, Threshold Ceremony & Self-Defense
1. **Hardware Key Isolation:** Decoupled the Cryptography interface so that the backend can seamlessly swap from `SoftwareKeyCustody` to YubiKeys or AWS KMS without rewriting the architecture.
2. **Threshold Ceremony:** Mathematical quorum validation (Shamir Secret Sharing) applied to all existential actions.
3. **Self-Defense (Isolation):** Agents are granted the power to pull their own plug if they detect manipulation, but are mathematically barred from irreversible self-destruction.

### How We Implemented It
* **Existential Quorums:** The `ThresholdCeremony` class intercepts requests for `create`, `restore`, and `destroy`. It mathematically demands `3-of-5` physical shares to execute standard actions.
* **Cryptographic Quorum Validation:** To ensure the threshold ceremony isn't tricked by fake/dummy shares (which successfully produce invalid "garbage" keys under raw Shamir math), the ceremony reconstructs the key and runs a constant-time HMAC validation against a known SHA-256 secure hash to mathematically guarantee the true master key was reformed.
* **Catastrophic Quorum:** A command to wipe all agents (`destroy_all`) is hardcoded to require the MAXIMUM quorum (`5-of-5`). No single "God Key" exists.
* **Reversible Isolation:** The `SessionController.self_isolate()` method allows the agent to trigger a `SELF_ISOLATION` event, immediately placing itself on the Global Revocation list while firing a PagerDuty alert.

### Why We Implemented It
To eliminate single points of failure at the human/administrative layer. An angry employee cannot destroy the fleet. A hijacked agent cannot wipe its own history. The architecture enforces separation of powers mathematically.

---

## Phase 4: MeshKor SDK & Real-Time Data Pipeline

### The Problem
Engineers cannot be expected to write manual signature schemes, construct HTTP APIs, and handle Shamir Thresholds just to enroll an agent. Furthermore, if Kormic is slowing down the agent's workflow by taking 12ms to write every log to the disk, the engineering team will remove it.

### What We Implemented: The MeshKor Python SDK & Sub-Millisecond Piping
1. **The MeshKor SDK:** A clean, drop-in SDK (`meshkor` package) with three simple components: `MeshKorAgent`, `ReceiverClient`, and `LocalAuthority`.
2. **Sub-Millisecond SQLite Writes:** Real-time event piping.
3. **Concurrency Safety & Distributed Nonces:** Strict thread-safe per-agent locking.

### How We Implemented It
* **The SDK:** The user simply imports `from meshkor import MeshKorAgent`, and the SDK abstracts away all `Pedigree`, `ProofToken`, and `CredentialRoot` complexity. The `ReceiverClient` does the same for the resource side.
* **Database Optimization:** We upgraded the backing `SQLiteRecordStore` to use `Write-Ahead Logging (WAL)` and persistent connection pooling. We implemented a **Per-Agent Lock** and a **Connection Lock** so that hundreds of multi-threaded AI events can be recorded simultaneously without dropping a single event, dropping latency to **<0.9ms per event**.
* **Global Nonce Purging:** Cross-replica replay protection is now mandatory by default (`central_sync`). Replicas "union" their spent nonces upon syncing, and the Central Authority globally purges any nonces older than the 300-second freshness window to prevent snapshot memory bloat.

### Why We Implemented It
To transition the system from a mathematical proof-of-concept into a production-ready software product that can be integrated into existing infrastructure with three lines of code.

---

## Phase 5: Two-Tier Identity & Containment

### The Problem
If a vendor builds an AI agent and deploys it to two different hospitals, using a single AIN means both deployments are indistinguishable. If the vendor's signature is required for everything, the hospitals cannot operate independently. Conversely, if only the hospital signs it, there's no way to prove what software is actually running, or safely recall a vulnerable version globally. Furthermore, if they use the same global event hash salt, an attacker could theoretically correlate Hospital A's actions with Hospital B's actions.

### What We Implemented: BAIN, DAIN, and Isolation
1. **Build AIN (BAIN):** A cryptographic identity (`KMC.BLD...`) for the software release, sealing the maximum allowed permissions (the Envelope).
2. **Deployment AIN (DAIN):** A cryptographic identity (`KMC.DPL...`) for the specific running instance, which cryptographically links to the BAIN.
3. **Cascading Kill Switch:** Revoking the BAIN implicitly revokes all child DAINs.
4. **Storage Isolation:** Each DAIN receives a unique sidecar salt, stored only locally.

### How We Implemented It
* **Containment Check:** During DAIN enrollment in `AgentManager`, the system strictly validates that the requested scopes are a subset of the parent BAIN's envelope. It refuses scalar over-grants or out-of-envelope tools.
* **Two-Tier Verification:** The FAST engine was refactored to verify *both* the DAIN and BAIN signatures in O(1) time. The `VerificationResult` now labels whether it verified the full deployment or just the build (`verified_scope`). *Important: Integrators calling in `build_only` mode MUST check that `verified_scope == "full"` before treating an agent as fully authenticated, as build-only skips the cryptographic head/history checks.*
* **JSON Null-Strip Protection:** The cryptographic parsing was hardened so that standard JSON transmission (which strips `null` keys) cannot accidentally alter the signature payload, protecting cross-network routing.
* **Salt Persistence:** The `SQLiteRecordStore` now features a dedicated `salts` table that physically segregates the private hashing salt from the public Pedigree payload.
* **Artifact Binding & Vendor Enrollment Subsystem:** When registering a BAIN, the system mathematically verifies a vendor's signature over the code's true digest (`artifact_digest`). Furthermore, the system implements a strict, globally verified **Vendor Enrollment Process**:
  * **Bind-Once Namespaces:** A vendor's public key must be securely enrolled in the distributed registry before they are allowed to issue BAINs. We enforce a strict `Bind-Once` invariant—if a namespace (e.g., "Acme") is ever enrolled, the Central Authority mathematically refuses any attempt to overwrite or rebind it, completely shutting down namespace squatting via silent re-enrollment.
  * **Proof of Possession:** During enrollment, the vendor must prove they mathematically hold the private key by signing a fresh ML-DSA challenge nonce issued by the Central Authority. 
  * **Root-Signed Replication:** Vendor identities are no longer kept in a local database utility. They are cryptographically packaged into the global `RegistrySnapshot`. Replicas mathematically verify the root signature of the snapshot to establish identity consistency worldwide. If a vendor is not in the signed snapshot, the system "fails closed" and refuses to issue a BAIN.
### Why We Implemented It
To solve the enterprise adoption hurdle. Vendors need to be able to issue a global recall if their software is vulnerable (BAIN Revocation), but Hospitals need mathematical proof that their specific instance data cannot be seen or correlated by the vendor (Storage Isolation & Salts).

### Why Not Other Ways?
* **Why not a single co-signed AIN?** A single AIN would require the vendor to constantly be online and co-sign every single action the hospital takes, destroying reliability and privacy.
* **Why not just revoke deployments manually?** If a vulnerable build is deployed 10,000 times, manually hunting down and revoking 10,000 DAINs is too slow and error-prone. The Cascading Kill Switch handles it mathematically in O(1) time.

---
*Last Updated: Comprehensive Phase 1 to Phase 5 Documentation*
