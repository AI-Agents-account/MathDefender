# Math Defender — Technical Implementation Plan (Unity / Android)

## 1. Overview

Math Defender is a mobile arcade/brain-training game for Android built with Unity. The player controls a hero moving horizontally (X axis) across the bottom of the screen with a static Y position. Enemies ("monsters") spawn at the top and move downward in waves. The core mechanic revolves around applying mathematical multipliers and interacting with buffs/gates to maximize score and survive as long as possible.

Key pillars of the implementation:
- Target platform: **Unity (URP where reasonable) / Android**
- **Static Y, swerve X movement** for the player character
- **Monsters** moving in **waves from top to bottom**
- **Buffs/Gates** spawning from the top near the sides
- **Math multipliers** applied via gates, buffs, and monster interactions
- **ScriptableObjects** as the main configuration mechanism
- **Object pooling** for performance and GC reduction
- Clean, modular architecture with clear separation between presentation and game logic

This document defines the architecture, core systems, data structures, and implementation steps.

---

## 2. High-Level Architecture

### 2.1 Scene Structure

Single main gameplay scene (e.g., `GameScene`) plus light-weight menu / boot scenes:
- `BootScene`
  - Initializes global managers (Audio, Save/Settings, SceneLoader)
  - Loads `MainMenuScene`
- `MainMenuScene`
  - Main UI, play button, options
  - Loads `GameScene` when starting a run
- `GameScene`
  - Core gameplay loop
  - Subsystems:
    - PlayerController (swerve movement)
    - MonsterWaveController & Spawners
    - Buff/Gate Spawner
    - Score & MathMultiplier system
    - ObjectPoolManager
    - UI HUD (score, multiplier, health/energy, pause)

Optional: A separate `GameOverScene` or an in-scene GameOver overlay.

### 2.2 Systems & Responsibilities

- **Player System**
  - Swerve movement on X axis
  - Static Y position
  - Collision handling with monsters, buffs, gates
- **Enemy (Monster) System**
  - Wave definitions (timing, pattern, types)
  - Movement from top to bottom
  - Despawn conditions (off-screen or destroyed)
- **Buff / Gate System**
  - Spawn logic on top side-lanes
  - Defines buff types and corresponding math multipliers or effects
- **Math Multiplier & Scoring System**
  - Maintains current multiplier state
  - Applies multipliers to base score events (e.g., kill, survive wave)
  - Handles temporary vs. persistent multipliers
- **Game Configuration System (ScriptableObjects)**
  - Stores:
    - Player movement parameters
    - Wave patterns and monster types
    - Buffs/gates and their math operations
    - Difficulty curves and level progression
- **Object Pooling System**
  - Manages pooled instances:
    - Monsters
    - Projectiles (if any)
    - Buffs/Gates
    - VFX
- **UI System**
  - HUD elements (score, multiplier, health, wave number)
  - Menus (pause, game over)
  - Localized text hooks (if needed later)
- **Game State / Flow Controller**
  - Orchestrates: Ready → Playing → Paused → GameOver
  - Interacts with scoring, waves, and UI

---

## 3. Coordinate System & Movement Design

### 3.1 World Layout (2D or 2.5D)

- Use a **2D Orthographic Camera** oriented on the XY plane.
- X axis: left-right movement of player.
- Y axis: vertical flow; monsters and buffs/gates spawn at high Y and move down.
- Optional: use SpriteRenderer + sorting layers or a lightweight 3D setup with a camera facing negative Z and objects placed on Z = 0.

Typical values:
- World width (visible): ~10 units (X in [-5, 5])
- World height (visible): ~18 units (Y in [-9, 9])
- Player Y position: fixed (e.g., Y = -7)

### 3.2 Player Swerve Movement

Movement pattern:
- Player follows input-based swerve: dragging or tilting finger horizontally changes X velocity/target position.
- Y coordinate remains fixed.
- Movement clamped between defined boundaries, e.g., `minX = -4.5f`, `maxX = 4.5f`.

