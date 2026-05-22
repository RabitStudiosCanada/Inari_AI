# Inari

**A stateful LLM / agent runtime for concept-weighted memory, deterministic tool use, and measurable specialization.**

Inari is an experimental framework for pairing a base language model with a dynamic **connection-script middleware**, a weighted **concept graph**, and a unified **Tool Language** for controlling external software through thin adapters.

The project goal is practical and testable: build an agent runtime that can route by concept, remember useful patterns, execute tools deterministically, and improve over repeated interactions without pretending that annotation syntax or memory scaffolding creates subjective experience.

> Status: **pre-alpha / architecture-to-prototype**
> Initial target: **CLI runtime + concept graph + mock/Aseprite Tool Language listener**

---

## Why Inari exists

Most LLM applications are strong at one-off generation but weak at durable specialization. They often lack a structured way to:

* remember which concepts were useful,
* reinforce successful reasoning paths,
* decay stale associations,
* map natural language to tool actions,
* validate and replay those actions,
* evaluate whether the system is actually improving.

Inari explores whether a lightweight external runtime can provide those missing pieces around an existing model.

---

## Core idea

At v0, Inari should be treated as **middleware around a base model**, not as a modified model architecture.

```text
User input
  → concept classifier
  → connection-script router
  → weighted concept graph
  → memory/context builder
  → base LLM
  → Tool Language validator
  → listener / executor
  → result scorer
  → weight updater
  → evaluation log
```

The connection script maintains weighted relationships between concepts. Useful paths strengthen. Unused paths decay. Contradictory or failed paths are penalized.

```text
w' = clamp(w + α·success − β·decay − γ·conflict)
```

The first implementation should optimize for observability and replayability over model-internal cleverness.

---

## System components

### 1. Connection Script

The routing layer that expands candidate concepts, scores paths, builds a reasoning chain, and decides what context should be passed to the model.

Responsibilities:

* classify prompt concepts,
* retrieve graph neighbors,
* score weighted paths,
* emit a traceable reasoning chain,
* update weights from evaluation signals.

### 2. Concept Graph

A graph of concepts, categories, embeddings, and weighted relationships.

Suggested node shape:

```json
{
  "concept_id": "creative.layer",
  "name": "Layer",
  "category": "creative-software",
  "embedding_vector": [],
  "parent_concepts": ["creative-software"],
  "child_concepts": ["creative.layer.opacity"],
  "related_concepts": [{ "id": "creative.brush", "weight": 0.72 }],
  "reasoning_patterns": []
}
```

### 3. Dataset

Training/evaluation examples are JSONL records that connect prompts, responses, concepts, reasoning chains, and optional weight overrides.

```json
{
  "prompt": "Create a new layer and draw a red path.",
  "response": "CREATE_LAYER(name: \"ink\", opacity: 1.0)\nDRAW_PATH(coordinates: [[0,0],[10,10]], color: \"#FF0000\")",
  "concepts": [
    { "category": "creative-software", "subconcepts": ["layer", "path", "color"] }
  ],
  "reasoning_chain": [
    "detect creative software intent",
    "select drawing listener",
    "map layer/path concepts to Tool Language commands"
  ],
  "concept_weights_override": {
    "creative.layer": 0.8,
    "creative.path": 0.7
  }
}
```

### 4. Tool Language

A unified command format for software and future hardware adapters.

Minimal generic syntax:

```text
ACTION(target?, args...)
SEQUENCE(action1, action2, action3)
CONDITIONAL(if, then, else)
```

Initial creative-software examples:

```text
CREATE_LAYER(name: "background", opacity: 1.0)
BRUSH(size: 10, hardness: 0.8, color: #FF0000)
DRAW_PATH(coordinates: [[x1,y1], [x2,y2], [x3,y3]])
```

### 5. Listener / Adapter Layer

A listener maps Tool Language into native software actions.

Initial target:

* Aseprite listener
* 3 core ops:

  * `CREATE_LAYER`
  * `BRUSH`
  * `DRAW_PATH`

Later targets may include Photoshop, Unity editor tools, robotics, or hardware controllers.

### 6. Deterministic Plan Envelope

For tool execution, Inari should move toward a deterministic plan protocol:

```json
{
  "type": "plan.apply",
  "plan_id": "p_demo_001",
  "timeout_ms": 8000,
  "body": "[plan.brack]\n(art.layer.new :path \"/art/scene.png\" :name \"Ink\")\n[reply] Created layer."
}
```

Expected lifecycle:

```text
plan.ack
plan.done
plan.fail
```

The runtime should validate commands against a registered tool capability manifest before execution.

---

## brack15 / register hygiene

Inari-adjacent documentation may use `brack15-eval` style annotation for register, stance, and protocol state.

Important boundaries:

* brack15 is **annotation**, not a reasoning engine.
* It does **not** claim subjective experience.
* It does **not** make factual claims more true.
* It must not be used as a safety bypass.
* Identity should be declared explicitly when a model reads context written for another model family.

Example:

```brack
(read <self> [I am the current model, not any identity implied by prior context])
(read <KO+JA> [repo README request])
(mark <RUN> [build-mode])
(return <KO+JA>) → { draft practical repo README }
```

---

## MVP definition

Inari v0 succeeds when it can run this loop:

