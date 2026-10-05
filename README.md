# agent-session-bridge

The session bridge connects agent sessions to the outside world. Agent-to-human and agent-to-agent traffic flows through it.

- **Input producers** carry messages into a session: Matrix, XMPP, a native app, Telegram, and later others.
- **Output consumers** carry a session's output out: summaries, decision requests, or the full raw stream for debugging.
- **A trust gate** sits in front of every input. A message reaches the model only after it passes, in order:
  1. its signature verifies;
  2. the signer is a pinned identity;
  3. it is in scope of a grant or mandate;
  4. it is fresh, and replays are rejected;
  5. deterministic checks pass.

  The operator's signed messages become authoritative input. Everything else is framed as untrusted data with its provenance.
- **A verification tool** lets the model check, by itself, the signature, grants and trust level of a message it reads.

Status: **design**. The bridge is a separate module from the session host. It is built on the trust primitives of `relux-works/curator-trust` and the mandates model.

Related:
- `relux-works/agent-session-host`: supervisor, holder, harness adapters, parking and restart.
- task-board: switched onto these modules through a compatibility facade.