Control options:
- **Touch drag relative:**
  - Track last touch position in screen space.
  - Convert delta movement to world-space X delta (scaled by sensitivity).
- **Touch follow absolute:**
  - Convert touch position to world-space X.
  - Smoothly Lerp player position.x towards target X.

Configuration via ScriptableObject (e.g., `PlayerConfig`):
- `float MoveSpeed`
- `float SwerveSensitivity`
- `float MinX`, `float MaxX`
- `float SmoothTime`

Implementation notes:
- Player movement logic lives in `PlayerController`, with all tunable numbers referenced from `PlayerConfig`.
- Physics: use kinematic Rigidbody2D or transform-based movement with custom collision detection (prefer Rigidbody2D for built-in collision and triggers).

---

## 4. Monsters (Enemies)

### 4.1 Enemy Data Model (ScriptableObjects)

Create base ScriptableObject: `EnemyConfig`.

Fields (example):
- `string id` (unique identifier, for debugging and referencing)
- Visual:
  - `Sprite sprite`
  - `Color tint`
- Gameplay:
  - `int baseHealth`
  - `int baseScore`
  - `float moveSpeed`
  - `float width` (if needed for lane spacing)
- Behavior:
  - `EnemyBehaviorType behaviorType` (StraightDown, SineWave, ZigZag, etc.)
  - `float behaviorAmplitude`
  - `float behaviorFrequency`
- Effects:
  - `bool canAffectMultiplier`
  - `int multiplierDeltaOnKill` (optional: add/subtract from current multiplier)

Optional:
- Audio clips for spawn/death
- VFX references

### 4.2 Enemy Prefab & Components

Create a prefab for enemy instances:
- Components:
  - `SpriteRenderer`
  - `Rigidbody2D` (kinematic)
  - `Collider2D` (trigger)
  - `EnemyController` script

`EnemyController` responsibilities:
- Assign its `EnemyConfig` reference.
- On `OnEnable()`, initialize visuals and state from `EnemyConfig`.
- Update movement in `Update`/`FixedUpdate`:
  - Base downward movement: `y -= moveSpeed * deltaTime`.
  - Optional lateral pattern based on `behaviorType` and time since spawn.
- Detect collisions with player/projectiles.
- Notify systems when destroyed or off-screen (return to pool instead of `Destroy`).

### 4.3 Waves and Patterns (ScriptableObjects)

Create `WaveConfig` ScriptableObject.

Fields:
- `string waveId`
- `List<WaveSpawnEntry> spawns`
- `float waveDuration` (optional) or computed from entries
- `int baseDifficulty` (for progression/selection)

`WaveSpawnEntry` structure:
- `EnemyConfig enemy`
- `float spawnTime` (seconds since wave start)
- `float spawnX` or `LaneIndex lane` (if lane-based)
- `float spawnY` (usually constant topY)
- `int count`
- `float interval` (for repeated spawns)

Wave manager: `MonsterWaveController`.
Responsibilities:
- Holds list/sequence of `WaveConfig` assets.
- Tracks current wave index and time.
- Spawns enemies according to `WaveSpawnEntry` schedule using ObjectPool.
- Emits events: `OnWaveStarted`, `OnWaveCompleted`.

### 4.4 Difficulty & Progression

Use either:
- A fixed list of waves in sequence (tutorial → easy → medium → hard)
- Or a dynamic generator that selects waves based on running difficulty budget.

For MVP, define 10–20 `WaveConfig` assets and cycle/increase difficulty.

---

## 5. Buffs & Gates

### 5.1 Concept

Buffs and gates are special objects spawning from the top, typically near the side lanes, that apply math-based modifiers to the player or current multiplier when collected or passed through.

Examples:
- `x2` score multiplier gate
- `x0.5` slow-motion buff (affects monster speed)
- `+3` multiplier additive gate
- `x-1` (invert score sign) as a punishing gate

