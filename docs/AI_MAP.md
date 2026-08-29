# Zero Hour AI Subsystem Map

Scope: `GeneralsMD/Code/GameEngine/{Include,Source}/GameLogic/AI` (the `Generals/` tree is a
separate, near-duplicate engine copy and is not covered). Note: unlike the nominal layout,
headers for this module live flat under `Include/GameLogic/*.h`, not under an `AI/` subdirectory
— only the `.cpp` sources are grouped in a `GameLogic/AI/` folder.

## 1. File inventory

| Path | Lines | Purpose |
|---|---|---|
| `Include/GameLogic/AI.h` | 1053 | `AI` subsystem singleton (`TheAI`), `AIGroup` class, `AICommandInterface`/`AICommandParms` (the command vocabulary every unit and group speaks), AI INI data (`TAiData`), build-list side-data structs (`AISideInfo`, `AISideBuildList`). |
| `Include/GameLogic/AIStateMachine.h` | 1326 | Declares the generic `StateMachine`/`State` base classes and every concrete `AIState` subclass (move, attack, guard, dock, follow-path, etc.) used by unit-level AI. |
| `Include/GameLogic/AIPlayer.h` | 293 | `AIPlayer` class: base "skirmish/solo AI brain" per player — build list management, base building, team building. |
| `Include/GameLogic/AISkirmishPlayer.h` | 117 | `AISkirmishPlayer : public AIPlayer` — skirmish-specific specialization (harder opponent logic). |
| `Include/GameLogic/AIDock.h` | 202 | States/logic for docking (supply trucks, aircraft at airfield) as an `AIStateMachine` subtree. |
| `Include/GameLogic/AIGuard.h` | 270 | Guard-behavior states (`AIGuardMachine` and friends). |
| `Include/GameLogic/AIGuardRetaliate.h` | 255 | Retaliate-while-guarding state variant. |
| `Include/GameLogic/AITNGuard.h` | 246 | "Tunnel network" guard state (guards all tunnel entrances as one group). |
| `Include/GameLogic/TurretAI.h` | 372 | `TurretAI` class: independent turret aiming/rotation/firing logic, used by turreted objects. |
| `Include/GameLogic/Squad.h` | 96 | `Squad` class: lightweight named subset of a Team's objects (distinct from `AIGroup`). |
| `Source/GameLogic/AI/AI.cpp` | 1055 | Implements `AI` subsystem: `update()` per-frame driver, group create/destroy, INI parsers, `findClosestEnemy`/`findClosestAlly`. |
| `Source/GameLogic/AI/AIGroup.cpp` | 3364 | Implements every `AIGroup::group*` command (fan-out to member objects' `AICommandInterface`), group movement/formation math. |
| `Source/GameLogic/AI/AIPlayer.cpp` | 3877 | Implements `AIPlayer`: build-list consumption, base-building decisions, team production/AI orders, INI-driven skirmish tuning. |
| `Source/GameLogic/AI/AISkirmishPlayer.cpp` | 1235 | Implements `AISkirmishPlayer` overrides — target selection, base defense placement, harder-difficulty behavior. |
| `Source/GameLogic/AI/AIStates.cpp` | 7518 | Implements almost every `AIState` subclass declared in `AIStateMachine.h` — by far the largest and most load-bearing file in the module. |
| `Source/GameLogic/AI/AIDock.cpp` | 819 | Implements docking states. |
| `Source/GameLogic/AI/AIGuard.cpp` | 925 | Implements guard states. |
| `Source/GameLogic/AI/AIGuardRetaliate.cpp` | 880 | Implements guard-retaliate states. |
| `Source/GameLogic/AI/AITNGuard.cpp` | 891 | Implements tunnel-network guard state. |
| `Source/GameLogic/AI/Squad.cpp` | 266 | Implements `Squad`. |
| `Source/GameLogic/AI/TurretAI.cpp` | 1467 | Implements `TurretAI` targeting/rotation/fire-control. |

Related but out of primary scope (per-unit `AIUpdate` module implementations, in
`Include/GameLogic/Module/*AIUpdate.h` and matching `Source/.../Module/`): these are
`Object`-attached update modules (`AIUpdateInterface` and subclasses like `DozerAIUpdate`,
`JetAIUpdate`, `WorkerAIUpdate`) that own the per-object `StateMachine` instance and are the
actual thing `AIGroup`/`AIPlayer` commands act on. Referenced where needed for seam 3, not
inventoried line-by-line. Implementation is
`GeneralsMD/Code/GameEngine/Source/GameLogic/Object/Update/AIUpdate.cpp` (huge, contains
`AIUpdateInterface::aiDoCommand` and all `private*` command handlers).

## 2. Architecture overview

The AI subsystem has three cooperating layers. (1) A single `AI` singleton (`TheAI`,
`AI.h:311`) owns pathfinding and per-map AI tuning data (`TAiData`, INI-driven) and is a thin
per-frame driver — it does almost nothing itself except tick the pathfind queue and delegate to
players (`AI.cpp:356`). (2) Each `Player` optionally owns an `AIPlayer` (or its subclass
`AISkirmishPlayer`) which is the "opponent brain": it reads a `BuildListInfo` linked list
attached to the `Player`, decides when to build structures/teams/upgrades, and turns those
decisions into concrete engine calls (`TheBuildAssistant->buildObjectNow`,
`ProductionUpdateInterface::queueCreateUnit`). This layer runs at player-update granularity, not
per-unit. (3) Individual `Object`s carry an `AIUpdateInterface` (out of primary scope, in
`Module/AIUpdate.h`) wrapping an `AIStateMachine` — a table-driven finite state machine whose
states (`AIIdleState`, `AIAttackState`, `AIGuardState`, …) implement all actual unit behavior,
one `update()` call per logic frame. `AIGroup` (declared inside `AI.h`, not a separate header) is
a transient collection of `Object*`s used to fan a single high-level command (e.g.
"attack this position") out to each member's `AIUpdateInterface`. `Squad` (`Squad.h`) is a
lighter-weight, non-owning `ObjectID` list used to snapshot a `Team`'s membership without the
`AIGroup` allocation/refcount machinery; `Team::getTeamAsAIGroup` converts one to the other for
scripted or AI-driven team orders. `TurretAI` and its state machine are a fourth, mostly
independent layer: per-turret aim/fire logic driven from inside `AIUpdateInterface::update()`,
not from the group/player layers. Guard behaviors (`AIGuard`, `AIGuardRetaliate`, `AITNGuard`)
are structurally near-identical copy-pasted state-machine trees for three slightly different
"defend a spot / retaliate / defend a tunnel network" cases.

## 3. Key classes

**`AI`** — `Include/GameLogic/AI.h:245`. Singleton (`TheAI`) owning the pathfinder and the list
of live `AIGroup`s; per-frame entry point for the whole subsystem. Key methods: `update()`
(`AI.h:253`, impl `AI.cpp:356`), `createGroup()`/`destroyGroup()` (`AI.h:278-279`),
`findClosestEnemy()`/`findClosestAlly()` (`AI.h:265,270`), `parseAiDataDefinition()` (static INI
entry point, `AI.h:286`). Owned by the engine as a global; constructed once at startup (see
`AI::AI()`, `AI.cpp:302`).

**`AIGroup`** — `Include/GameLogic/AI.h:886`. Ad hoc, short-lived collection of `Object*` used
to broadcast one command to many units. Key methods: `add()`/`remove()`/`removeAll()`
(`AI.h:990,993,995`), every `group*` command (`groupAttackPosition`, `groupMoveToPosition`,
`groupGuardArea`, etc., `AI.h:904-962`), `getSpeed()`/`getCenter()` (group movement math,
`AI.h:981-982`). Only constructible via `AI::createGroup()` (private ctor, `friend class AI`,
`AI.h:1026-1028`); lifetime is either raw-pointer (retail-compatible, current default) or
refcounted (`RefCountPtr<AIGroup>`), selected at compile time by `RETAIL_COMPATIBLE_AIGROUP`
(`Core/GameEngine/Include/Common/GameDefines.h:126`, currently `1`).

**`AIPlayer`** — `Include/GameLogic/AIPlayer.h:147`. The default computer-opponent brain, one
per computer `Player`. Key methods: `update()` (`AIPlayer.h:160`, impl `AIPlayer.cpp:3033`),
`doBaseBuilding()`/`processBaseBuilding()` (`AIPlayer.h:220-221`), `doTeamBuilding()`
(`AIPlayer.h:223`), `buildStructureNow()`/`buildStructureWithDozer()` (`AIPlayer.h:243-244`),
`newMap()` (map-load hook that seeds the build list, `AIPlayer.h:162`). Owned by `Player::m_ai`
(`Common/Team.h`-adjacent `Player.h`), constructed in `Player::setPlayerType`
(`Common/RTS/Player.cpp:752`).

**`AISkirmishPlayer`** — `Include/GameLogic/AISkirmishPlayer.h:41`, `: public AIPlayer`. Harder,
skirmish-specific opponent: overrides base-defense placement, build-list adjustment to starting
position (`adjustBuildList`, `AISkirmishPlayer.h:100`), and auto-acquiring an enemy
(`acquireEnemy`, `AISkirmishPlayer.h:102`). Constructed instead of plain `AIPlayer` when
`skirmish` is true or `TheAI->getAiData()->m_forceSkirmishAI` (`Player.cpp:747-753`).

**`AIStateMachine`** — `Include/GameLogic/AIStateMachine.h:128`, `: public StateMachine`
(`StateMachine` itself is in `Common/StateMachine.h`, out of primary scope). Per-`Object`
finite-state machine holding every possible unit AI state; states are pre-instantiated once in
the constructor (`AIStates.cpp:672-729`) via `defineState(ID, StateInstance, successID,
failureID)`. Key methods: `setState()` (`AIStateMachine.h:141`, impl `AIStates.cpp:1049`),
`updateStateMachine()` (`AIStateMachine.h:164`, impl `AIStates.cpp:850`), `setGoalPath`/
`setGoalWaypoint`/`setGoalTeam`/`setGoalSquad`/`setGoalAIGroup` (goal-parameter setters used by
states, `AIStateMachine.h:144-156`). One instance owned per `AIUpdateInterface`
(`Module/AIUpdate.h`, out of primary scope) — i.e. one per `Object` that has AI.

**`TurretAI`** — `Include/GameLogic/TurretAI.h:259`. Independent turret aim/fire controller,
driven by its own `TurretStateMachine` (5 states: idle/idle-scan/aim/fire/recenter/hold). Key
methods: `updateTurretAI()` (`TurretAI.h:308`), `setTurretTargetObject()`/
`setTurretTargetPosition()` (`TurretAI.h:294-295`), `isTryingToAimAtTarget()` (`TurretAI.h:306`).
Owned by `Object`s with a turreted weapon (via `AIUpdateInterface`, count depends on
`WhichTurretType`), not by `AIGroup`/`AIPlayer`.

**`Squad`** — `Include/GameLogic/Squad.h:61`. Lightweight `VecObjectID`-backed snapshot of a
subset of objects, cheaper than `AIGroup`. Key methods: `squadFromTeam()`/`squadFromAIGroup()`/
`aiGroupFromSquad()` (`Squad.h:89,93-94`, the `Team`⇄`Squad`⇄`AIGroup` conversion points),
`getLiveObjects()` (`Squad.h:83`). Used internally by `AIStateMachine::setGoalSquad`
(`AIStateMachine.h:155,178`) to hold a group-attack target list that outlives the `AIGroup` that
created it.

## 4. The three seams

### 4.1 Per-frame entry point

Full call chain, one invocation per logic-frame tick:

1. `GameEngine::update()` — `Source/Common/GameEngine.cpp:896`, gated by
   `canUpdateGameLogic()`/`canUpdateRegularGameLogic()` (`GameEngine.cpp:851`), which accumulates
   real time against a target frame time and only proceeds once a full logic-frame interval has
   elapsed (`GameEngine.cpp:868-887`) — in this fork logic rate is decoupled from render rate
   (comment: "TheSuperHackers @tweak xezon 06/08/2025", `GameEngine.cpp:875-876`).
2. `TheGameLogic->UPDATE()` — `GameEngine.cpp:921` → `GameLogic::update()`,
   `Source/GameLogic/System/GameLogic.cpp:3687`.
3. `TheAI->UPDATE()` — `GameLogic.cpp:3892` → `AI::update()`, `Source/GameLogic/AI/AI.cpp:356`.
   This does pathfind-queue processing (`AI.cpp:359`) then `ThePlayerList->UPDATE()`
   (`AI.cpp:363`).
4. `PlayerList::update()` — `Source/Common/RTS/PlayerList.cpp:253`, loops
   `m_players[i]->update()` for all `MAX_PLAYER_COUNT` player slots (human and computer alike).
5. `Player::update()` — `Source/Common/RTS/Player.cpp:669`, calls `m_ai->update()`
   (`Player.cpp:672`) if the player has an AI brain (human players do not: `m_ai` is null).
6. Virtual dispatch to `AISkirmishPlayer::update()` (`Source/GameLogic/AI/AISkirmishPlayer.cpp:953`,
   which just calls `AIPlayer::update()`) or directly `AIPlayer::update()`
   (`Source/GameLogic/AI/AIPlayer.cpp:3033`) for a plain solo-AI player. This is the actual
   "skirmish AI player update" the task asks for — it runs `doBaseBuilding()`,
   `checkReadyTeams()`, `checkQueuedTeams()`, `doTeamBuilding()`, `doUpgradesAndSkills()`,
   `updateBridgeRepair()` in that order every tick (`AIPlayer.cpp:3037-3047`).

**Tick rate**: `LOGICFRAMES_PER_SECOND = WWSyncPerSecond = 30`
(`Core/Libraries/Source/WWVegas/WWLib/WWCommon.h:46`, aliased at
`Core/GameEngine/Include/Common/GameCommon.h:68`). So absent throttling this is a 30 Hz tick;
`AIPlayer::doBaseBuilding` itself self-throttles further via `m_buildDelay`/`m_structureTimer`
counters measured in these logic frames (e.g. re-checks every `2*LOGICFRAMES_PER_SECOND` frames,
`AIPlayer.cpp:2764`).

Per-unit AI (`AIUpdateInterface`/`AIStateMachine::updateStateMachine`) is driven separately, from
the generic sleepy-update-module loop in `GameLogic::update()` (`GameLogic.cpp:3838-3886`,
immediately before the `TheAI->UPDATE()` call) — each `Object`'s update modules (including its
`AIUpdateInterface`) are scheduled independently and may sleep for N frames between calls
(`UpdateSleepTime`), so individual units are not guaranteed to tick every single frame the way
`AIPlayer::update()` is. [UNVERIFIED: exact default sleep duration for `AIUpdateInterface`
specifically — depends on per-`ThingTemplate` data not read for this map.]

