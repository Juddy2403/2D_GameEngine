# 2D Game Engine

This repository contains a small 2D C++ codebase for arcade-style projects. It is intentionally minimal and explicit:
SDL2 for windowing/input/rendering, GLM for math, Dear ImGui for debug tooling, and straightforward systems you can
extend.

This document explains how the engine is structured, the design patterns it uses, and how the core systems are
implemented.

---

## Overview

- Language: C++
- Rendering/Input/Windowing: [SDL2](https://www.libsdl.org/)
- Math: [GLM](https://github.com/g-truc/glm)
- Debug/UI: [Dear ImGui](https://github.com/ocornut/imgui) (vendored)
- Configuration/Serialization: [nlohmann/json](https://github.com/nlohmann/json)
- Platform/IDE: Windows + Visual Studio (x64)

Repository layout:

- Minigin/ — engine scaffolding (game loop, window, input, basic rendering glue)
- GameProject/ — example project using the engine
- Data/ — assets (textures, fonts, etc.)
- 3rdParty/ — external libraries (Dear ImGui, etc.)

---

## Build & Run

1. Clone the repository.
2. Open `Minigin.sln` in Visual Studio (x64).
3. Point `sdl.props`
4. Set `GameProject` as the startup project for an example game.
5. Build and run.

Dear ImGui is vendored under `3rdParty`, no extra setup required.

---

## Engine Architecture

The runtime frame flow:

1. Poll input (SDL events).
2. Update game state with delta time.
3. Render 2D scene (SDL renderer).
4. Render ImGui overlays.

Modules (high level):

- Core: game loop, timing, window lifecycle.
- Input System: command mapping, controller/keyboard handling via Pimpl.
- Audio System: event-driven, thread-backed, accessed via Service Locator.
- Rendering: SDL texture/sprite helpers and ImGui for debug.
- Entity/Component model: lightweight components attached to game objects.
- Utilities: resource management helpers, timers, math (via GLM).

---

## Design Patterns

The codebase uses a small set of well-known patterns to keep systems decoupled and testable.

### Game Loop

Central loop drives input, update, render at a steady cadence.

- Input sampling happens before update.
- Fixed or semi-fixed delta (SDL ticks) used in `Update`.
- Render collects draw calls and presents once per frame.

```c++
while (doContinue)
    {
        float frameStartTime = SDL_GetTicks() / 1000.0f; // Get the current time in seconds
        
        time.Update();
        doContinue = input.ProcessInput();

        sceneManager.Update();
        renderer.Render();

        float frameEndTime = SDL_GetTicks() / 1000.0f; // Get the current time in seconds
        float frameTime = frameEndTime - frameStartTime; // Calculate the elapsed time 

        if (frameTime < g_TargetFPS)
        {
            SDL_Delay(static_cast<Uint32>((g_TargetFPS - frameTime) * 1000)); // Delay for the remaining time
        }
    }
```

### Command (Input)

Maps user intent to executable actions. Decouples input devices from gameplay code.

- Actions (Jump, Attack, Pause, etc.) map to `Command` objects.
- Commands operate on receivers (player, UI, camera).

Example:

```c++
class Command { explicit Command(GameObject* actor); virtual void Execute() = 0; };

class Move final: public Command {
    explicit Move(GameObject* actor,const glm::vec2& direction, int speed = 100);
    void Execute() override;
};

// Binding
 input.BindCommand(GameEngine::KeyboardInputKey::A,
 std::make_unique<GameEngine::Move>(GetGameObjParent(), glm::vec2{ -1.f,0.f }, m_PlayerSpeed));

// Dispatch per frame
switch (command->ExecuteOnKeyState())
{
case Command::ExecuteOn::keyPressed:
    if (m_pControllers[i]->IsKeyPressed(inputKey))
        command->Execute();
    break;
case Command::ExecuteOn::keyUp:
    if (m_pControllers[i]->IsKeyUp(inputKey)) command->Execute();
    break;
case Command::ExecuteOn::keyDown:
    if (m_pControllers[i]->IsKeyDown(inputKey)) command->Execute();
    break;
}
```

### Pimpl (Input backends)

Hides device-specific code (e.g., XInput/SDL GameController) behind a stable interface.

- `InputSystem` exposes a small public API.
- Internals are `struct Impl` with platform-specific details.
- Reduces compile-time and dependency leakage.

```c++
class Controller final
	{
	public:
		explicit Controller(unsigned int controllerIdx);
		~Controller();
		void ProcessControllerInput() const;

	private:
		class XInput;
		std::unique_ptr<XInput> m_pXInput;
	};
```

### Observer (Events)

Loosely couples event producers and consumers (e.g., UI reacting to game state, SFX reacting to gameplay events).

- Subscribers register callbacks for event types.
- Publishers emit events without knowing listeners.

```c++
void GameEngine::Subject::Notify(int event, int message, EventData* eventData)
{
    for (auto& observer : m_Observers[message])
        observer->Notify(this, event, eventData);
}
```

### Event Queue (Async tasks)

Used to decouple producers from consumers and cross thread boundaries (notably for audio).

- Producers enqueue lightweight commands/events.
- Consumer thread processes them at its own cadence.

```c++
void SdlSoundSystem::PlaySound(const SoundId id, const int volume)
{
    if ((m_QueueTail + 1) % maxPending == m_QueueHead)
    {
        std::cerr << "Too many pending sounds\n";
        return;
    }
    for (int i = m_QueueHead; i != m_QueueTail; i = (i + 1) % maxPending)
    {
        if (m_PendingSounds[i].id == id)
        {
            // Use the larger of the two volumes.
            m_PendingSounds[i].volume = std::max(volume, m_PendingSounds[i].volume);
            // Don't need to enqueue.
            return;
        }
    }
    std::lock_guard<std::mutex> lock(m_Mutex);
    m_PendingSounds[m_QueueTail] = { id,volume };
    m_QueueTail = (m_QueueTail + 1) % maxPending;
    m_ConditionVariable.notify_one();
}
```

### Service Locator (Audio)

Provides a globally accessible service without hard dependencies.

- Gameplay code calls `ServiceLocator::GetSoundSystem()`.
- Defaults to a no-op “Null” implementation (safe in tests).
- Can swap in a real backend at startup.

```c++
ServiceLocator::GetSoundSystem().Play(swordSwing, 0.9f);
```

### Singleton (Minimal use)

Used sparingly where a single instance is structurally required (e.g. a top-level configuration or the Service Locator
instance). Prefer explicit ownership where feasible.

### State

For encapsulating behavior across discrete modes (e.g., main menu, playing, paused) or within AI/animation state
machines.

- Each state implements a consistent interface (`Enter/Update/Exit`).
- Transitions are explicit and owned by a state machine.

```c++
virtual void Enter([[maybe_unused]] EnemyComponent* enemyComponent);
virtual void Exit([[maybe_unused]] EnemyComponent* enemyComponent);
```

### Component (Entity-Component Model)

Game objects are lightweight containers of components (e.g., Transform, Sprite, Collider).

- Components own specific behavior and data.
- Systems operate on sets of components.
- Favors composition over inheritance.

```cpp
gameObject = std::make_unique<GameEngine::GameObject>(static_cast<int>(GameId::misc));
gameObject->AddComponent<FormationComponent>();
scene->AddObject(std::move(gameObject));
```

### Dirty Flag

Avoids unnecessary recomputation or uploads.

- Components mark themselves dirty when state changes (e.g., Transform).
- Systems update only when dirty, then clear the flag.

### Object Pool

Reduces allocation churn for frequently created/destroyed objects (e.g., bullets, particles, transient audio handles).

- Pre-allocates a pool.
- Reuse slots when lifetimes end.

### Optimization Techniques

- Threading: background thread(s) for audio playback and event consumption.
- Decoupling: interfaces and event-driven boundaries to isolate subsystems, improve testing, and reduce rebuild scope.
- Data locality: simple arrays/vectors for hot paths; minimize indirection in tight loops.

---

## Systems Specifics

### Input System

- Pattern(s): Command, Pimpl (to hide device backend), Observer.
- Devices: SDL Keyboard, SDL GameController (or platform-specific controller via the Pimpl).
- Mapping: actions -> `Command` objects; supports per-frame press/hold handling.

Flow:

1. Poll SDL events.
2. Translate device state to actions.
3. Enqueue/collect `Command`s for this frame.
4. Execute commands in update.

Notes:

- The Pimpl idiom keeps controller-specific code out of public headers.
- Swappable backends (e.g., keyboard-only vs controller) without changing client code.

### Sound/Audio System

- Pattern(s): Service Locator, Event Queue, Observer (optional), Threading.
- Access: `ServiceLocator::GetSoundSystem()` returns an interface (real or Null).
- Playback: gameplay code emits events (e.g., Play/Stop/SetVolume) to an audio queue; a dedicated audio thread processes
  them.

Flow:

1. Gameplay calls `SoundSystem::Play(SoundId, volume)`.
2. Implementation enqueues an `AudioEvent`.
3. Audio thread pops events and talks to the underlying audio backend.
4. Optional mixing/caching handled on the audio side.

Properties:

- Non-blocking from the game thread’s perspective.
- Thread-safe queue guarding between producers (game) and consumer (audio thread).
- NullSoundSystem provides safe no-op semantics when audio is disabled.

### Rendering

- SDL2-based 2D rendering (textures/sprites).
- Optional Dear ImGui overlays for runtime inspection and tweaking.
- Dirty-flagged transforms reduce redundant recompute.

### Entity/Component Layer

- Minimalistic component model.
- Typical components: Transform, Sprite/Render, Collider, Script-like behavior.
- Systems iterate over components per frame (Update/Render).

### Resources & JSON

- nlohmann/json used for level config/metadata.
- Simple resource helpers for textures, sounds, etc. 

---

## Configuration Examples

Action bindings (JSON):

```json
{
  "enemyType": "BossGalaga",
  "positions": [
    {"formationPosition": [298, 38], "formationStage": 1, "turn": 4},
    {"formationPosition": [335, 38], "formationStage": 1, "turn": 5},
    {"formationPosition": [261, 38], "formationStage": 1, "turn": 6},
    {"formationPosition": [372, 38], "formationStage": 1, "turn": 7}
  ]
}
```
# Start project provided

## Minigin 

Minigin is a very small project using [SDL2](https://www.libsdl.org/) and [glm](https://github.com/g-truc/glm) for 2D c++ game projects. It is in no way a game engine, only a barebone start project where everything sdl related has been set up. It contains glm for vector math, to aleviate the need to write custom vector and matrix classes.

[![Build Status](https://github.com/avadae/minigin/actions/workflows/msbuild.yml/badge.svg)](https://github.com/avadae/msbuild/actions)
[![GitHub Release](https://img.shields.io/github/v/release/avadae/minigin?logo=github&sort=semver)](https://github.com/avadae/minigin/releases/latest)

## Goal

Minigin can/may be used as a start project for the exam assignment in the course 'Programming 4' at DAE. In that assignment students need to recreate a popular 80's arcade game with a game engine they need to program themselves. During the course we discuss several game programming patterns, using the book '[Game Programming Patterns](https://gameprogrammingpatterns.com/)' by Robert Nystrom as reading material. 

## Disclaimer

Minigin is, despite perhaps the suggestion in its name, not a game engine. It is just a very simple sdl2 ready project with some of the scaffolding in place to get started. None of the patterns discussed in the course are used yet (except singleton which use we challenge during the course). It is up to the students to implement their own vision for their engine, apply patterns as they see fit, create their game as efficient as possible.

---

# Extending the Engine

- Create an Event Queue for the input system.
- Improve the Observer pattern to be more decoupled. 
- Implement the flyweight pattern for sprites.

---

# Known Limitations

- No built-in physics/audio mixing middleware (audio is minimal by design).
- No editor or content pipeline.
- Single-platform build files (Visual Studio). Cross-platform project files are not included here.

---

# Libraries

- [SDL2](https://www.libsdl.org/) — windowing, input, rendering
- [GLM](https://github.com/g-truc/glm) — math
- [Dear ImGui](https://github.com/ocornut/imgui) — debug UI
- [nlohmann/json](https://github.com/nlohmann/json) — JSON parsing

---

# Credits & License

- Based on the minimal “Minigin” starter concept.
- Third-party components are used under their respective licenses.
- See [LICENSE](./LICENSE) for this repository’s licensing.