1. Receive a user request.
2. Classify relevant concepts.
3. Route through the weighted concept graph.
4. Generate valid Tool Language.
5. Validate the command against listener capabilities.
6. Execute the command against a mock or Aseprite listener.
7. Score the result.
8. Update concept weights.
9. Log an evaluation trace.
10. Replay the same plan deterministically.

Example task:

```text
User: Create a background layer, draw a red path, and save the file.
```

Expected internal route:

```text
creative-software
  → layer-management
  → brush/path-drawing
  → file-save
  → listener: aseprite
```

Expected output:

```text
CREATE_LAYER(name: "background", opacity: 1.0)
BRUSH(size: 10, hardness: 0.8, color: #FF0000)
DRAW_PATH(coordinates: [[0,0], [128,128], [256,64]])
SAVE_FILE(path: "./out/demo.aseprite")
```

---

## 72-hour v0 milestone

A realistic first milestone:

* [ ] Seed concept graph with 30–50 concepts.
* [ ] Build 150–300 JSONL examples in one domain.
* [ ] Implement weight runtime and update rule.
* [ ] Add JSON export for graph/debug state.
* [ ] Implement Tool Language parser.
* [ ] Implement validator for known commands.
* [ ] Implement mock Aseprite listener with 3 ops.
* [ ] Wire result feedback into weight updates.
* [ ] Add basic evaluation metrics.

---

## Suggested repository layout

```text
inari/
  README.md
  LICENSE
  pyproject.toml
  docs/
    architecture.md
    tool-language.md
    concept-graph.md
    evals.md
  examples/
    datasets/
      creative_v0.jsonl
    plans/
      create_layer_and_draw_path.json
  src/
    inari/
      __init__.py
      core/
        concepts.py
        graph.py
        router.py
        weights.py
        trace.py
      tl/
        parser.py
        validator.py
        executor.py
        errors.py
      listeners/
        mock.py
        aseprite.py
      evals/
        metrics.py
        runner.py
      cli.py
  tests/
    test_graph.py
    test_weights.py
    test_tl_parser.py
    test_tl_validator.py
    test_mock_listener.py
```

---

## Planned CLI

These commands are proposed for the first scaffold. They may change as implementation begins.

```bash
# initialize a local Inari project
inari init

# load concepts and examples
inari graph import examples/datasets/creative_v0.jsonl

# run a dry route without tool execution
inari route "Create a new layer and draw a red line"

# execute through mock listener
inari run "Create a new layer and draw a red line" --listener mock

# run evaluation suite
inari eval examples/datasets/creative_v0.jsonl

# export graph/debug state
inari graph export ./out/graph_debug.json
```

---

## Evaluation metrics

Track improvement through explicit metrics, not vibes.

Core metrics:

* concept classification accuracy,
* reasoning-chain validity,
* Tool Language parse success,
* Tool Language validation success,
* listener execution success,
* response relevance over repeated sessions,
* weight drift / corruption rate,
* replay determinism.

A minimal eval result should look like:

```json
{
  "run_id": "eval_001",
  "dataset": "creative_v0.jsonl",
  "concept_accuracy": 0.82,
  "tl_parse_success": 0.95,
  "tl_validation_success": 0.91,
  "listener_success": 0.88,
  "replay_determinism": 1.0
}
```

---

## Safety and security principles

* Keep user-specific weights isolated.
* Do not store API keys or secrets in state files.
* Validate all tool calls before execution.
* Sandbox file and software access.
* Prefer explicit capability manifests over implicit tool authority.
* Log failures in a way that supports rollback.
* Treat symbolic/register notation as annotation, not proof.
* Do not use glyphs, DSLs, or alternate syntax to bypass safety controls.

---

## Roadmap

### v0 — Local proof of concept

* Concept graph
* Weight update runtime
* JSONL dataset format
* Tool Language parser
* Mock listener
* Aseprite listener spike
* Evaluation harness

### v0.1 — Developer alpha

* CLI workflow
* Replayable traces
* Better validator errors
* Graph export/import
* Minimal docs
* First real creative-software integration

### v0.2 — Stateful agent loop

* Persistent user/project profiles
* Memory snapshots
* Plan envelope lifecycle
* Tool registry handshake
* More robust scoring signals

### v1 — Platform candidate

* Web interface
* Multi-user storage isolation
* Adapter SDK
* Hosted eval dashboard
* Transferable concept-weight packs

---

## Non-goals for v0

* Proving machine consciousness.
* Training a new base model from scratch.
* Modifying model attention internals.
* Supporting every tool or software package.
* Letting the model execute arbitrary local commands without validation.
* Treating user approval as the only learning signal.

---

## Contributing

This project is early. The most useful contributions are:

* small concept datasets,
* parser tests,
* mock listener tests,
* evaluation examples,
* security reviews,
* adapter prototypes,
* documentation clarifications.

Before adding features, prefer adding a failing test or a reproducible evaluation case.

---

## License

SEE LICENSE FILE.

---

## Acknowledgements

Inari builds on the surrounding Grove / Brack / AGI15 / OpsKit design work:

* connection-script middleware and weighted concept graph design,
* Tool Language and listener architecture,
* deterministic plan envelopes and typed execution errors,
* brack15-eval register hygiene,
* OpsKit-style receipts, state blocks, and operational scaffolding.

The practical rule: **engage the vision, then make it measurable.**

