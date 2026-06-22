# TWAIN

**Natural language → validated simulation → plain-language answer.** A research co-pilot that turns a plain-English question into a real simulation on a trusted scientific engine — with full provenance — and then explains the result. The AI routes and interprets; it never invents the math.

![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic-v2-E92063?logo=pydantic&logoColor=white)
![Typer](https://img.shields.io/badge/CLI-Typer-1F8B4C)
![Anthropic](https://img.shields.io/badge/LLM-Anthropic%20Claude-D97757?logo=anthropic&logoColor=white)
![OpenRouter](https://img.shields.io/badge/LLM-OpenRouter-6E56CF)
![SciPy](https://img.shields.io/badge/SciPy-8CAAE6?logo=scipy&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-013243?logo=numpy&logoColor=white)
![SLURM](https://img.shields.io/badge/HPC-SLURM%20%2F%20AiiDA-2C3E50)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)

---

## What it does

Scientific simulation is powerful but gated behind expertise: knowing *which* engine fits a question, how to express it in that engine's parameter schema, whether the request even sits inside the regime the engine was validated for, and how to read the output. That friction keeps simulation out of reach for most researchers and slows down the ones who can use it.

TWAIN closes that gap. You ask a question in plain English — *"How does a damped oscillator settle for these initial conditions?"* or *"What's a reasonable force field for liquid water at 300 K?"* — and TWAIN:

- **picks the right engine** from a large matrix of physics, chemistry, and biology backends,
- **binds your intent to that engine's exact parameter schema**, asking for clarification only when something is genuinely ambiguous,
- **refuses to guess** when a request is out of scope or under-specified (it abstains rather than fabricating a run),
- **runs the real simulation** locally or on an HPC cluster, capturing a replayable provenance manifest,
- **explains the result in plain language**, grounded strictly in the numbers the engine produced — and, where a certified reference case exists, compares the output against published values.

The guiding principle is that the language model is a router and an interpreter, never the source of the physics. Every number a user sees comes from a real solver, not the LLM.

## How it works

A single orchestration entry point drives the same pipeline for both the CLI and the REST API, so behavior is identical everywhere:

```
  Plain-English question
          │
          ▼
  ┌─────────────────┐   typed, validated Intent (Pydantic) via
  │  Intent parsing │   Instructor-guided structured extraction
  └─────────────────┘   • simulation_explicit  • property_driven
          │
          ▼
  ┌─────────────────┐   engine registry + property-driven pathways;
  │     Routing     │   certified engines preferred over experimental
  └─────────────────┘
          │
          ▼
  ┌─────────────────┐   re-extract free-form intent into the chosen
  │ Parameter bind  │   engine's own schema (skipped when already valid)
  └─────────────────┘
          │
          ▼
  ┌─────────────────┐   layered validation stack:
  │  Validate /     │   schema → physics → regime envelope →
  │  clarify /      │   pre-flight dry-run; clarify or abstain on
  │  abstain        │   ambiguous or out-of-scope requests
  └─────────────────┘
          │
          ▼
  ┌─────────────────┐   local runner or thin-SLURM / AiiDA runner —
  │    Execute      │   both write the same provenance manifest
  └─────────────────┘
          │
          ▼
  ┌─────────────────┐   plain-language findings, caveats, next steps,
  │   Interpret     │   grounded only in the engine's output; optional
  └─────────────────┘   comparison to certified reference values
          │
          ▼
   Typed PipelineResult  →  CLI · REST API · web UI
```

**Components**

- **Intent layer** — converts free-form questions into a typed, validated `Intent` using structured LLM extraction, with automatic retry on schema-validation failure. Handles both explicit "run engine X" requests and open-ended, property-driven questions that propose candidate pathways for the user to choose from.
- **Engine registry** — a single canonical source of truth for every adapter, shared by the CLI, API, and eval harness so the live engine matrix is identical across all entry points. Missing optional backends are recorded as tolerated skips; a mis-named adapter is a hard, surfaced error — failures are never swallowed silently.
- **Validation stack** — a multi-layer gate (schema → physics → regime envelope → pre-flight dry-run → post-run checks). Engines can declare the parameter envelope they were tested in, so requests outside that envelope are flagged with lowered confidence rather than presented as trustworthy.
- **Execution** — a local runner and an HPC runner (thin SLURM submission, with optional AiiDA provenance graphs) emit an identical run manifest, so provenance and one-command replay work the same whether a job ran on a laptop or a cluster.
- **Interpretation** — produces structured findings, caveats, and suggested next steps strictly grounded in the engine's output, plus a validated/known-value comparison when the run matches a certified reference case.
- **Surfaces** — a Typer-based CLI, a FastAPI REST service (with auto-generated docs), and a lightweight static web UI for asking questions and reading results.

## Highlights

- **The LLM never invents the math.** Every quantity comes from a real solver; the model only routes the question and narrates the result.
- **Knows when to say "I don't know."** A dedicated clarify/abstain path declines under-specified or out-of-scope questions instead of producing a confident-looking but fake run.
- **Two-pass parameter binding.** A first pass chooses the engine; a second pass re-extracts the request into that engine's exact Pydantic schema — and is skipped entirely when the first pass already satisfies it, keeping well-formed requests fast.
- **Broad engine matrix.** Roughly thirty adapters spanning ODE/dynamical systems, agent-based and discrete-event modeling, stochastic and Bayesian methods, N-body/astrophysics, molecular dynamics, DFT and electronic structure, systems biology, epidemiology, FEM/CFD/electromagnetics, and more — each behind a uniform adapter protocol.
- **Certified vs. experimental engines.** Certified engines reproduce published reference results within a stated tolerance; the router prefers them, and results carry a validation badge when they match a certified case.
- **Reproducible by construction.** Each run writes a provenance manifest and a replay command, with the same manifest format across local and HPC execution.
- **Provider-agnostic LLM layer.** Auto-selects between Anthropic and OpenRouter based on available credentials, with a single environment override to swap models.
- **Tested and CI-gated.** A reference-case regression runner verifies that certified engines stay within tolerance and fails the build on a genuine drift, while cleanly skipping environments that lack a heavy optional backend. A separate routing eval measures end-to-end natural-language-to-engine accuracy without needing the engines installed.

## Tech stack

- **Language & core:** Python 3.11+, Pydantic v2 for typed intent and result models
- **LLM orchestration:** Instructor for structured extraction; Anthropic Claude and OpenRouter as interchangeable providers
- **Service & CLI:** FastAPI + Uvicorn (REST API and static web UI), Typer + Click + Rich (CLI)
- **Scientific core:** NumPy, SciPy, Pint for unit-aware quantities, Matplotlib for charts
- **Engine backends (optional extras):** a broad set of domain solvers across molecular dynamics, electronic structure, systems biology, agent-based/discrete-event modeling, sampling, and more — installed à la carte via extras
- **Execution & infra:** local runner plus a thin SLURM submitter with optional AiiDA provenance for running heavy pipelines on a SLURM HPC cluster; Docker / docker-compose for packaging
- **Quality:** pytest (with `slow` markers for heavy engine runs), Ruff, mypy

## Status

Actively developed. The core pipeline — intent parsing, routing, parameter binding, the validation stack, local and HPC execution with provenance, and grounded interpretation — is in place and exercised by the test suite, with a built-in analytical reference engine that makes the full flow testable end-to-end without any external dependency. The engine matrix is expanding, with heavier backends moving from experimental to certified as they are installed and validated against published reference cases.
