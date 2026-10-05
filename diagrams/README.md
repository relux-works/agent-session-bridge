# Diagrams

Architecture diagrams as code for the session-bridge review package ([`../docs/architecture.md`](../docs/architecture.md), [`../docs/threat-model.md`](../docs/threat-model.md)).

## Structure

```
diagrams/
├── README.md                    this file: tools, commands, conventions
├── plantuml/
│   ├── _includes/theme.puml     shared look, the same palette as agent-session-host
│   ├── component/               structure: what the bridge is made of
│   └── sequence/                runtime flows
└── artefacts/plantuml/          rendered SVG and PNG, committed
```

## Diagrams

| Diagram | Kind | Source | Renders |
| --- | --- | --- | --- |
| Session bridge: producers and consumers meet the session host through the trust gate | PlantUML component | [bridge-components.puml](plantuml/component/bridge-components.puml) | [SVG](artefacts/plantuml/bridge-components.svg), [PNG](artefacts/plantuml/bridge-components.png) |
| Inbound signed message through the ingress trust gate, with refusal paths | PlantUML sequence | [inbound-signed-message.puml](plantuml/sequence/inbound-signed-message.puml) | [SVG](artefacts/plantuml/inbound-signed-message.svg), [PNG](artefacts/plantuml/inbound-signed-message.png) |
| Outbound projection with the guard and receipts | PlantUML sequence | [outbound-projection.puml](plantuml/sequence/outbound-projection.puml) | [SVG](artefacts/plantuml/outbound-projection.svg), [PNG](artefacts/plantuml/outbound-projection.png) |

## Tools

| Tool | Used for | Install |
| --- | --- | --- |
| A Java runtime | runs PlantUML | `brew install openjdk` |
| PlantUML (`plantuml.jar`) | turns each `.puml` source into SVG and PNG | `brew install plantuml`, or the JAR: `curl -L -o plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar` |

No Graphviz needed: sequence diagrams use PlantUML's built-in engine, and the component diagram pins `!pragma layout smetana`.

## Render

From the repository root:

```bash
java -jar plantuml.jar -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
java -DPLANTUML_LIMIT_SIZE=8192 -jar plantuml.jar -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
```

`plantuml.jar` in these commands is the plain JAR filename from the [official PlantUML release](https://github.com/plantuml/plantuml/releases/latest), placed in the repository root where the commands run. If you keep the JAR elsewhere, set `PLANTUML_JAR` to its file location and use `java -jar "$PLANTUML_JAR" …` with the same arguments:

```bash
java -jar "$PLANTUML_JAR" -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
java -DPLANTUML_LIMIT_SIZE=8192 -jar "$PLANTUML_JAR" -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
```

With the `plantuml` executable (`brew install plantuml`), use the same render arguments:

```bash
plantuml -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
PLANTUML_LIMIT_SIZE=8192 plantuml -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/sequence/*.puml
```

The output directory is absolute because PlantUML resolves a relative `-o` against the folder of each source file.

The PNG commands raise PlantUML's image size limit from its default of 4096 pixels to 8192. The inbound sequence diagram is taller than 4096 pixels, and with the default limit its PNG is cut off at the bottom. The `plantuml` executable reads the same limit from the `PLANTUML_LIMIT_SIZE` environment variable.

## Conventions

- One purpose per diagram, with a title and a caption naming its sources.
- Technical terms only, as in the reviewed documents.
- Non-sequence diagrams pin `!pragma layout smetana` so they render without Graphviz.
- Colour says what a thing is: blue for node services, purple for agents and harness processes, amber for boards and committed records, grey for private state, green for people, teal for carriers.
- A change to a source is committed together with its re-rendered SVG and PNG.