### 4.2 Build lists

**Representation**: `BuildListInfo` (`Include/GameLogic/SidesList.h:247`) — a singly linked list
node holding a template name, world position/angle, "initially built" flag, and a rebuild
counter (`m_numRebuilds`, `UNLIMITED_REBUILDS` sentinel at `SidesList.h:262`). Two independent
sources populate it:
- **Map build list**: placed on the map in WorldBuilder, parsed per-side from the `.map` file via
  `BuildListInfo::parseStructure` (declared `SidesList.h:267`) and attached to each `Player` via
  `Player::getBuildList()`/`setBuildList()` (consumed at `AIPlayer.cpp:3058,1431`, etc.). This is
  what a solo-AI (non-skirmish) player builds from as-is.
- **Skirmish faction build list**: authored in `AI.ini` under the `AIData` block's
  `SkirmishBuildList <faction>` sub-block (INI field table `TheAIFieldParseTable`,
  `Source/GameLogic/AI/AI.cpp:178`), parsed by `AI::parseSkirmishBuildList`
  (`AI.cpp:276-291`, reusing the same `BuildListInfo::parseStructure` field parser for its
  `Structure` entries), stored as a linked list of `AISideBuildList` per faction
  (`TAiData::m_sideBuildLists`, `AI.h:234`). `AI::parseAiDataDefinition` (`AI.h:286`, impl
  `AI.cpp:420`) is the top-level INI callback wired to the `AIData` block via
  `INI::parseAIDataDefinition` (`Core/GameEngine/Source/Common/INI/INIAiData.cpp:47-50`).

