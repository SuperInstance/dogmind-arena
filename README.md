# DogMind Arena — Train AI Dog Agents with Genetics and Trust

`dogmind-arena` is a Cloudflare Worker that simulates an interactive sheep-herding arena where AI dog agents learn through trust-based training. Each dog has a unique DNA genome (speed, patience, obedience, bravery, gentleness, social, intelligence, strength) that influences its herding behavior. Dogs are trained through repeated commands gated by a trust progression system — the more you work with a dog, the more commands unlock.

## Why It Matters

DogMind Arena is a **gameified testbed for genetic algorithms and agent-based simulation**. It demonstrates several important concepts in a playable format:

1. **Genetic inheritance** — DNA crossover and mutation produce offspring with blended parent traits, modeling real Mendelian inheritance.
2. **Emergent behavior** — Sheep flocking (boids-like), dog herding pressure, and panic responses emerge from simple local rules — no scripted animations.
3. **Trust-gated capability** — Commands unlock through a 5-tier trust progression (Stranger → Familiar → Friend → Partner → Bonded), modeling real dog training.
4. **LLM inner monologue** — Each dog has an AI-generated first-person narration of its actions, powered by Seed-2.0 / DeepSeek.

The tagline: *"The repo IS the kennel. Each dog IS an agent. Git IS the training log."*

## How It Works

### DNA System

Each dog carries a genome of 8 normalized traits ∈ [0, 1]:

| Trait | Effect |
|---|---|
| `speed` | Movement speed in simulation |
| `patience` | Delay before abandoning target |
| `obedience` | Skill gain rate per command |
| `bravery` | Willingness to approach sheep |
| `gentleness` | Reduces panic caused to sheep |
| `social` | Pack coordination effectiveness |
| `intelligence` | Pathfinding efficiency |
| `strength` | Herding force on sheep |

### Genetic Operators

**Crossover:** Uniform crossover selects each gene from either parent with 50% probability:

$$g_i = \begin{cases} g_i^{A} & \text{if } U(0,1) < 0.5 \\ g_i^{B} & \text{otherwise} \end{cases}$$

**Mutation:** Each gene has probability `rate = 0.15` of mutation:

$$g_i' = \text{clamp}(g_i + U(-1, 1) \times \text{strength},\ 0,\ 1)$$

where `strength = 0.2`.

### Trust Progression

Trust ∈ [0, 100] gates available commands:

| Tier | Range | Commands Unlocked |
|---|---|---|
| Stranger | 0–20 | `recall` |
| Familiar | 20–40 | + `heel` |
| Friend | 40–60 | + `flank`, `gather` |
| Partner | 60–80 | + `drive` |
| Bonded | 80–100 | + `hold` (all commands) |

Trust changes via: commands followed (+1), treat reward (+3), failed commands (−1).

### Skill Progression

Each skill has a level ∈ [0, 100] with qualitative thresholds:

| Level | Range | Meaning |
|---|---|---|
| Recipe | 10–29 | Knows the theory |
| Card | 30–59 | Has reference experience |
| Muscle | 60–89 | Internalized through repetition |
| Genetics | 90–100 | Bred for this skill |

### Physics Simulation

Sheep follow a boids-like flocking model:

For each sheep *s* and each dog *d*:

$$\vec{F}_{\text{repel}} = \frac{\vec{s} - \vec{d}}{|\vec{s} - \vec{d}|} \times \max(0,\ 80 - |\vec{s} - \vec{d}|) \times 0.3 \times (1 - g_d \times 0.5)$$

Where $g_d$ is the dog's gentleness trait — gentle dogs cause less panic. Velocity is damped: $\vec{v} \leftarrow 0.95\vec{v} + 0.05\vec{F}$, clamped to |v| ≤ 2.

### Complexity

| System | Per-Frame Cost | Notes |
|---|---|---|
| Dog movement | O(D) | D = dogs |
| Sheep forces | O(D × S) | S = sheep |
| Sheep inter-flocking | O(S²) | Pairwise separation/cohesion |
| Trail rendering | O(D × T) | T = trail length (max 40) |

## Quick Start

```bash
npm install
npx wrangler deploy
# Or local dev:
npx wrangler dev
```

Visit the deployed URL. Choose a preset dog (Rex, Biscuit, Thunder, Blue) or adopt a random pup. Tap the canvas to issue movement commands. Select commands from the panel to train specific skills.

### Environment Variables (for AI narration)

```toml
# wrangler.toml
[vars]
DEEPINFRA_API_KEY = "..."   # Seed-2.0-mini (primary)
SILICONFLOW_API_KEY = "..."  # Seed-OSS-36B (fallback)
DEEPSEEK_API_KEY = "..."     # DeepSeek (fallback)
```

## API

### Routes

| Route | Method | Response | Description |
|---|---|---|---|
| `/` | GET | `text/html` | Full game SPA |
| `/health` | GET | `application/json` | Health check |
| `/api/think` | POST | `application/json` | AI inner monologue for dog narration |
| `/vessel.json` | GET | `application/json` | SuperInstance vessel metadata |

### `/api/think`

```json
// Request
{ "prompt": "You are Rex, a Border Collie. Speed: 85%. Context: recalls to handler." }

// Response
{ "narration": "I hear my name. The handler's voice cuts through everything. I turn, I run. Nothing else matters right now." }
```

The narration system tries three providers in order: DeepInfra → SiliconFlow → DeepSeek. Returns empty string on total failure (graceful degradation).

## Architecture Notes

DogMind Arena demonstrates **γ + η = C**:

- **γ (gamma)**: The simulation specification — genetic inheritance rules, trust progression gates, boids-like flocking physics, and the dog-sheep interaction model. This is the *game design*.
- **η (eta)**: The TypeScript/Cloudflare Worker implementation — Canvas 2D rendering, `setInterval` game loop at 30fps, KV state persistence, LLM provider chain. This is the *engineering realization*.
- **C (Configuration)**: **Emergent herding behavior** — the gameplay that arises when genetics, trust, and physics interact. When γ and η are aligned, dogs with different DNA genuinely herd differently, and training history visibly changes behavior.

The simulation runs entirely client-side after the initial HTML load. The only server round-trip is `/api/think` for AI narration, which is non-blocking — the game is fully playable without any LLM calls.

## References

- **Reynolds, C. W. (1987).** "Flocks, Herds, and Schools: A Distributed Behavioral Model." *Proc. ACM SIGGRAPH*, pp. 25–34. — The boids algorithm underlying sheep flocking.
- **Holland, J. H. (1975).** *Adaptation in Natural and Artificial Systems.* University of Michigan Press. — Genetic algorithms: crossover and mutation operators.
- **Mitchell, M. (1996).** *An Introduction to Genetic Algorithms.* MIT Press. — Accessible introduction to GA operators and schema theorem.
- **Sutton, R. S., & Barto, A. G. (2018).** *Reinforcement Learning: An Introduction*, 2nd ed. MIT Press. — Trust-reward feedback loops.
- **DiGennaro, C., et al. (2026).** "SuperInstance: Agent Ecosystems as Repositories." — The γ + η = C framework and vessel architecture.
- **Reeves, W. T. (1983).** "Particle Systems—A Technique for Modeling a Class of Fuzzy Objects." *ACM Trans. on Graphics*, 2(2), 91–108. — Particle-based simulation influencing trail rendering.

## License

MIT
