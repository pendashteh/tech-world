# CLAUDE.md - Tech World

## Project Overview

Tech World ("Adventures In Tech World, for fun and profit") is a Flutter-based multiplayer 2D game combining gather.town-style virtual spaces with educational gameplay inspired by TwilioQuest. Players navigate a tile-based map, interact in real-time via WebSockets, and authenticate through Firebase.

**Note:** This repository was merged into the [enspyrco monorepo](https://github.com/enspyrco/monorepo/tree/main/packages/tech_world). The standalone repo is archived but retains the full commit history and codebase at commit `969d2ed`.

## Tech Stack

- **Language:** Dart (SDK >=2.13.0 <3.0.0)
- **Framework:** Flutter (>=2.2.0)
- **Game Engine:** Flame 1.0.0-rc.13
- **State Management:** Redux (via `redux` + `flutter_redux`)
- **Auth/Scaffolding:** Redfire (custom framework from enspyrco)
- **Code Generation:** Freezed + json_serializable + build_runner
- **Backend:** Firebase (Hosting + Firestore), WebSocket game server on Cloud Run
- **CI/CD:** GitHub Actions deploying to Firebase Hosting

## Repository Structure

```
lib/
├── main.dart                       # App entry point, store & game creation
├── main_page.dart                  # Main UI page wrapping the Flame game
├── tech_world_game.dart            # Flame BaseGame subclass (input, rendering, state sync)
├── game/
│   ├── game_state.dart             # GameState freezed model
│   ├── background/
│   │   └── barriers.dart           # Map barrier definitions
│   └── components/
│       ├── map_component.dart      # Tile map rendering & pathfinding
│       └── player_component.dart   # Player sprite, animation, movement
├── redux/
│   ├── app_state.dart              # Root AppState (extends RedFireState)
│   ├── actions/                    # Freezed action classes
│   ├── reducers/                   # Pure reducer functions
│   ├── middleware/                  # Side-effect middleware (auth, networking)
│   └── services/
│       ├── locator.dart            # Service locator for dependency injection
│       └── networking_service.dart  # WebSocket client for game server
├── shared/
│   ├── constants.dart              # Grid size, square size, server URLs
│   ├── direction_enum.dart         # Movement direction enum
│   └── local_constants.dart        # Local-only config (gitignored)
└── utils/
    ├── input.dart                  # Input handling helpers
    ├── movement_vector.dart        # Movement math
    ├── credentials.dart            # Credential utilities
    ├── effects/                    # Custom Flame animation effects
    └── extensions/                 # Dart extension methods on Vector2, Offset, etc.

test/
├── widget_test.dart
└── unit-tests/
    └── reducers/
        └── reducers_test.dart

.github/workflows/
├── on-pull-request.yml             # Build, test, preview deploy on PR
└── on-merge.yml                    # Build, test with coverage, deploy to live
```

## Development Commands

```bash
# Install dependencies
flutter pub get

# Run code generation (freezed, json_serializable)
flutter pub run build_runner build

# Run tests
flutter test

# Run tests with coverage
flutter test --coverage

# Static analysis
flutter analyze

# Build for web
cp web/index.auto web/index.html
flutter build web

# Run locally (web)
flutter run -d chrome

# Run locally (macOS)
flutter run -d macos
```

## Architecture & Patterns

### State Management (Redux)

The app uses a unidirectional data flow:

1. **AppState** (`lib/redux/app_state.dart`) — root immutable state combining RedFire auth state with `GameState`
2. **Actions** — freezed classes in `lib/redux/actions/` (e.g., `SetOtherPlayerIdsAction`, `SetPlayerPathAction`)
3. **Reducers** — pure functions in `lib/redux/reducers/` that transform state
4. **Middleware** — side effects in `lib/redux/middleware/` (auth subscriptions, networking setup)
5. **Store** — created in `main.dart`, combining redfire reducers with app-specific ones

### Game Engine (Flame)

- `TechWorldGame` extends `BaseGame` with `KeyboardEvents` and `TapDetector`
- Listens to the Redux store's `onChange` stream to sync game state
- Components: `PlayerComponent` (sprites, animation), `MapComponent` (tile grid, pathfinding via A*)
- Movement uses `MoveEffect` and custom `SpriteDirectionAnimationEffect`

### Service Locator

`Locator` class (`lib/redux/services/locator.dart`) provides lazy-initialized services:
- `NetworkingService` — manages WebSocket connections to the game server
- Nullable fields allow test mocking

### Code Generation (Freezed)

All data models use `@freezed` for immutability and serialization:

```dart
@freezed
class MyModel with _$MyModel {
  factory MyModel({required Type field}) = _MyModel;
  factory MyModel.fromJson(Map<String, Object?> json) => _$MyModelFromJson(json);
}
```

Generated files (`*.freezed.dart`, `*.g.dart`) are gitignored and must be regenerated after model changes.

## Naming Conventions

- **Files:** `snake_case.dart` (e.g., `set_player_path_action.dart`)
- **Classes:** `PascalCase` (e.g., `SetPlayerPathAction`)
- **Constants:** `camelCase` (e.g., `gridRows`, `squareSize`)
- **Private members:** underscore prefix (e.g., `_userId`, `_player`)
- **Actions:** `Verb` + `Noun` + `Action` (e.g., `SetOtherPlayerIdsAction`)
- **Reducers:** `Verb` + `Noun` + `Reducer` (e.g., `SetOtherPlayerIdsReducer`)
- **Middleware:** `Verb` + `Noun` + `Middleware` (e.g., `SetAuthUserDataMiddleware`)

## Linting & Analysis

Configured in `analysis_options.yaml`:
- Base: `package:flutter_lints/flutter.yaml`
- **Strong mode:** `implicit-casts: false`, `implicit-dynamic: false`
- Generated files excluded from analysis (`*.g.dart`, `*.freezed.dart`, `*.mocks.dart`)

## CI/CD Pipeline

### On Pull Request (`.github/workflows/on-pull-request.yml`)
1. Install Java + Flutter (dev channel)
2. `flutter pub get`
3. `flutter pub run build_runner build`
4. `flutter test`
5. Build web and deploy preview to Firebase Hosting

### On Merge to Main (`.github/workflows/on-merge.yml`)
1. Same setup steps
2. `flutter test --coverage`
3. Build web and deploy to Firebase Hosting live channel

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `flame` | 2D game engine |
| `redux` / `flutter_redux` | State management |
| `redfire` | Auth, routing, and app scaffolding (enspyrco) |
| `freezed` / `freezed_annotation` | Immutable data classes + code gen |
| `json_serializable` / `json_annotation` | JSON serialization code gen |
| `fast_immutable_collections` | Efficient immutable IList, ISet, IMap |
| `web_socket_channel` | WebSocket client |
| `ws_game_server_types` | Shared types with the game server |
| `a_star_algorithm` | A* pathfinding for map navigation |
| `vector_math` | Vector/matrix math for game coordinates |

## Important Notes

- `lib/shared/local_constants.dart` is gitignored — contains local-only configuration not available in CI
- `web/index.html` is gitignored — generated from `web/index.auto` during build to keep Firebase config out of version control
- Git dependencies (`redfire`, `ws_game_server_types`) are fetched from GitHub; commented-out path overrides in `pubspec.yaml` exist for local development
- `build.yaml` configures `json_serializable` with `explicit_to_json: true` so nested objects (including FIC collections) serialize correctly