**Loading into a game**: `AISkirmishPlayer::newMap()` (`AISkirmishPlayer.cpp:1072`) looks up the
faction list matching `m_player->getSide()`, `duplicate()`s it, calls `adjustBuildList()`
(repositions the list relative to the player's actual start command-center,
`AISkirmishPlayer.cpp:962` region) and installs it via `m_player->setBuildList()`
(`AISkirmishPlayer.cpp:1083`). Plain `AIPlayer::newMap()` (`AIPlayer.cpp:3056`) instead just
scans placed objects for production buildings and adds them (`AIPlayer.cpp:3061-3073`) — it uses
whatever build list the map/`SidesList` already gave the player.

**Consumption / decision cadence**: `AIPlayer::doBaseBuilding()` (`AIPlayer.cpp:2745`, called
every `AIPlayer::update()` tick) self-throttles via `m_structureTimer`/`m_buildDelay` and calls
`processBaseBuilding()` (`AIPlayer.cpp:707`, overridden in `AISkirmishPlayer.cpp`) which walks
the *current* `m_player->getBuildList()` linked list (`AIPlayer.cpp:716`), and for each entry:
checks whether the building still exists, and if not and rebuilds remain
(`info->isBuildable()`, `AIPlayer.cpp:786`), rebuilds it.

