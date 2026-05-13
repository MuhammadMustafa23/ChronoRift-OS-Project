# ChronoRift —  Turn-Based RPG via Shared Memory IPC

A wave-based  RPG built in C++17 that demonstrates core Operating Systems concepts: **POSIX shared memory**, **semaphore synchronization**, **multi-process architecture**, **pthreads**, **signal handling**, and **deadlock detection**.

> **Course Project** — BCSA | 24i-0525 Muhammad Mustafa · 24i-0806 Muhammad Mughees Tariq Khawaja

---

## Architecture Overview

The game runs as **three separate processes** communicating through a single POSIX shared memory segment:

```
┌─────────────────────────────────────────────────────────────────┐
│                        SHARED MEMORY                            │
│  (SharedState: players, enemies, turn queue, artifacts, log)    │
└────────────────┬──────────────────┬───────────────┬────────────┘
                 │                  │               │
         ┌───────▼──────┐   ┌───────▼──────┐  ┌────▼──────┐
         │   ARBITER    │   │     HIP(s)   │  │    ASP    │
         │  (arbiters)  │   │   (hips)     │  │  (asps)   │
         │              │   │              │  │           │
         │ • Game loop  │   │ • Player 1   │  │ • Enemy   │
         │ • Renderer   │   │   input      │  │   AI per  │
         │ • Scheduler  │   │ • Player 2   │  │   thread  │
         │ • Deadlock   │   │   input      │  │           │
         │   monitor    │   │              │  │           │
         └──────────────┘   └──────────────┘  └───────────┘
```

### Process Roles

| Process | Binary | Responsibility |
|---------|--------|----------------|
| **Arbiter** | `arbiters` | Spawns HIP & ASP child processes; owns the SFML render thread; runs game loop, stamina timer, and deadlock monitor as pthreads |
| **HIP** (Human Input Process) | `hips` | Attaches to shared memory; spawns one `PlayerController` pthread per player; captures keyboard events via a hidden SFML window |
| **ASP** (Autonomous Simulation Process) | `asps` | Attaches to shared memory; spawns one `EnemyController` pthread per enemy; uses `EnemyAI` to pick targets and actions |

---

## OS Concepts Demonstrated

### 1. POSIX Shared Memory
- Single `shmget`/`shmat` segment keyed to `shmKey = 245826`
- `SharedState` struct holds all game data: players, enemies, turn queue, artifact table, action log, keyboard input, and all control flags
- All three processes read/write the same memory region

### 2. Semaphore Synchronization
| Semaphore | Purpose |
|-----------|---------|
| `shmLock` | General-purpose shared state guard |
| `turnSem` | Arbiter → entity: "your turn" signal |
| `actionSem` | Entity → Arbiter: "action submitted" signal |
| `artifactLock` | Resource table (artifact ownership) guard |
| `logLock` | Action log ring-buffer guard |
| `inputLock` | Keyboard input struct guard |
| `hipRequestLock` | HIP → Arbiter pause request guard |

### 3. Multi-Processing
- Arbiter uses `fork()`/`execv()` to spawn `hips` and `asps` as child processes
- PIDs tracked in `SharedState` (`hip1Pid`, `hip2Pid`, `aspPid`) for signal delivery and cleanup
- `waitForChildren()` / `killChildren()` manage process lifecycle

### 4. POSIX Threads (pthreads)
| Thread | Owner | Role |
|--------|-------|------|
| `gameLoopThread` | Arbiter | Main turn loop: grant turns, wait for actions, commit results |
| `renderThread` | Arbiter | SFML window: render game state at 60 fps |
| `staminaThread` | Arbiter | `StaminaTimer`: increments stamina each tick, enqueues ready entities |
| `deadlockThread` | Arbiter | `DeadlockMonitor`: DFS cycle detection on artifact wait-for graph |
| `initThread` | Arbiter | One-time initialization before game loop starts |
| `monitorThread` | Arbiter | Watches for enemy timeouts |
| Per-player thread | HIP | `PlayerController`: handles input phases for each player |
| Per-enemy thread | ASP | `EnemyController`: runs AI turn for each enemy |

### 5. Signal Handling
| Signal | Handler |
|--------|---------|
| `SIGTERM` | Graceful cleanup: release shared memory, kill children |
| `SIGALRM` | Enemy timeout enforcement |
| `SIGUSR1` | Wave transition / new-wave notification to ASP |
| `SIGUSR2` | Stun delivery to target entity's process |

### 6. Deadlock Detection
`DeadlockMonitor` builds a **resource allocation graph** from `SharedState.artifacts`:
- Nodes = entities (players + enemies)
- Edge A → B = entity A is waiting for an artifact held by entity B
- Runs **DFS cycle detection** every tick
- On cycle detected: picks a **victim** (lowest-priority entity in cycle), forces artifact release via `resolveDeadlock()`

