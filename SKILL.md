---
name: dioxus
version: 1.0.0
description: Expert Dioxus development for Rust-based cross-platform apps (web, desktop, mobile, server). Covers dx CLI workflows, RSX components, signals state management, fullstack server functions, and mobile builds (iOS/Android via WKWebView/Android WebView).
source: github:vl4dt/dioxus-skill
license: MIT OR Apache-2.0
---

# Dioxus

## Rules
1. **ALWAYS use `dx` CLI** — orchestrates cargo builds, hot-reloading, bundling.
2. **ALWAYS enable correct Cargo feature** (`web`/`desktop`/`mobile`) before building.
3. **ALWAYS prefer signals** for local reactive state.

## Quick start

```bash
cargo install dioxus-cli --locked
dx new my_app && cd my_app && dx serve
```

## DX CLI commands

| Command | Purpose |
|---------|---------|
| `dx new <name>` | Create project (bare-bones / jumpstart / workspace) |
| `dx serve` | Dev server with hot-reloading (default: web) |
| `dx serve --platform android|ios|web|desktop` | Target platform |
| `dx serve --hotpatch` | Subsecond Rust hot-patching (experimental) |
| `dx build` | Production build |
| `dx bundle` | Package (.app/.msi/.deb/.ipa/.apk) |
| `dx run` | Run without hot-reload (CI) |
| `dx check` | Diagnose issues |

### Fullstack

```bash
dx build @client --features client-feats @server --features server-feats
```

## Core patterns

```rust
// Component
use dioxus::prelude::*;
#[component] fn App() -> Element { rsx! { h1 { "Hello!" } } }
fn main() { dioxus::launch(App); }

// Signals
let mut count = use_signal(|| 0);
count += 1;        // write
*count             // read

// Hooks: use_signal | use_memo | use_callback | use_context::<Signal<T>> | use_future
// Props: #[derive(Props, Clone)] struct P { title: String, on_click: Callback<MouseEvent> }
// Server: #[server] pub async fn fetch(id: String) -> Result<String, ServerFnError>
// Router: #[derive(Routable)] enum Route { #[route("/")] Home { }, #[route("/:id")] Detail { id: String } }
```

## Mobile

```bash
dx serve --platform android     # Android emulator/device
dx serve --platform ios         # iOS simulator
dx bundle --platform android    # .apk
dx bundle --platform ios        # .ipa (needs Mac)
```

**Architecture:** webview — WKWebView on iOS, Android WebView via JNI.

**FFI:** `#[cfg(target_os = "android")] use jni::*; #[cfg(target_os = "ios")] use objc2::*; #[cfg(target = "wasm32")] use web_sys;`

**Remote Mac:** GitHub Actions (`macos-14`), Scaleway, SSH + `xcrun`. See [REFERENCE.md](REFERENCE.md).

## Debugging

```bash
dx check; dx doctor
RUST_LOG=debug dx serve
DX_TELEMETRY_ENABLED=false dx serve
```

| Problem | Fix |
|---------|-----|
| Android fail | Set `ANDROID_HOME`; install SDK/NDK |
| iOS non-Mac | Requires Xcode; use remote Mac or CI runner |
| Hot-reload | Use `dx serve`, not `cargo run` |
| Features | `dx serve` reads Cargo.toml features; enable `web`/`desktop`/`mobile` |

## Key resources

- **Docs:** https://dioxuslabs.com/learn/0.7/
- **API:** https://docs.rs/dioxus/latest/dioxus/
- **GitHub:** https://github.com/DioxusLabs/dioxus
- **Discord:** https://discord.com/invite/dioxus
