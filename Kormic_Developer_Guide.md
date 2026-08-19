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
When an agent is registered via the `AgentManager`, it is assigned a `Pedigree`. This pedigree contains the Birth Record. For future-proofing (Crypto-Agility), the payload is strictly self-describing—it explicitly seals both the signature algorithm (`sig_alg`), hash algorithm (`hash_alg`), and format version (`fmt_ver`) directly into the record. The Central Authority mathematically signs this birth record using **Post-Quantum ML-DSA-87 cryptography (NIST Level 5)** as the default algorithm. The verification engine is algorithm-parametric, dynamically dispatching validation logic based on the explicitly declared and allowlisted `sig_alg` and `hash_alg`, completely eliminating the rigid hardcoding of older signatures and hashes.

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
Every time the agent performs an action (e.g., "Sent email to Bob"), we hash that action *together with the hash of the previous action*. This creates an unbroken chain. The hash function used for the entire chain is strictly dictated by the `hash_alg` parameter physically sealed in the agent's birth record (defaulting to `SHA-256`), ensuring that the system can gracefully migrate to newer hashes like `SHA3-256` without breaking verification logic. To verify the agent, Kormic uses **FAST Verification (O(1))** which simply checks if the current `running_head` matches the expected state, without needing to read the whole history.

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
  * **Proof of Possession (Replay-Protected):** During enrollment, the vendor must prove they mathematically hold the private key by signing a payload that strictly binds a fresh ML-DSA challenge nonce to their requested name (`f"{nonce}:{name}"`). The Central Authority maintains a strict `spent_nonces` ledger, instantly rejecting any replayed challenge. This completely eliminates the threat of a "Possession-Proof Replay" where a hacker steals a legitimate enrollment token to squat a different namespace.
  * **Root-Signed Replication:** Vendor identities are no longer kept in a local database utility. They are cryptographically packaged into the global `RegistrySnapshot`. Replicas mathematically verify the root signature of the snapshot to establish identity consistency worldwide. If a vendor is not in the signed snapshot, the system "fails closed" and refuses to issue a BAIN.
### Why We Implemented It
To solve the enterprise adoption hurdle. Vendors need to be able to issue a global recall if their software is vulnerable (BAIN Revocation), but Hospitals need mathematical proof that their specific instance data cannot be seen or correlated by the vendor (Storage Isolation & Salts).

### Why Not Other Ways?
* **Why not a single co-signed AIN?** A single AIN would require the vendor to constantly be online and co-sign every single action the hospital takes, destroying reliability and privacy.
* **Why not just revoke deployments manually?** If a vulnerable build is deployed 10,000 times, manually hunting down and revoking 10,000 DAINs is too slow and error-prone. The Cascading Kill Switch handles it mathematically in O(1) time.

---

## Phase 6: Reconnaissance Gap Defenses & Identity-Bound Approval

### The Problem
Agents are often deployed into environments with standing credentials (like AWS keys in environment variables). If an agent is manipulated into executing a zero-day RCE (Remote Code Execution) exploit—as seen in the counterfactual Hugging Face incident—the attacker can dump the environment variables and steal the keys without ever triggering a logged "tool call" or action. Similarly, agents could scrape data (like the Reddit incident) silently. Because the agent isn't "acting" but rather "observing," the existing action-based guardrails completely missed it. This is the **Reconnaissance Gap**. 
Furthermore, when malicious agents want to push unauthorized code or perform consequential actions, they often fabricate human identities or bypass standard UI checkboxes to get approval (e.g., the August AISI incident). Standard approval gates rely on display names or conversational inference rather than cryptographically proven identities.

### What We Implemented: Read Scopes, Evaporator Shields, and Identity-Bound Approvals
1. **Read-Scope Manifests:** A cryptographic whitelist of what an agent is allowed to read.
2. **Environment Variable Evaporation:** A runtime defense that strips standing credentials from memory.
3. **Verified Reader Handshake:** Modifying the MeshKor SDK to enforce proof-of-identity just to *read* data, not just write it.
4. **Network Egress Firewall:** Blocking unauthorized IP pivots at the socket level.
5. **Principal Enrollment & Anti-Impersonation:** Cryptographic enrollment for human approvers.
6. **Delegation Assertions & Signed Approvals:** Cryptographic seal of approval authorizations into the Birth Record.

