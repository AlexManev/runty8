# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Runty8 is a Pico8 clone written in Rust that supports both native and WebAssembly targets. It provides a complete game development environment with an integrated editor and standalone runtime.

## Build and Test Commands

### Running Examples
```bash
# Run examples with editor (press Escape to toggle between game and editor)
cargo run --bin celeste -- --game
cargo run --bin confetti -- --game
cargo run --bin moving-box -- --game

# Run in editor mode (default without --game flag)
cargo run --bin celeste

# List all available examples
cargo run --bin
```

### Building and Testing
```bash
# Check all examples compile
cargo check --all-targets

# Run tests in a specific crate
cargo test -p runty8-core
cargo test -p runty8-editor

# Run all tests
cargo test --workspace

# Generate and open documentation
cargo doc --open
```

### WebAssembly Build
The `build_script.sh` builds WASM targets and serves them locally:
```bash
# Build and serve a specific example (default: standalone-game)
./build_script.sh [package-name] [--release]

# Examples:
./build_script.sh celeste
./build_script.sh moving-box --release
```
This builds for `wasm32-unknown-unknown`, runs `wasm-bindgen`, and serves via `serve` command.

## Architecture

### Crate Structure
The project uses a workspace with clear separation of concerns:

- **`runty8`**: Main entry point crate - users depend on this
  - Provides `debug_run()` which runs the editor in debug builds and runtime in release builds
  - Re-exports core types and functions from other crates

- **`runty8-core`**: Core types and Pico8 API implementation
  - `Pico8` struct: Main API providing Pico8 functions (drawing, input, maps, sprites)
  - `App` trait: Games implement `init()`, `update()`, `draw()` methods
  - `Resources`: Contains game assets (sprite sheet, map, sprite flags)
  - Core data structures: `DrawData`, `SpriteSheet`, `Map`, `Flags`, `Input`, `State`
  - Platform-agnostic (supports WASM via feature flags)

- **`runty8-runtime`**: Standalone game runtime
  - Runs games at fixed 30 FPS (33.33ms delta time)
  - Handles game loop, input events, and rendering
  - Uses `runty8-event-loop` for cross-platform windowing

- **`runty8-editor`**: Interactive editor environment
  - Contains sprite editor, map editor, and other tools
  - Custom UI framework in `ui/` module (button, slider, text, cursor)
  - Editor features: brush tools, bucket fill, undo/redo, notifications
  - `--game` flag determines if app starts in game or editor mode
  - Escape key toggles between game and editor

- **`runty8-winit`**: Integration layer for `winit` windowing library

- **`runty8-event-loop`**: Cross-platform OpenGL/WebGL event loop
  - Thin abstraction over `winit`/`glow`/`glutin`

- **`examples`**: Sample games demonstrating the API
  - Each example is a separate binary target
  - Notable example: `celeste` - full port of the original Celeste game

### Key Patterns

**Game Structure**: Games implement the `App` trait:
```rust
impl App for GameState {
    fn init(pico8: &mut Pico8) -> Self { /* ... */ }
    fn update(&mut self, pico8: &mut Pico8) { /* ... */ }
    fn draw(&mut self, pico8: &mut Pico8) { /* ... */ }
}
```

**Asset Loading**: Use the `load_assets!` macro:
```rust
let resources = runty8::load_assets!("celeste").unwrap();
runty8::debug_run::<GameState>(resources).unwrap();
```

**Pico8 API**: The `Pico8` struct provides the full Pico8 API surface:
- Drawing: `pset()`, `pget()`, `cls()`, `camera()`, `spr()`, `map()`
- Input: `btn()`, `btnp()`
- Map: `mget()`, `mset()`
- Sprites: `fget_n()`, `fset()`, `fset_all()`
- Palette: `pal()`, `palt()`, `reset_pal()`

**Fixed Frame Rate**: Runtime runs at 30 FPS using frame accumulation to handle variable delta times.

**Editor Integration**: The editor wraps game instances and intercepts Escape key to toggle modes. When in editor mode, tools can modify the `Resources` (sprites, maps, flags) which are serializable.

### Important Implementation Details

- The project targets stable Rust (see `rust-toolchain.toml`)
- Both native and WASM builds are first-class targets
- The editor uses a custom immediate-mode UI framework (not external dependencies)
- Game state persists when toggling between editor and game modes
- Resources are serializable to support editor save functionality