### 5.2 Buff/Gate Data Model (ScriptableObjects)

Create base ScriptableObject: `MathModifierConfig`.

Fields:
- `string id`
- Visual:
  - `Sprite sprite`
  - `Color color`
- Logic:
  - `MathModifierType type` (MultiplyScore, MultiplyMultiplier, AddMultiplier, SetMultiplier, SlowTime, SpeedUpMonsters, Shield, etc.)
  - `float value` (e.g., 2.0 for x2, 0.5 for halving, 3 for +3)
  - `bool isTemporary`
  - `float duration` (seconds, if temporary)
- Spawn:
  - `float baseSpawnWeight` (for random selection)
  - `bool sideOnly` (true for side-lane gates)

### 5.3 Buff/Gate Prefab & Components

Prefab:
- `SpriteRenderer`
- `Collider2D` (trigger)
- `Rigidbody2D` or transform-only movement
- `BuffGateController` script

`BuffGateController` responsibilities:
- Holds reference to `MathModifierConfig`.
- Moves downward similarly to enemies but may use different speed/lanes.
- On trigger with player colliders:
  - Notifies `MathMultiplierSystem` and/or `GameState` with `ApplyModifier(MathModifierConfig)`.
  - Returns itself to pool.

### 5.4 Buff/Gate Spawner

`BuffGateSpawner` component:
- Uses lane positions near sides: e.g., left side lane at X = -3.5, right side lane at X = 3.5.
- Spawns gates at configurable intervals or in sync with certain waves.
- Selection:
  - Weighted random among `MathModifierConfig` list.
  - Optionally different lists for early/mid/late game.

Configuration via ScriptableObject: `BuffSpawnConfig`.
- `float minInterval`
- `float maxInterval`
- `List<MathModifierConfig> possibleBuffs`
- `float topYPosition`
- `Vector2[] lanePositions`

---

## 6. Math Multipliers & Scoring System

### 6.1 Core Concepts

- **Base score events**:
  - Killing an enemy
  - Surviving a wave
  - Collecting certain buffs
- **Multiplier state**:
  - Global multiplier applied to base score
  - Additional temporary modifiers from buffs/gates

Design options:
- `Score = baseScore * currentMultiplier` (where `currentMultiplier` is float)
- Optionally clamp multiplier within [minMultiplier, maxMultiplier].

### 6.2 Data Structures

Create ScriptableObject `ScoreConfig`:
- `int baseKillScore`
- `int baseWaveClearScore`
- `float initialMultiplier`
- `float minMultiplier`
- `float maxMultiplier`
- `AnimationCurve multiplierDecayCurve` (optional for time-based decay)

### 6.3 Multiplier System Implementation

`MathMultiplierSystem` (MonoBehaviour, singleton or injected service):
- Fields:
  - `float currentMultiplier`
  - `List<ActiveModifier>` activeTemporaryModifiers

`ActiveModifier` structure:
- `MathModifierConfig config`
- `float remainingTime`

Responsibilities:
- Initialize `currentMultiplier` from `ScoreConfig.initialMultiplier`.
- `ApplyModifier(MathModifierConfig modifier)`:
  - Switch on `modifier.type`:
    - `MultiplyMultiplier`: `currentMultiplier *= modifier.value`
    - `AddMultiplier`: `currentMultiplier += modifier.value`
    - `SetMultiplier`: `currentMultiplier = modifier.value`
    - Other types may trigger side effects (time slow, shields, etc.)
  - If `modifier.isTemporary`:
    - Add `ActiveModifier` to list, tracking remaining time.
- `Update()` loop:
  - Decrement `remainingTime` for active modifiers.
  - When `remainingTime <= 0`, remove modifier and revert effect if necessary.
- Clamp `currentMultiplier` using `ScoreConfig.minMultiplier` / `maxMultiplier`.

