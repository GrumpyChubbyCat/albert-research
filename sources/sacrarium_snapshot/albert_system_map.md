> Я знаю как мы соберём Альберта:
>* Perception   → what is happening
>* Experience   → what it means now
>* Memory       → what is stored
>* Reflection   → what is learned
>* Reaction     → what is done
---

# Albert System Map

`Albert` is the long-term integrative goal of the LamantinAI AI program.

It should not be understood as a single model trained all at once, but as a future assembly of several research and infrastructure directions that mature over time.

## Connections

- **Upstream goal:** [[llm_research_roadmap]] — Albert is defined there as the long-term integrative goal.
- **Upstream values:** [[lamantin_manifesto]] — Engineering Ethics shapes what each functional layer is allowed to do.
- **Deepest foundation:** [[living_universe]].
- **Bridge:** [[values_mapping]] — from worldview to engineering constraints.
- **Downstream tracks:** [[streaming_scene_model_project_context]] (perception), [[narrative_alignment]] (experience), [[embodied_agent_memory]] (memory), [[connectome_reasoning]] (reflection), [[rust_agent_frameworks]] (reaction).

## Functional Decomposition

### 1. Perception

Question:

What is happening?

This direction includes:

- streaming visual perception
- scene analysis in real time
- entity persistence across time
- structured metadata extraction
- incident and event perception

Current LamantinAI focus:

- streaming scene model
- real-time scene state tracking
- metadata stream generation

### 2. Experience

Question:

What does it mean now?

This is the layer that interprets perceived state in the context of current goals, prompts, ontology, task framing, or local situation.

Likely topics:

- scene-state interpretation
- task-conditioned relevance
- prompt-conditioned extraction
- context-sensitive event meaning

### 3. Memory

Question:

What is stored?

This is not limited to hidden state inside perception models.
It is a broader research direction around memory systems for agents and AI.

Examples:

- episodic memory
- semantic memory
- retrieval memory
- compression of experience
- adaptive memory layers
- systems inspired by projects like `Titans` or `Mem0`

### 4. Reflection

Question:

What is learned?

This direction covers higher-order reasoning and revision over accumulated state and memory.

Examples:

- connectome-like MoE
- reflective inference
- uncertainty-aware reasoning
- hypothesis testing
- post-perception interpretation

### 5. Reaction

Question:

What is done?

This is the action and orchestration layer.
For LamantinAI, it is likely to be built through Rust-based agent runtimes and their integration into the wider ecosystem.

Examples:

- agent frameworks on Rust
- tool use orchestration
- workflow execution
- broker-driven reaction logic
- integration with messaging and service infrastructure

## Infrastructure Connection

Albert is not only an AI research project.
It depends on active infrastructure work inside the LamantinAI ecosystem.

- [[fluxion|Fluxion]] provides a neuromorphic and streaming-oriented substrate for intelligent media analysis.
- `CrabbyQ` provides a Tower-based service framework for message-driven systems with NATS, Redis, and MQTT backends; natural foundation for broker-backed connectors and reflex-handler stacks in the future Octo runtime.

These projects are not the same thing as the AI research directions, but they are part of the eventual assembly path.