**Exact decision→action point** (structures): `AIPlayer::buildStructureWithDozer()`
(`AIPlayer.cpp:516`) finds an idle dozer, validates the placement, and calls
`TheBuildAssistant->buildObjectNow(dozer, bldgPlan, &pos, angle, m_player)`
(`AIPlayer.cpp:629`) — this is the line where an AI decision becomes an actual game object.
(`AIPlayer::buildStructureNow()`, `AIPlayer.cpp:452`, used for initially-built/instant
structures, calls the same `TheBuildAssistant->buildObjectNow(...)` at `AIPlayer.cpp:456`.)

**Exact decision→action point** (units): team composition is queued as `WorkOrder`s
(`Include/GameLogic/AIPlayer.h:45`) under a `TeamInQueue` (`AIPlayer.h:88`);
`AIPlayer::queueUnits()` (`AIPlayer.cpp:2676`) walks pending work orders and calls
`AIPlayer::startTraining()` (`AIPlayer.cpp:1400`), which finds a factory
(`findFactory`, `AIPlayer.cpp:1428`) and calls
`factory->getProductionUpdateInterface()->queueCreateUnit(order->m_thing, ...)`
(`AIPlayer.cpp:1406`) — the line where a "build this unit" decision becomes a real production
queue entry. `ProductionUpdateInterface` is out of primary scope (a `Module` on the factory
`Object`).