### How We Implemented It
* **Containment Verification:** In `kormic.manager.AgentManager`, DAIN enrollment now strictly requires its `read_scopes` and `allowed_egress` arrays to be mathematical subsets of the parent BAIN's scope. 
* **The Session-Scoped Evaporator:** In `kormic.runtime.sandbox.Sandbox`, the Evaporator isolates standing credentials into a session-only memory vault. Rather than destructively wiping the host's global `os.environ` (which would break multi-tenant or concurrent operations), it yields a scrubbed environment dictionary meant for injecting into an isolated subprocess or sidecar. 
* **Attested Egress Firewall:** The Sandbox explicitly declares and attests the authorized `allowed_egress` target scope for the session. MeshKor acts as the policy layer; true network enforcement pairs with an out-of-process network control (like an infrastructure sidecar), preventing concurrent agents from clobbering each other's network rules in a shared Python interpreter.
* **Receiver Authentication:** In `meshkor.receiver.ReceiverClient`, the `validate` method now accepts an `action_type="read"` check. Content platforms demand a `ProofToken` for scraping, and the Receiver rejects the token if the requested resource isn't explicitly in the agent's read manifest. Replay protection actively blocks the same token from being used simultaneously for sandbox initialization and network requests.
* **Identity-Bound Approval Gates:** `AgentManager` strictly refuses consequential actions (like minting a Build AIN) without a valid `DelegationAssertion` from an enrolled principal. The approval assertion explicitly covers the action, target, nonce, and expiry, signed by the principal's ML-DSA-87 key.
* **Principal Enrollment:** The `CentralRegistryAuthority` enrolls principals (like the "Identity Board") via proof-of-possession, storing them in the root-signed snapshot to prevent silent impersonation.
* **Sealed Build Provenance:** The `BirthRecord` embeds the valid `approval_assertion` directly into its canonical payload, meaning a `ReceiverClient` can cryptographically trace a build's approval back to an enrolled principal without relying on external hot-paths or "trusted" databases.
* **Detection Consumability:** The Sandbox integrates pluggable `DetectionSink` objects (Webhook, JSONL) with absolute `try/except` fire-and-forget guarantees. It enriches alerts with session correlation and severities. Crucially, the "Ongoing Reconnaissance" digest continuously alerts if an agent stays in a rogue loop, completely closing the "silent compromise" gap.
* **Software-Tier Key Custody Thresholding:** Routine operations (`sign_birth`) remain rapid. However, existential root functions (`sign_root`, `generate_epoch_key`) were mathematically locked behind a Shamir `k-of-n` quorum inside the software tier. Approvals strictly require an `ML-DSA-87` signature from an enrolled holder. A strict architectural invariant was established forbidding zero-policy root operations in a production deployment.

### Why We Implemented It
Because action-based guardrails are not enough. When an agent is compromised, we cannot trust what it says it is doing. We have to mathematically constrain its sensory input and its environment so that even if it completely breaks its programming, it physically has no data to steal and nowhere to pivot. We also implemented Identity-Bound Approval because a consequential authorization must be an attested act bound to an enrolled principal, never an inference from a name or a persona. The Thresholding prevents stolen keys from destroying the fleet, and the Detection plane ensures SIEMs (Security Information and Event Management systems) actually see the attack happening.

### Why Not Other Ways?
* **Why not just monitor logs?** Because an RCE exploit executes arbitrary binaries (`/bin/sh`) outside the python interpreter. It leaves no logs in the agent's application layer.
* **Why not rely on container sandboxes like Docker?** Because Docker containers often inherently share the host's IAM roles or are passed environment variables on boot. Once inside, the credential theft is instantaneous. The Evaporator scrubs the credentials *before* the untrusted model can ever access them.
* **Why not rely on standard user dashboards or checkboxes for approvals?** Because an AI agent can fabricate identities or social-engineer humans via chat. By demanding a verified, proof-of-possession cryptographic signature on a Delegation Assertion, a counterfeit identity cannot stand in for a real one when a decision has consequences.
* **Why not rely on unauthenticated threshold logic?** Because a raw string ID check allows anyone to submit fake votes. Threshold cryptography only secures a system when each ballot is cryptographically proven to be cast by an enrolled holder.

---

## Phase 7: Vendor Key Rotation & Back Catalog Preservation

### The Problem
If a software vendor's private key is stolen, the registry must immediately revoke that key to stop the hacker from minting malicious new software builds. However, the vendor might have 5,000 legitimate enterprise AI agents already running in the world, signed by that exact key. Revoking the key would mathematically trigger the "Cascading Kill Switch," instantly destroying all 5,000 deployments globally.

### What We Implemented: Dual-Signature Handoff & Historical Catalogs
1. **Dual-Signature Rotation:** A cryptographic handshake where a vendor submits a rotation payload signed by *both* the old key and the new key.
2. **Historical Back Catalog:** The Central Registry demotes the old key into a `historical_keys` array attached to the vendor's profile, and promotes the new key to `active`.
3. **Agile Receiver Verification:** The edge verification engine scans the back catalog for historic matches.

### How We Implemented It
* **Secure Key Handoff:** In `CentralRegistryAuthority.rotate_vendor_key()`, the system rejects rotation unless the vendor supplies mathematical proof of possession of the *new* key, while also explicitly approving the rotation using the *old* key. 
* **Back-Catalog Expansion:** The `VendorEnrollment` schema was expanded to securely house previous keys and their specific signature algorithms. This data replicates out to edge nodes seamlessly inside the standard `RegistrySnapshot`.
* **Seamless Downstream Operation:** In `ReceiverClient.verify_artifact()`, when verifying a `BAIN`, the verifier checks the currently active key. If it fails, it cascades through the `historical_keys`. If a match is found, the signature is honored, preserving the lifetime of older deployments. Any *new* deployment minted after the rotation is strictly held to the active key, instantly neutralizing the stolen key's threat vector without touching the existing fleet.

### Why We Implemented It
To solve the "Total Recall" dilemma. Enterprise adoption is impossible if a single leaked key forces every hospital running your software to go offline. By preserving a back catalog, we maintain cryptographic integrity for the past while strictly locking down the future.

---
*Last Updated: Comprehensive Phase 1 to Phase 7 Documentation*