### 7. Inventory Allocation
`InventoryAllocator` manages a **slot-based inventory** (20 contiguous slots per player):
- Weapons occupy variable slot sizes (2–10 slots each)
- First-fit allocation with long-term storage overflow (20 extra slots)
- Smallest-weapon eviction when inventory is full

---

## Game Design

### Characters (Playable)
| Sprite | Name |
|--------|------|
| Crono | Fast striker |
| Frog | Balanced |
| Magus | High damage |
| Slash | Heavy hitter |

### Enemies
Cybot, Goblin, Macabre, Gato, Grimalkin — assigned randomly per wave.

### Actions per Turn
| Action | Description |
|--------|-------------|
| `Strike` | Basic attack on selected enemy |
| `Exhaust` | Drain target's stamina |
| `Use Weapon` | Deal weapon damage (weapon consumed) |
| `Swap In` | Pull weapon from long-term storage |
| `Heal` | Restore 10% HP |
| `Skip` | Regain 50 stamina |
| `Ultimate` | Activate ultimate mode (10-turn duration) |

### Weapons (9 types)
| Name | Slots | Damage |
|------|-------|--------|
| Eclipse Rune | 10 | 100 |
| Solar Core | 10 | 95 |
| Lunar Blade | 10 | 90 |
| Iron Halberd | 7 | 55 |
| Thunderstaff | 6 | 50 |
| Frostbow | 6 | 48 |
| Obsidian Axe | 5 | 45 |
| Venom Dagger | 4 | 30 |
| Splinter Stick | 2 | 12 |

### Artifacts (Shared Resources — trigger deadlock scenarios)
- **Solar Core** — held exclusively; others must wait
- **Lunar Blade** — held exclusively; others must wait
- **Eclipse Relic** — spawns dynamically; player prompted to pick up or decline

### Win / Loss
- **Win**: Defeat 10 enemies across waves
- **Lose**: All players reach 0 HP

---

## Rendering (SFML)

Window: **1366 × 720** px

Layout regions:
- **HUD** (top 50 px): wave number, enemy kill count, alive counts
- **Battle area**: players on left 40%, enemies on right 40%, center panel for artifact/action display
- **Log panel** (bottom 160 px): scrolling action log with timestamps and PID/thread tags

Features:
- Sprite sheet animation (idle + attack) per entity
- Teleport animation for ranged attacks
- Death particle effects
- Stun indicators, selection circles (yellow = acting, red = targeted)
- HP and stamina bars per entity
- Full combat UI: action menu → target selection → weapon select → confirmation popup
- Main menu, party select, wave transition screen, win/lose screens, escape menu

---

## Build & Run

### Prerequisites

- Linux (POSIX APIs required)
- `g++` with C++17 support
- SFML 2.x (`libsfml-graphics`, `libsfml-window`, `libsfml-audio`, `libsfml-network`, `libsfml-system`)

```bash
sudo apt install libsfml-dev   # Debian/Ubuntu
```

### Build

```bash
make
```

Produces three binaries: `arbiters`, `hips`, `asps`

### Run

The Arbiter spawns HIP and ASP automatically:

```bash
./arbiters
```

For  (two keyboards on same machine), the Arbiter spawns a second HIP process automatically when party size > 2 is selected in the menu.

### Clean

```bash
make clean
```

---

## Project Structure

```
.
├── arbiter/
│   ├── arbiter.cpp              # Entry point — creates GameArbiter
│   ├── gameArbiter.cpp/.h       # GameArbiter, Renderer, TurnScheduler,
│   │                            # StaminaTimer, InventoryAllocator,
│   │                            # DeadlockMonitor, AnimationManager
│   ├── sharedMemoryUtilities.cpp/.h  # shmget/shmat/shmdt wrappers
│   ├── sharedState.h            # SharedState struct (entire game state)
│   └── stats.cpp/.h             # Statistics/logging helpers
│   └── assets/                  # Sprite sheets, fonts, backgrounds
├── hip/
│   ├── hip.cpp                  # Entry point — creates HIPManager
│   ├── hipManager.cpp/.h        # HIPManager, PlayerController, InterfaceState
│   ├── sharedMemoryUtilities.*  # Shared memory attach helpers
│   └── sharedState.h            # Mirror of arbiter's sharedState.h
├── asp/
│   ├── asp.cpp                  # Entry point — creates ASPManager
│   ├── aspManager.cpp/.h        # ASPManager, EnemyController, EnemyAI
│   ├── sharedMemoryUtilities.*  # Shared memory attach helpers
│   └── sharedState.h            # Mirror of arbiter's sharedState.h
├── Makefile
├── Dockerfile
└── report.pdf
```

---

## Docker

A `Dockerfile` is included for containerized builds. Note: SFML requires a display server; use with X11 forwarding or a virtual framebuffer (`Xvfb`) inside the container.

---