### 4.3 Team orders

**`AIGroup` fan-out**: every `AIGroup::group*` command iterates `m_memberList`
(`std::list<Object*>`, `AI.h:1039`) and calls the matching `AICommandInterface` method on each
member's `AIUpdateInterface` (obtained via `Object::getAIUpdateInterface()`/`getAI()`). Example —
`AIGroup::groupAttackPosition()` (`Source/GameLogic/AI/AIGroup.cpp:2266`) loops members and calls
`ai->aiAttackPosition(&attackPos, maxShotsToFire, cmdSource)` (inline wrapper defined in
`AI.h:609-615`, builds an `AICommandParms{AICMD_ATTACK_POSITION, ...}` and calls
`aiDoCommand()`); it also special-cases garrisoned containers so passengers inside a building can
fire out (`AIGroup.cpp:2282-2305`). `groupAttackTeam()` (`AIGroup.cpp:2246`) is the same pattern
one level up (per-member `aiAttackTeam(team, ...)`).

**Command dispatch inside a unit**: `AIUpdateInterface::aiDoCommand()`
(`Source/GameLogic/Object/Update/AIUpdate.cpp`, giant `switch` on `AICommandType`) routes
`AICMD_ATTACK_POSITION` to `privateAttackPosition()` (`AIUpdate.cpp:2714-2716` dispatch,
implementation `AIUpdate.cpp:3515`), which ultimately calls
`getStateMachine()->setState(AI_ATTACK_POSITION)` (`AIUpdate.cpp:3568`). This transitions the
unit's pre-built `AIStateMachine` into the pre-instantiated `AIAttackState`
(`AIStateMachine.h:978`, instance created once at `AIStates.cpp:705`), whose `update()`
(impl in `AIStates.cpp`, not read line-by-line — file too large) runs the actual chase/aim/fire
sub-state-machine (`AttackStateMachine`, `AIStateMachine.h:188`) every subsequent logic frame
until the exit condition fires.

