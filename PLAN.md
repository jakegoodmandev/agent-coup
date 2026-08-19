# Plan: Agent Coup

A cheat-proof engine for Coup (the classic card game of incomplete information) where
player agents are defined by markdown strategy files interpreted by LLMs. The core
engine enforces all rules and information hiding so nobody — human or agent — can cheat.

## End State

1. Play Coup with friends (local hot-seat first, online multiplayer later)
2. Play against an LLM agent whose strategy is defined by a markdown prompt
3. Run simulations of agents playing each other to find dominant strategies
4. Bonus: an evolution loop where the engine mutates strategy prompts and improves
   them over time to find the ultimate strategy

## Decisions

- **Language/tooling:** Python, managed with `uv`, tested with `pytest`, linted with `ruff`
- **Multiplayer:** local hot-seat first; online (FastAPI + WebSockets) as a later phase
- **Agent brains:** LLM interprets markdown strategy files
- **LLM layer:** provider-agnostic (Anthropic / OpenAI / Ollama adapters); Ollama enables
  cheap mass simulation

## Architecture Overview

```
agent-coup/
├── pyproject.toml            # uv-managed, pytest + ruff
├── src/coup/
│   ├── engine/               # Pure rules engine (no I/O, no LLM)
│   │   ├── state.py          # GameState, Player, Card, Deck
│   │   ├── actions.py        # Action/response definitions
│   │   ├── machine.py        # State machine: turn → challenge → block → resolve
│   │   ├── views.py          # Per-player redacted views (info hiding)
│   │   └── log.py            # Event log (public + secret streams)
│   ├── agents/
│   │   ├── base.py           # Agent protocol: decide(view, request) -> choice
│   │   ├── random_agent.py   # Baseline
│   │   ├── heuristic.py      # Scripted baselines (honest, aggressive, etc.)
│   │   ├── human_cli.py      # Human via terminal prompts
│   │   └── llm_agent.py      # Markdown strategy → LLM decisions
│   ├── llm/
│   │   ├── provider.py       # Abstract: complete(messages) -> str
│   │   └── providers/        # anthropic, openai, ollama adapters
│   ├── cli.py                # `coup play`, `coup sim`, `coup evolve`
│   ├── sim/                  # Tournament runner + stats
│   └── evolve/               # Strategy self-improvement loop
├── strategies/               # Markdown agent files
└── tests/                    # Heavy rules coverage
```

## Key Design Decisions

### 1. Cheat-proof by construction

The engine is the sole authority:

- Agents never see the full state — only a `PlayerView` (own cards, everyone's
  coin/card *counts*, public event history).
- The engine computes **legal choices** for every decision and hands them to the agent
  as an enumerated menu. Agents return an index/ID; anything invalid → retry once →
  forfeit to a default legal move. An agent literally cannot act on hidden info or
  make an illegal move.

### 2. Engine as an inversion-of-control state machine

The engine never calls agents directly; it emits
`DecisionRequest(player, type, legal_options, view)` and waits for an answer. This
makes it trivially reusable for CLI hot-seat now and WebSockets later, and makes
agents interchangeable (human / LLM / scripted).

### 3. The Coup interaction flow is the hard part

Explicit phases:

```
ActionDeclared → ChallengeWindow (all opponents, turn-order priority)
              → BlockWindow → BlockChallengeWindow
              → Resolution → CardLoss/Exchange sub-decisions
```

Modeled as a phase enum with a pending-decision queue. Full rules: 5 roles × 3 copies
(Duke, Assassin, Captain, Ambassador, Contessa); actions income / foreign aid / coup /
tax / assassinate / steal / exchange; forced coup at 10+ coins; challenge
reveal-and-redraw; double card loss on failed assassination challenge.

### 4. Determinism

Seeded RNG throughout → reproducible games, replayable from the event log. Essential
for debugging and fair simulations.

### 5. LLM agents

Markdown file = system prompt (strategy + personality). Per decision, the agent
receives: rendered game view, public history, its private scratchpad notes (persisted
across turns so it can track suspicions), and the legal-options menu. Returns JSON
`{reasoning, choice, notes}`. Provider-agnostic layer with Anthropic / OpenAI / Ollama
adapters.

## Phases

### Phase 1 — Core engine + tests (the foundation, most effort)

- Full rules implementation, per-player views, event log, replay
- Extensive pytest suite covering challenge/block edge cases
- Verify with random-agent self-play (thousands of games, invariant checks: card
  conservation, coin conservation, always terminates)

### Phase 2 — Agent protocol + CLI play

- Agent base class, `RandomAgent`, 2–3 heuristic agents
- `coup play` CLI: hot-seat humans (screen-clear between turns to hide hands),
  human vs agents, spectate agent vs agent with narrated output

### Phase 3 — LLM markdown agents

- Provider layer, prompt rendering, JSON parsing with fallback, scratchpad memory
- Ship 4–5 starter strategies (`strategies/honest_duke.md`, `bluffer.md`,
  `paranoid_challenger.md`, ...)
- Milestone: play against a prompt

### Phase 4 — Simulation harness

- `coup sim`: round-robin/pool tournaments, N seeded games, parallel execution,
  JSONL game logs
- Stats: win rate, Elo, action/bluff/challenge frequencies, avg game length
- Cost controls (model choice per agent, concurrency limits)

### Phase 5 — Strategy evolution (bonus)

- `coup evolve`: evolutionary loop — run tournament → feed standings + annotated
  transcripts of wins/losses to an optimizer LLM → it mutates/crosses strategy
  markdowns → evaluate offspring vs incumbent pool → keep winners
- Persist lineage tree so strategy evolution is observable generation over generation

### Phase 6 (future) — Online multiplayer

- FastAPI + WebSockets server wrapping the same engine, room codes, minimal web UI
- The IoC design from Phase 1 means the engine needs zero changes

## Rules Edge Cases to Nail (test list)

- Failed challenge on Assassinate → target can lose 2 influence in one turn
- Challenged player reveals correct card → shuffles it back, draws replacement
- Exchange: draw 2, return any 2 of 4
- Steal from a player with 0/1 coins; block-steal by Captain *or* Ambassador
- Must coup at 10+ coins; can't coup/assassinate without funds
- Elimination mid-challenge-window; 2-player endgame; dead players can't respond
