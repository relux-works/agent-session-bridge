# agent-session-bridge

The session bridge connects agent sessions to the outside world. Agent-to-human and agent-to-agent traffic flows through it, in both directions, through one trust gate.

- **Input producers** carry messages into a session: Matrix, XMPP, a native app, Telegram, and later others. Producers never hold a terminal or a native harness endpoint.
- **Output consumers** carry a session's output out: state, decision requests, summaries, or the full raw stream for debugging. Each consumer holds a separate read scope.
- **A trust gate** sits in front of every managed input. A message reaches the model only after it passes, in order:
  1. its signature verifies against exact frozen bytes;
  2. the signer is a pinned identity;
  3. it is in scope of a grant or mandate;
  4. it is fresh, and replays are rejected;
  5. deterministic checks pass;
  6. an optional guard verdict allows it.

  The operator's signed messages become authoritative input. Everything else is framed as untrusted data with its provenance.
- **A verification tool** lets the model check, by itself, the signature, grants and trust level of a message it reads. It returns evidence, never the unverified body.

Status: **architecture review**. This repository holds the design under review, not a working bridge. No implementation code is accepted until the review closes. See [docs/architecture.md](docs/architecture.md) and [docs/threat-model.md](docs/threat-model.md).

## What it is NOT

- It is not the session host. Supervision, execution custody, harness adapters, parking and restart live in `relux-works/agent-session-host`.
- It is not a trust library. Signatures, identity, grants and mandates come from `relux-works/curator-trust` and the mandates model; the bridge only applies them at its boundary.
- It is not a chat server, a relay, or a general message bus. It admits envelopes into sessions and projects session state out, nothing more.
- It is not a sandbox. Gating managed bridge input does not certify native tool output, files, shell results, hooks, resumed history, or anything a native harness tool returns. The exact scope of the guarantee is in [docs/architecture.md](docs/architecture.md).

## Layout

```
.
├── README.md                  this file
├── docs/
│   ├── architecture.md        the bridge design under review
│   └── threat-model.md        assets, adversaries, mitigations, out of scope
└── diagrams/                  architecture diagrams as code (PlantUML sources + renders)
```

Related:

- `relux-works/agent-session-host`: supervisor, holder, harness adapters, parking and restart.
- task-board: switched onto these modules through a compatibility facade.