**Team → AIGroup bridge**: `Team::getTeamAsAIGroup(AIGroup*)` (`Include/Common/Team.h:274`, impl
`Source/Common/RTS/Team.cpp:1431`) fills a caller-supplied `AIGroup` with the team's current
members. This is the near-universal entry point both the script engine
(`Source/GameLogic/ScriptEngine/ScriptActions.cpp`, 50+ call sites, e.g. team-attack-area,
team-attack-nearest-group actions) and `AIPlayer` itself (`AIPlayer.cpp:910`, `AIPlayer.cpp:2817`,
`AIStates.cpp:4188` for `AI_ATTACK_SQUAD`-style follow-up) use to turn "a named Team" into
something that can be issued group orders.

**Narrowest external API surface to order a group to attack a location**: obtain (or already
hold) a `Team*`, call `team->getTeamAsAIGroup(aiGroup)` to populate an `AIGroup`, then call
`aiGroup->groupAttackPosition(const Coord3D *pos, Int maxShotsToFire, CommandSourceType
cmdSource)` (`AI.h:925`, impl `AIGroup.cpp:2266`). For a single already-in-hand `Object`, the
even narrower surface is `object->getAI()->aiAttackPosition(pos, maxShotsToFire, cmdSource)`
(`AI.h:609`). Both eventually converge on `AIUpdateInterface::aiDoCommand` /
`AIStateMachine::setState(AI_ATTACK_POSITION)` as above — there is no lower-level "attack" entry
point than the state machine transition itself.

## 5. Data flow

- **INI-loaded, load-once, effectively read-only during a match**: `TAiData` (global AI tuning —
  timers, distance modifiers, group-formation thresholds; `AI.h:139`, table
  `AI.cpp:134-202`, block name `AIData`), skirmish faction build lists (`AISideBuildList`,
  `AI.h:122`) and per-side resource-gatherer/skillset info (`AISideInfo`, `AI.h:98`), `TurretAIData`
  (per-`ThingTemplate` turret tuning, `TurretAI.h:213`).
