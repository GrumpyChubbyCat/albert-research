# Octo Runtime

**Source:** `sources/octo_memory_design_docs.md` (Octo Runtime section)
**Ingested:** 2026-04-17
**Kind:** project / design doc (LamantinAI Reaction-layer assembly)
**Status:** design
**Language:** Rust
**Async runtime:** tokio

## Overview

Octo is an **event-driven runtime for embodied, always-on agents**.

It is inspired by biological systems — specifically the distributed nervous system of an octopus.

## Core Idea

> Not a brain, but a system that lives in time and reacts to the world.

## Architecture

### 1. Connectors ("Tentacles")

Interfaces to the external world.

**Examples:**
- Telegram bot
- Email mailbox
- ONVIF / RTSP cameras
- HTTP webhook listeners
- MQTT-published sensors (Zigbee2MQTT, industrial telemetry, etc.)
- ROS-style ticker nodes

**Responsibilities:**
- Run as long-lived independent processes (autonomous tokio tasks)
- Own connection state and lifecycle: reconnect, retry, supervised restart, graceful shutdown
- Observe / poll / listen on their own schedule (not driven by a central request loop)
- Perform simple local reactions when applicable
- Emit normalized events into the runtime
- For bidirectional connectors: accept actions from the runtime and translate them back to the external world

**Position vs. service frameworks:** each connector is autonomous — it runs continuously and pushes events on its own cadence. Octo supervises a tree of such long-lived tasks. This is the key distinction from Tower/axum/CrabbyQ-style request-driven services, where a handler only fires when a message is delivered to it. Octo's value is exactly this connector lifecycle and supervision layer; once an event has landed inside the runtime, the downstream dispatch may borrow request-driven idioms (and CrabbyQ may live inside specific broker-backed connectors and inside reflex-layer handler stacks).

### 2. Event Runtime (Core)

The central coordination layer.

**Responsibilities:**
- Event routing
- Policy evaluation
- Triggering handlers
- Managing execution lifecycle

### 3. Reflex Layer

Fast, deterministic reactions.

**Characteristics:**
- No LLM required
- Low latency
- Predictable

### 4. Cognition Layer

Advanced reasoning using LLMs or workflows.

**Triggered when:**
- Ambiguity exists
- Complex decisions are needed

### 5. Memory Layer

Stores and retrieves agent experience.

## Execution Flow

1. Event received
2. Normalization
3. Policy check
4. Reflex handling
5. Optional escalation to cognition
6. Action(s) emitted
7. Memory update

## Key Principles

- Everything is an event
- LLM is optional
- Reflexes handle the majority of cases
- Cognition handles uncertainty
- System is always running

## Position in Ecosystem

Octo complements existing tools:

| Layer | Role |
|------|------|
| LLM SDKs | reasoning |
| Graph frameworks | structured workflows |
| MCP/tools | external capabilities |
| Octo | continuous runtime |

## Design Philosophy

- Distributed control (tentacles)
- Central coordination (core)
- Selective intelligence (LLM only when needed)
- Observable behavior

## Target Use Case

Embodied agents that:
- operate continuously
- react to multiple inputs
- exhibit consistent behavior over time

## Summary

Octo is not an agent.
It is the **environment in which an agent exists and behaves**.

## Connections

- **Albert layer:** Reaction — [[albert_system_map]]
- **Memory layer it consumes:** [[memory_model_design]] (model) / [[wiki_as_agent_memory]] (candidate substrate)
- **Adjacent reaction work:** [[rust_agent_frameworks]] — surveys candidate frameworks Octo could integrate with or supersede in the LamantinAI stack.
- **Adjacent OOP framing:** [[alan_kay_and_agent_oop]] — the "tentacles + core" decomposition is canonical Actor Model / messaging in action.
- **Program:** [[llm_research_roadmap]]
- **Adjacent service framework:** CrabbyQ — Tower-based «Axum for message brokers» (NATS / Redis / MQTT). Natural fit *inside* broker-backed connectors and inside the reflex-layer handler stack. Does not replace Octo's connector lifecycle / supervision layer — those are orthogonal concerns Tower does not cover. See [[llm_research_roadmap]] § Infrastructure Projects.
- **Adjacent streaming substrate:** [[fluxion]] — streaming-oriented substrate for media-aware pipelines, candidate for high-throughput perception connectors.
- **Detailed protocol design:** [[octo_architecture]] — connector trait, envelope schema, lifecycle, dispatch, backpressure, supervision tree.
