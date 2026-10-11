# Kaggriculture — Shop-Routed Farming Agent

**Precomputed routes. Adaptive execution. Shop-aware economics.**

A single-file Python agent for [Kaggle's Kaggriculture competition](https://www.kaggle.com/competitions/kaggriculture), combining embedded farming schedules with reactive production, market optimization, and worker-management logic.

Built around the challenge of coordinating crops, animals, labor, inventory, and sales in a competitive simulation—not just choosing the next move.

## 🏆 Competition achievement

### Rank 190 · Leaderboard score 2761.8

**Nakul Maurya reached rank 190 on the Kaggriculture leaderboard**, as recorded in the screenshot below on September 20, 2026.

![Kaggriculture leaderboard showing Nakul Maurya at rank 190 with a score of 2761.8](assets/kaggriculture-leaderboard.png)

The author reports a field of approximately **10,000 participants**, including competitors from multiple labs—an achievement in a large-scale competitive agent-building setting.

<sub>This is a recorded leaderboard position, not a claim of final placement. The screenshot confirms the name, rank, and displayed score; the participant count and lab participation are author-reported. The screenshot does not identify the exact submitted source version.</sub>

---

## At a glance

| Component | Implementation |
| :--- | :--- |
| Agent interface | `agent(observation, configuration=None)` |
| Deployment | Single Python file: `main.py` |
| Base schedules | 41 embedded routes, each 719 steps long |
| Routing signal | First two unlocked shops, with explicit overrides |
| Controls | Farmer actions, hired-hand actions, market orders |
| Runtime imports | Python standard library |
| Simulator | `kaggle-environments` with Kaggriculture support |

## The approach

A fixed action tape is useful only while the live game remains compatible with it. Shop openings, worker availability, cash, inventories, and market conditions can change what makes sense to execute.

This agent uses **scheduled execution plus reactive correction**:

1. **Start from an embedded farming schedule.**
2. **Choose a route for the shop combination.**
3. **Adapt production, delivery, and market decisions to live state.**
4. **Apply targeted cash and animal-feeding safeguards.**

The result is a layered policy: the schedule supplies structure, while controllers handle selected deviations and economic opportunities.

### Core capabilities

- **Shop-aware production** — selects routes based on the first two unlocked shops.
- **Worker coordination** — aligns scheduled actions with available hired hands.
- **Field recovery** — includes weed-repair and other live-state corrections.
- **Crop and livestock management** — adjusts inputs, fertilizer, animal choices, feeding, and deliveries.
- **Market optimization** — combines sale timing, price modeling, rival-sensitive reservations, sale reordering, and queue compaction.
- **Storage and endgame handling** — includes overflow-related sales and terminal stock/cargo recovery.
- **Conservative rescue** — tries to save at-risk animals without abandoning productive worker tasks.

## Architecture

```text
Embedded action schedules
          │
          ▼
   Chassis + shop router
          │
          ▼
 Production, delivery, and market controllers
          │
          ▼
 Final SELL-block ordering
          │
          ▼
 Early hire-cash guard
          │
          ▼
 Conservative animal-feed rescue
          │
          ▼
 { farmer, hands, market }
```

The uploaded policy retains a substantial inherited wrapper/controller stack. Wrapper order matters: later layers can modify earlier outputs and update shared state.

### Route selection

The base `_router` uses zero-based game steps:

| Phase | Behavior |
| :--- | :--- |
| Before step 144 | Uses route `0`, with opening-controller adjustments. |
| From step 144 | Makes a one-time route decision using the first two unlocked shops. |
| From step 648 | Switches to route `2` for the endgame. |

Shop pairs without `YARN_STORE` use the newer table labeled `EXP240`; yarn-containing pairs use the older table labeled `V39`. Explicit shop-pair overrides and a narrow rival-state signature can further adjust the choice.

**The decision is step-triggered.** It assumes the expected shop-opening schedule rather than waiting indefinitely for a second shop. Later controllers can also alter the actions returned by the base schedule.

### Final safeguards

**Early hire-cash protection**

At steps **24 and 25**, when farm cash is below **5**, the final cash guard removes `BUY_PRODUCT`, `BUY_SEED`, `BUY_ANIMAL`, and `BUY_LAND` orders. Other orders remain. This is a narrow purchase-suppression rule, not a general affordability planner.

**Animal-feeding rescue**

The final rescue layer identifies unfed animals that have already missed at least one feeding day, then checks whether the remaining schedule covers them using the workers that actually exist.

If feeding is not covered, it may use an idle worker already on the animal's tile to feed, or pick up wheat when shed access and the next scheduled action permit it. The current drift limit is **zero**: workers are not sent away from their route to rescue an animal. Rescue starts after the daily hiring turn and stops on the final day.

## Quick start

### 1. Prepare the repository

```text
.
├── main.py
├── README.md
├── LICENSE.txt
└── assets/
    └── kaggriculture-leaderboard.png
```

Keep the image at the path shown above so it renders in this README on GitHub.

### 2. Create a Python environment

```bash
python -m venv .venv
source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install kaggle-environments
```

Use a simulator release that includes Kaggriculture and matches the intended competition rules. Pin its exact version for reproducible benchmarks.

### 3. Check the entrypoint

```bash
python -m py_compile main.py
python -c "import main; print('Agent callable:', callable(main.agent))"
```

### 4. Run a local smoke test

Save this as `run_local.py` beside `main.py`:

```python
import json
from pathlib import Path

from kaggle_environments import make
import main


def passive_agent(observation, configuration=None):
    farm = observation["farms"][int(observation["player"])]
    return {
        "farmer": ["PASS"],
        "hands": [["PASS"] for _ in (farm.get("hands") or [])],
        "market": [],
    }


env = make("kaggriculture", debug=True)
env.run([main.agent, passive_agent])

print([(state.status, state.reward) for state in env.state])
Path("replay.json").write_text(json.dumps(env.toJSON()), encoding="utf-8")
```

```bash
python run_local.py
```

This example is a smoke test, not a competitive benchmark, and should be checked against your installed simulator version. For meaningful evaluation, use multiple seeds, both seats, and stronger opponents.

## Entrypoint and configuration notes

- **Use `main.agent` for local tests.** The earlier `kaggle_submission_agent` alias omits the final hire-cash and feed-rescue layers in this file.
- **Check competition callable discovery.** Follow the current submission instructions and verify which function the runner executes.
- **Default runtime needs no external route file.** Schedules are embedded in `main.py`.
- **Optional predictor data:** `V92_SELL_LIB` can point to a compatible external JSON sell-stream library. Leave it unset unless that path has been validated.
- **Feature switches are layered.** Some chassis defaults are disabled by `_SETTINGS`, while later controllers implement their own behavior.
- **Use fresh processes for isolated benchmarks.** The agent maintains module-level state and mutates some schedules.

### Code landmarks

| Symbol | Role |
| :--- | :--- |
| `Chassis` / `make_agent` | Base execution, state tracking, and fallbacks |
| `_ROUTES` | Decoded base schedules |
| `_router` | Shop-pair selection and endgame switch |
| `_SETTINGS` | Initial chassis feature switches |
| `_final_sell_block_reorder_entrypoint` | Late contiguous-sale reordering |
| `_hr_guard` | Early low-cash purchase suppression |
| `_er_rescue` | Conservative animal-feed rescue |

## Validation and limitations

For the accompanying file, syntax parsing and local import succeeded, the final `agent` was callable, and the embedded base route count and lengths were inspected.

**No fresh simulator tournament or win-rate measurement was performed while preparing this documentation.** The leaderboard screenshot records the author's achievement; it has not been linked to this exact uploaded source version.

Before submitting changes, test worker-count changes, locked-field operations, low-cash purchases, crop inputs, feeding, deliveries, storage overflow, market execution, final liquidation, and runtime limits.

> **Version scope:** This is the uploaded shop-routed policy, not the separately developed Majkel three-quadrant router. It does not establish a two-land-upgrade limit, Southeast exclusion, or persistent land-purchase retry behavior. The safeguards described here are targeted corrections, not a complete legality guarantee.

## Credits and licensing

This project builds on upstream schedules and controllers. The full Apache License 2.0 text is provided in `LICENSE.txt`. `main.py` retains upstream attribution and modification notices, including credits to:

- thomastschinkel
- yhay81 / Yusuke Hayashi
- destbreso
- aurax7
- tetsutani
- prvsiyan
- Dmitrii Gluzdov
- Ahmed Berat Ozer

Individual source sections record additional contributions and later modifications, including prvsiyan/Codex. Preserve those notices; this summary does not replace them or claim sole authorship of inherited work.

### References

- [Kaggriculture competition](https://www.kaggle.com/competitions/kaggriculture)
- [Kaggle Environments](https://github.com/Kaggle/kaggle-environments)
- [Yusuke Hayashi — Shop Router 0913](https://www.kaggle.com/code/yhay81/shop-router-0913)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)

The embedded Apache License 2.0 text has been moved into `LICENSE.txt` for cleaner source presentation; agent logic is unchanged. Before publishing, verify the licensing of the included code and embedded action data, retain any required upstream `NOTICE` material, and identify your modifications clearly.