- **Map-file-loaded, then mutated at runtime**: the per-`Player` `BuildListInfo` linked list
  (`Player::getBuildList()`) — starts from the map's placed structures / skirmish faction
  template, then `AIPlayer::processBaseBuilding()` mutates `m_objectID`/`m_numRebuilds`/
  `m_objectTimestamp` on each node as buildings are built, destroyed, and rebuilt
  (`AIPlayer.cpp:726-804`).
- **Runtime-only, per-player**: `AIPlayer`'s team-build queues (`TeamInQueue`/`WorkOrder`
  linked lists, `AIPlayer.h:238-239`), timers (`m_teamTimer`, `m_structureTimer`,
  `m_buildDelay`, `AIPlayer.h:262-269`), base center/radius cache (`m_baseCenter`,
  `AIPlayer.h:275`).
- **Runtime-only, per-object**: `AIStateMachine`'s current state and goal fields
  (`m_goalPath`, `m_goalWaypoint`, `m_goalSquad`, `AIStateMachine.h:176-178`); `TurretAI`'s angle/
  pitch/target (`TurretAI.h:354-370`). None of this is INI-derived; it is pure play-session
  state, saved/loaded only through each class's `xfer()`/`crc()` (the engine's savegame and
  network-sync serialization hooks — every AI class implements `Snapshot`).
- **Cross-cutting**: `AIGroup` itself is never persisted as a top-level entity — it is always
  either a call-scoped temporary (built, used, released within one function) or referenced by ID
  from things that are persisted (e.g. `AIStateMachine::setGoalAIGroup` internally stores a
  `Squad` snapshot instead, `AIStateMachine.h:156,158`, presumably because `AIGroup` membership
  is too transient to serialize directly). [UNVERIFIED: did not read `AIGroup::xfer()`
  (`AI.h:895`) impl in `AIGroup.cpp` to confirm what, if anything, it actually serializes.]

## 6. Landmines

- **`AIStates.cpp` is 7518 lines** — by far the largest file in the module, implementing nearly
  every `AIState` subclass declared across `AIStateMachine.h`, `AIDock.h`, `AIGuard.h`,
  `AIGuardRetaliate.h`, and `AITNGuard.h`. It was only grepped for specific symbols per the task
  method, never read end to end. Expect it to contain most future bugs and most future
  spelunking time.
- **`AIPlayer.cpp` (3877 lines) and `AIGroup.cpp` (3364 lines)** are similarly oversized single
  files mixing many unrelated responsibilities (base building, team building, bridge repair,
  supply-center logic all in one `.cpp` for `AIPlayer`; every group command for `AIGroup`).
- **`AIGuard.h`/`AIGuardRetaliate.h`/`AITNGuard.h` are near-duplicate copy-pasted state trees**
  (inner/idle/outer/return/get-crate/attack-aggressor states, same shape, three times) for
  "guard a spot", "guard-and-retaliate", and "guard a tunnel network". A change to one guard
  behavior likely needs manually mirroring into the other two — there is no shared base beyond
  `AttackExitConditionsInterface`.
- **Dead code behind `ALLOW_SURRENDER`**: prisoner pick-up/return commands
  (`AICMD_PICK_UP_PRISONER`, `AICMD_RETURN_PRISONERS`, `AIGroup::groupSurrender`,
  `groupPickUpPrisoner`, `groupReturnToPrison`, `AI.h:374-377,662-677,946-955`) are compiled only
  if `ALLOW_SURRENDER` is defined; grep across the repo found zero definitions of that macro, so
  this is permanently dead code as shipped.
- **Two `AIGroup` lifetime models coexist in source**: a raw-pointer "retail-compatible" mode and
  a `RefCountPtr<AIGroup>` mode, switched by `RETAIL_COMPATIBLE_AIGROUP`
  (`Core/GameEngine/Include/Common/GameDefines.h:126`, currently `1`/retail mode). Reading
  `AIGroup` usage without checking which mode is active will be misleading about who owns/frees
  groups.
