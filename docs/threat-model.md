# Session bridge threat model

Status: **architecture review**. This is a short threat model of the bridge as designed in [architecture.md](architecture.md) — proposed mitigations, not shipped controls. It covers the bridge boundary only: producers, consumers, the trust gate, the verifier, keys and audit. The asset statements below hold within that boundary; they are not guarantees about native inputs or egress outside it.

## Security goals

Three design requirements; each applies through the accepted mode boundaries in [architecture.md](architecture.md) §4–§4.1, not as universal mediation. The mitigations below exist to keep them:

- **Cryptography and verification of session input.** Every managed input into a session is cryptographically verified before it can act: in mode T all managed bridge input crosses the gate; in mode S every model input is mediated or refused. Native and vendor paths outside the declared mode stay outside the guarantee. Answers: forged or unauthorized input, replay and reorder, instruction injection inside allowed content, compromised-key signing.
- **No process can simply write into a session: a state report is an observation channel, never a command path into the session.** Managed writes cross the gate; agent state reports update display state and reported-status waits for one execution and never enter the gate as commands. A compatibility (mode T + RC) session may still receive vendor-origin RC input outside per-message assurance (§4.1); a strict session without evidenced RC exclusion is refused or parked. Answers: local-attacker writes, vendor side channels, malicious-app input, verifier and receipt misuse.
- **Scale to large swarms.** Admission, queues, and audit stay bounded under flood and fleet growth. Answers: resource exhaustion and bot loops, carrier duplication storms.

## Assets

- **Session integrity.** Only admitted commands reach the model as instructions; everything else arrives as attributed data or not at all.
- **Operator authority.** The operator's signature and presence cannot be forged, replayed, or borrowed by another principal.
- **Confidentiality of session content.** Transcripts, decisions and artifacts leave only through audience-bound projections on appropriate bindings (E2E where promised).
- **Audit integrity.** The hash-chained record of admissions, deliveries and rejections detects tampering relative to a retained head.
- **Keys and grants.** Operator, host, agent and encryption keys; grant/mandate scopes; revocation data.
- **Bounded local resources.** Intake, replay state, quarantine, reviews, subscriptions, outbox and blob storage stay bounded under flood; an emergency-control route survives saturation.

## Adversaries

- **Malicious peer.** A valid carrier participant (or a compromised peer account) sending crafted envelopes, including signed ones, to inject instructions or exfiltrate data.
- **Carrier/relay attacker.** A curious or hostile relay, a network attacker, or a weaker-mode binding: reads metadata or plaintext, replays, reorders, delays or drops envelopes.
- **Malicious app or device.** A compromised consumer endpoint, a stolen push token, a cloned session view.
- **Local attacker.** A same-user process or a reader of on-disk state: replay-store tampering, audit truncation, key-file theft. (Stronger isolation claims need evidenced tool/process boundaries; see below.)
- **Compromised key holder.** A stolen or copied operator, agent or host key used to sign new input or new keys.
- **Privileged guard or classifier operator.** A guard adapter with tools, credentials or side effects, or a classifier provider outside the authorized data audience, reading private content or acting on injected requests before verdicting.
- **Unauthorized verifier caller.** A model or process attached to one session requesting another session's envelope references or evidence.
- **Resource-exhaustion attacker.** A principal (valid or not) flooding intake, unique-ID claims, reviews or outbox growth; two bridged endpoints amplifying each other in a bot loop.

## Entry points

| Entry point | What crosses it |
|---|---|
| Producer envelope intake (Matrix, XMPP, app, Telegram, later) | Remote bytes → immutable original envelopes with raw bytes retained |
| Local API and model mailbox reads | Local bytes → the same gate |
| Bridge profile binding (`curator run` profile) | Session↔channel mapping, classes, grants, audience |
| Pairing flow | New channel/device enrollment |
| Consumer subscriptions | Projection cursors, delivery, revocation-closure |
| Verifier (MCP + CLI) | Caller-bound envelope references in, authorized evidence out — read-only |
| Guard execution boundary (when enabled) | Fixed-instruction, tool-less, bounded classifier calls; verdicts; exact-digest release approvals |
| Audit store and replay bindings | Durable state the gate depends on |
| Emergency-control route | Independently authorized stop/fence actions only — admits nothing |
| Agent state-report channel (vendor reports) | Execution-local telemetry → display state and reported-status waits only; observation-only, with no path to dispatch, authority, or grants |

## Mitigations (by threat)