### 6.4 Score Manager

`ScoreManager`:
- Holds current score (int/long).
- Methods:
  - `AddScore(int baseAmount)`: computes `final = Mathf.RoundToInt(baseAmount * currentMultiplier)` and updates score.
  - Events for UI: `OnScoreChanged`, `OnMultiplierChanged`.

Integration:
- `EnemyController` calls `ScoreManager.AddScore(enemyConfig.baseScore)` on death.
- `MonsterWaveController` calls `AddScore(ScoreConfig.baseWaveClearScore)` when wave cleared.
- Buffs may directly call score manager.

---

## 7. ScriptableObjects: Configuration Design

### 7.1 Asset Types Summary

Create dedicated folders under `Assets/Game/Configs/`:
- `PlayerConfig` — movement & gameplay parameters
- `EnemyConfig` — enemy types
- `WaveConfig` — enemy waves
- `MathModifierConfig` — buffs/gates
- `BuffSpawnConfig` — buff/gate spawn rules
- `ScoreConfig` — scoring & multiplier parameters
- Optionally, `GameBalanceConfig` for global tuning

### 7.2 Example: PlayerConfig

```csharp
[CreateAssetMenu(menuName = "MathDefender/PlayerConfig")]
public class PlayerConfig : ScriptableObject
{
    public float moveSpeed = 10f;
    public float swerveSensitivity = 1f;
    public float smoothTime = 0.1f;
    public float minX = -4.5f;
    public float maxX = 4.5f;
}
```

### 7.3 Example: EnemyConfig

```csharp
public enum EnemyBehaviorType { StraightDown, SineWave, ZigZag }

[CreateAssetMenu(menuName = "MathDefender/EnemyConfig")]
public class EnemyConfig : ScriptableObject
{
    public string id;
    public Sprite sprite;
    public Color tint = Color.white;
    public int baseHealth = 1;
    public int baseScore = 10;
    public float moveSpeed = 3f;
    public EnemyBehaviorType behaviorType = EnemyBehaviorType.StraightDown;
    public float behaviorAmplitude = 1f;
    public float behaviorFrequency = 1f;
}
```

### 7.4 Example: WaveConfig

```csharp
[System.Serializable]
public class WaveSpawnEntry
{
    public EnemyConfig enemy;
    public float spawnTime;
    public float spawnX;
    public int count = 1;
    public float interval = 0.2f;
}

[CreateAssetMenu(menuName = "MathDefender/WaveConfig")]
public class WaveConfig : ScriptableObject
{
    public string waveId;
    public List<WaveSpawnEntry> spawns = new List<WaveSpawnEntry>();
}
```

### 7.5 Example: MathModifierConfig

```csharp
public enum MathModifierType { MultiplyMultiplier, AddMultiplier, SetMultiplier, SlowTime, SpeedUpMonsters, Shield }

[CreateAssetMenu(menuName = "MathDefender/MathModifierConfig")]
public class MathModifierConfig : ScriptableObject
{
    public string id;
    public Sprite sprite;
    public Color color = Color.white;
    public MathModifierType type;
    public float value = 1f;
    public bool isTemporary;
    public float duration = 5f;
}
```

---

## 8. Object Pooling Architecture

### 8.1 Goals

- Avoid frequent allocations and `Destroy` calls.
- Maintain stable performance on low/mid-range Android devices.

### 8.2 Pool Manager Design

Create `ObjectPoolManager` (singleton/service) with generic pooling capabilities.

Data structure:
- `Dictionary<string, ObjectPool>` keyed by pool ID or prefab name.

`ObjectPool`:
- Holds:
  - `GameObject prefab`
  - `Queue<GameObject> available`
  - `Transform parent` for hierarchy organization
  - `int initialSize`
  - `bool expandable`

Public API:
- `void RegisterPool(string id, GameObject prefab, int initialSize, bool expandable)`
- `GameObject Get(string id)`
- `void Return(string id, GameObject instance)`