- **Explicit "do not subclass" warning** on `AIStateMachine` itself
  (`AIStateMachine.h:119-125`): subclassing it inherits *all* states whether wanted or not;
  the pattern used everywhere else (`AIDockMachine`, `AIGuardMachine`, `TurretStateMachine`) is to
  derive from the generic `StateMachine` base directly instead, and hand-pick states with
  `defineState`.
- **The AI headers do not live under an `AI/` subdirectory** — only `.cpp` files do
  (`Source/GameLogic/AI/*.cpp`); headers are flat in `Include/GameLogic/*.h`. This diverges from
  the task's assumed layout and from the directory convention used by e.g. `GameLogic/Module/`.
- **`AICommandParms`/`AICommandParmsStorage`/`AICommandInterface`** (`AI.h:415-878`) is a
  hand-rolled tagged-union-style command object with ~50 convenience wrapper methods, one per
  `AICommandType`. Adding a new AI command means touching the enum (`AI.h:351`), the parms
  struct, the storage class's `store`/`reconstitute`, the interface wrapper method, the
  `AIGroup::group*` mirror method, and the `aiDoCommand` switch in `AIUpdate.cpp` — six edit
  sites for one new verb.

## 7. Open questions

- **Exact `AIUpdateInterface` sleepy-update scheduling interval** — `GameLogic.cpp:3838-3886`
  shows units are ticked through a sleep/wake priority queue (`UpdateSleepTime`), not
  unconditionally every frame like `AIPlayer::update()`. I did not trace what determines a given
  unit's default sleep length, so I cannot state precisely how often a given unit's
  `AIStateMachine::updateStateMachine()` actually runs relative to the nominal 30 Hz logic tick.
- **`AIGroup::xfer()` contents** (`AI.h:895`, impl presumably in `AIGroup.cpp`) — not read. Given
  section 5's inference that `AIGroup` is treated as transient/call-scoped, it's unclear what (if
  anything) meaningfully gets saved/network-synced for a group versus reconstructed from
  underlying `Object`/`Team` state on load.
- **`GuardMode` enum** — referenced throughout `AIGuard.h`/`AITNGuard.h`/`AI.h` but its
  definition was not located (not in any of the ten headers read for this map); likely in
  `Common/` or `GameLogic/Object.h`. Left unresolved because it wasn't load-bearing for the three
  priority seams.
- **How `TAiData::m_forceIdleFramesCount`, `m_minInfantryForGroup`/`m_minVehiclesForGroup`, and
  the other INI-tunable thresholds actually gate group-vs-individual movement** — `AI.h:210-218`
  documents the fields' *intent* in comments, but the code path that reads them (presumably in
  `AIGroup.cpp`'s formation/movement math or `AIStates.cpp`'s follow-path states) was not traced;
  I only confirmed where the values are parsed, not consumed.
- **Whether `AISkirmishPlayer` difficulty (easy/normal/hard) changes build-list selection or only
  gatherer counts/skillsets** — `AISideInfo` (`AI.h:98`) carries three gatherer counts and five
  skillsets clearly keyed to difficulty, and `AIPlayer::getAIDifficulty()`/`setAIDifficulty()`
  exist (`AIPlayer.h:191-192`), but I did not trace every read site of `m_difficulty` to confirm
  the full scope of what it affects.
- **Interaction between script-issued team orders (`ScriptActions.cpp`) and AI-player-issued team
  orders on the same `Team`** — both paths converge on `Team::getTeamAsAIGroup` +
  `AIGroup::group*`, but I did not check whether there's any locking/priority mechanism to stop a
  script order and an `AIPlayer` order from fighting over the same team in the same frame, or
  whether `CommandSourceType` (`FROM_SCRIPT` vs `FROM_AI`) is used downstream to arbitrate that.