- **Forged or unauthorized input.** Exact-byte signature verification, pinned identity, grant/mandate scope intersection with proof of possession; signature and grant failures never reach any model (§3). Carrier login never equals application identity.
- **Replay.** Durable `(trust-domain, signer, message-id)` → envelope-hash binding with in-flight/completed/outcome-unknown states; action-specific expiry and skew policy; completed duplicates return the cached outcome, unresolved duplicates reconcile, conflicting bytes conflict (§3 step 4, §3.3).
- **Reorder (separate from replay).** Per-sender sequences scoped to (recipient session, sender, ordered class); compatible bindings preserve sender/recipient sequence checks with a bounded gap policy; the new namespace documents its replacement order contract; unordered classes are explicit in the bridge profile; concurrent same-class claims serialize (§3.2).
- **Replay recovery abuse.** Per-class retention horizon / durable anti-replay floor; tombstones survive to the horizon; backwards clock, old-snapshot restore or lost high-water state parks intake; bindings follow the stable principal across key epochs; rewraps stay derivatives of the origin ID; conflicts and cached outcomes survive migration and recovery (§3.3).
- **Instruction injection inside allowed content.** Authoritative input requires an enrolled operator signature plus presence for critical acts; everything else is framed as untrusted data with provenance (§3.1). Deterministic checks run before any classifier; deterministic denies cannot be approved (§3 step 5).
- **Guard bypass, confusion or privilege.** Fixed instructions, one admitted immutable model identity with response/model matching, bounded strict verdicts, fixed execution limits — refusal on every error or mismatch, no silent bypass (§3.4). The adapter exposes no tools, session credentials, signing authority, retrieval or side effects; its own credential stays in its transport adapter; configuration changes are owner-controlled and audited.
- **Unauthorized classifier disclosure.** The classifier endpoint is authorized as a data recipient before every guard call in either direction, under audience/privacy policy including retention; content denied to it is never submitted — the outcome is refusal/park or a separately authorized local guard (§3.4, §5).
- **Guard-refusal fallthrough.** Deny, refused/expired review and guard failure terminate the pipeline: commit and delivery live only inside the permitted branch; pending review stays pending, never accepted (§3 step 6–7, §5; both sequence diagrams).
- **Cross-carrier duplication and reflection loops.** Application origin-ID dedup across carriers with the (principal, origin ID, binding revision, derivative) binding; mirrored output recognized and refused as input; distinct origin/command/carrier IDs (§2.3).
- **Over-broad reads.** Separate consumer scopes for state, decisions, transcript and raw terminal; projections by default, raw behind distinct permissions; audience/grant rechecks at delivery with cursor reset on audience/filter change; revoked subscriptions closed (§5).
- **Secret leakage outbound.** Deterministic DLP filtering before any classifier; redaction as separately signed derivatives; outbound guard over the proposal content with owner release outside the agent's reach; approved proposal identity covers content, audience, direction and policy — any change voids the release (§5). Filters are best effort.
- **Key theft and misuse.** Separate key classes; agent keys never hold the root seed; signer brokers check caller/class/target/digest/approval and never sign arbitrary model bytes; compromised-key revocation cascades down descendants; recovery needs an independently authorized anchor with a visible trust change; rotation preserves replay bindings on the stable principal (§7, §3.3).
- **Rogue enrollment.** Pairing needs expiry, nonce, proof of possession and human confirmation; unknown pins and key changes go to operator review (§2.3, §7).
- **Audit tampering.** Domain-separated hash chain with signed, optionally remotely witnessed checkpoints; verification on reopen with recorded recovery gaps (§8).
- **Verifier misuse.** The verifier returns evidence, never the unverified body; inspection consumes no delivery and issues no grant; results are evidence at a named policy revision, not reusable tickets (§6).
- **Cross-session verifier reads.** Requests bind to an authenticated caller/session; reference resolution and returned evidence are authorized against that caller; references resolve only within an approved store/namespace or documented capability; denial and nonexistence are indistinguishable where the distinction would leak; metadata redaction; no invalid-body or error echoes (§6).
- **Receipt forgery or overstatement.** One issuer per stage — local acceptance, transport enqueue/wake, endpoint receipt/fetch, host dispatch/application evidence, rejection, unknown; signatures bind digest, recipient, status, policy and audit reference; only evidenced stages reported; push acceptance never implies data fetch; forged/mismatched receipts rejected; crash-to-receipt gap reports `unknown` (§8).
- **Local resource exhaustion and bot loops.** Early global/per-principal admission budgets before expensive work; bounded parser work, quarantine, replay state, reviews, subscriptions, outbox/blob storage and classifier concurrency/spend; backpressure with recorded refusals; replay safety preserved during reclaim; explicit evidence accounting when auditing fails; a bounded, independently authorized emergency-control route that admits nothing (§3 step 0, §3.5).
- **Vendor side channels.** Claude Remote Control is classified as an operator compatibility channel outside per-message assurance; strict sessions refuse every activation path (initial flags, in-session enablement, reconnect/resume, attachments, permission races) or park; compatibility outcomes and vendor/unknown provenance are explicit per path (§4.1).
- **Vendor state reports.** Classified observation-only: they update display state and reported-status waits for one execution and never enter the gate as commands; no report path reaches dispatch, authority, or grants.

## Out of scope

- **Native harness behavior.** In transport-gated mode, tool output, files, shell results, hooks, resumed history, subagent returns and MCP content are outside the guarantee by design (§4). Strict mode mediates them only where evidenced isolation exists.
- **Endpoint compromise beyond the gate.** A fully compromised operator device, a coerced presence approval, or a stolen unrevoked presence key defeats any input gate.
- **Carrier availability and traffic analysis.** Carriers and relays can observe metadata, delay or drop traffic; the design answers with ciphertext-only relays, expiry, receipts and explicit gaps — not with metadata hiding or guaranteed delivery. Bridge-side resource safety (§3.5) stays in scope: local exhaustion must not become silent admission or lost control.
- **Classifier completeness.** The guard screens for known-bad patterns; no verdict schema proves a classifier catches every novel injection. Deterministic gates carry the guarantee.
- **Whole-history replacement.** The hash chain exposes alteration relative to a retained head; it cannot stop a disk owner from replacing an unwitnessed history wholesale.
- **Instant remote revocation while disconnected.** Offline revocation bounds are per-deployment; no design promises immediate effect without connectivity.
- **Forward secrecy.** Static keys, E2E branding and optional derivation imply no historical confidentiality or forward secrecy.