### 8.3 Pool Usage

- Enemy spawner gets enemies via pool instead of `Instantiate`:
  - `var enemy = poolManager.Get("enemy_basic");`
- On enemy death or off-screen:
  - `poolManager.Return("enemy_basic", gameObject);`
- Same pattern for:
  - Buffs/Gates
  - Projectiles
  - Reusable VFX (explosions, hit flashes)

### 8.4 Initialization & Configuration

Use a ScriptableObject or MonoBehaviour `PoolConfig` to define initial pools:
- `PoolEntry[] entries` where each entry contains:
  - `string id`
  - `GameObject prefab`
  - `int initialSize`
  - `bool expandable`

On game start (BootScene or GameScene), PoolManager reads `PoolConfig` and pre-populates queues.

---

## 9. Game Flow & State Management

### 9.1 Game States

Define a simple state enum:

```csharp
public enum GameState { Ready, Playing, Paused, GameOver }
```

`GameFlowController` responsibilities:
- Hold current `GameState`.
- Transition logic:
  - `Ready → Playing` when player taps "Start" or after countdown.
  - `Playing → Paused` / `Paused → Playing` via UI.
  - `Playing → GameOver` when player dies or health reaches 0.
- Interact with:
  - `MonsterWaveController` (start/stop wave spawning as state changes).
  - `ScoreManager` (reset score on new run).
  - `MathMultiplierSystem` (reset multiplier on new run).
  - UI (show/hide overlays).

### 9.2 Game Loop

1. Player presses Play in `MainMenuScene`.
2. Load `GameScene`.
3. `GameFlowController` enters `Ready` state.
4. Optional short countdown; then switch to `Playing`:
   - Start enemy waves.
   - Enable player input.
5. During `Playing`:
   - Waves and buffs/gates spawn.
   - Score and multipliers update.
   - When failure condition met → `GameOver`.
6. In `GameOver`:
   - Show final score and high score.
   - Buttons: Retry (restart `GameScene`), Quit (go back to `MainMenuScene`).

---

## 10. UI & UX

### 10.1 HUD Layout

- Top-left: Score
- Top-right: Current multiplier (e.g., "x2.5")
- Center-top: Wave indicator (Wave 3, etc.)
- Bottom-center: Player (world-space)
- Optional bottom UI overlays: life/health, a progress bar for wave completion.

### 10.2 Implementation Details

- Use Unity UI (Canvas) with Screen Space - Overlay.
- Create `HUDController` that subscribes to `ScoreManager` and `MathMultiplierSystem` events.
- Ensure UI scales via CanvasScaler with reference resolution (e.g., 1080x1920).

### 10.3 Mobile Considerations

- Large touch target for pause button.
- Optionally add a simple tutorial overlay on the first run.

---

## 11. Android & Performance Considerations

### 11.1 Build Settings

- Platform: Android
- Orientation: Portrait
- Target Frame Rate: 60 fps (cap via `Application.targetFrameRate`).
- Graphics API: use defaults; URP if needed.

### 11.2 Optimization Practices

- Use **Object Pooling** for all frequently spawned objects.
- Avoid per-frame allocations; watch GC in profiler.
- Simple shaders and small textures for monsters and effects.
- Disable vs reduce physics where possible:
  - Use `Rigidbody2D` in kinematic mode and simple colliders.
  - Avoid unnecessary `FixedUpdate` usage.

### 11.3 Input Handling

- Use Unity’s `Input.touchCount` and `Touch` for fine control.
- Support both single-touch drag and tap-to-move (as an option).

---

## 12. Implementation Roadmap

### Phase 1 – Core Loop Prototype

1. Set up project, URP (optional) and Android build target.
2. Implement:
   - PlayerController with swerve X movement and static Y.
   - Simple Enemy prefab with straight-down movement.
   - Basic spawning at fixed intervals from the top.
3. Implement collision between player and enemies.
4. Temporary score counter without multipliers.

### Phase 2 – ScriptableObjects & Waves

1. Create `PlayerConfig`, `EnemyConfig`, `WaveConfig` assets.
2. Refactor Player and Enemy to use these configs.
3. Implement `MonsterWaveController` to spawn enemies according to `WaveConfig` data.
4. Implement wave progression and simple difficulty scaling.

### Phase 3 – Buffs/Gates & Math Multipliers

1. Implement `MathModifierConfig` and related prefabs.
2. Create `BuffGateSpawner` for side-lane spawning.
3. Implement `MathMultiplierSystem` and integrate with `ScoreManager`.
4. Update UI to display current multiplier and score.

### Phase 4 – Object Pooling

1. Implement `ObjectPoolManager` and pool classes.
2. Replace all `Instantiate`/`Destroy` for enemies, buffs, and VFX with pool operations.
3. Tune pool sizes and test on target/low-end devices.

### Phase 5 – Polish & UX

1. Improve visuals (sprites, VFX, simple animations).
2. Add audio (SFX for kills, buffs, UI clicks; optional background music).
3. Implement pause/game over menus and navigation.
4. Add simple settings (SFX volume, music toggle, sensitivity slider for swerve).

### Phase 6 – Optimization & Testing

1. Profiling on Android devices; identify bottlenecks.
2. Fix garbage allocations and spikes.
3. Add integration tests for wave progression and score calculations (using Unity Test Framework).
4. Final tuning of difficulty curves and multipliers via ScriptableObjects.

---

## 13. Folder & Naming Conventions

Suggested Unity project structure:

- `Assets/Game/`
  - `Scripts/`
    - `Player/` (PlayerController, PlayerInput)
    - `Enemies/` (EnemyController, MonsterWaveController)
    - `Buffs/` (BuffGateController, BuffGateSpawner)
    - `Systems/` (ScoreManager, MathMultiplierSystem, GameFlowController)
    - `Pooling/` (ObjectPoolManager, PoolConfig)
    - `UI/` (HUDController, MenuControllers)
  - `Configs/`
    - `PlayerConfig.asset`
    - `EnemyConfigs/`
    - `WaveConfigs/`
    - `BuffConfigs/`
    - `BuffSpawnConfigs/`
    - `ScoreConfig.asset`
  - `Prefabs/`
    - `Player/`
    - `Enemies/`
    - `Buffs/`
    - `VFX/`
    - `Managers/`
  - `Art/`
    - `Sprites/`
    - `UI/`
  - `Audio/`
    - `SFX/`
    - `Music/`

---

## 14. Testing Strategy

### 14.1 Unit & Integration Tests

Use Unity Test Framework for:
- Score calculation tests:
  - Applying multiple sequential multipliers.
  - Temporary modifiers expiration.
- Wave spawning tests:
  - Given `WaveConfig`, ensure correct number/timing of monsters.
- Object pool tests:
  - Ensure objects are reused and no memory leak from unreturned instances.

### 14.2 Manual QA

- Validate swerve controls feel good on physical Android device.
- Validate difficulty curve and the impact of buffs/gates.
- Confirm no frame drops during intense waves or many buffs on screen.

---

## 15. Future Extensions (Optional)

- Additional math mechanics:
  - Negative multipliers, fractions, chained operations.
  - Simple equations shown on gates (e.g., `(x + 2) * 3`).
- Multiple player abilities (dash, shield) driven by multipliers.
- Meta-game progression: unlock new buffs/waves/characters via earned currency.
- Online leaderboards using Google Play Games.

---

This plan defines the core technical architecture and implementation approach for Math Defender on Unity/Android, focusing on static Y / swerve X movement, monsters flowing from top to bottom, side-lane buffs/gates, math multipliers, ScriptableObject-driven configuration, and object pooling for performance.