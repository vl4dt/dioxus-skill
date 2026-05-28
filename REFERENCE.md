# Dioxus Reference

## Full CLI reference

### dx new templates

| Template | Description |
|----------|-------------|
| bare-bones | Simple setup with main.rs + assets folder |
| jumpstart | Scaffolded app with components, views, suggested structure |
| workspace | Full Cargo workspace with different crates per platform |

Prompts when creating: fullstack website (yes/no), router (yes/no), TailwindCSS (yes/no), LLMs (yes/no), default platform.

### dx serve options

```bash
dx serve [OPTIONS]

Options:
  --platform <web|desktop|android|ios>   Target platform
  --hotpatch                              Enable subsecond Rust hot-patching
  --example <name>                        Run an example from the Dioxus repo
  --no-default-features                   Pass to cargo build
  --features <features>                   Cargo features to enable
```

### dx build options

```bash
dx build [OPTIONS]

Options:
  --platform <platform>    Target platform
  --release                Build in release mode
  --features <features>    Cargo features
  --ssg                    Enable static site generation
  --out-dir <path>         Output directory
```

### dx bundle options

```bash
dx bundle [OPTIONS]

Options:
  --platform <platform>        Target platform
  --release                    Build in release mode
  --out-dir <path>             Destination directory
  --package-types <types>      Specific package formats (dmg, msi, deb, etc.)
```

## Dioxus.toml configuration

### Basic config

```toml
[app]
default_platform = "web"
title = "My App"
```

### Bundle config

```toml
[bundle]
identifier = "com.company.appname"
publisher = "Company Name"
icon = ["assets/icon.png"]

[bundle.android]
package_name = "com.company.appname"

[bundle.ios]
bundle_identifier = "com.company.appname"

[bundle.windows]
allow_downgrades = false

[bundle.deb]
depends = []
post_install_script = ""
```

## Fullstack development

### Server functions

```rust
use dioxus::prelude::*;

#[server]
pub async fn fetch_data(id: String) -> Result<String, ServerFnError> {
    // Server-side code runs on the backend
    Ok(format!("Data for: {}", id))
}

#[component]
fn DataFetcher(id: String) -> Element {
    let data = use_future(move || fetch_data(id));
    
    match data.value() {
        Some(Ok(v)) => rsx! { p { "{v}" } },
        Some(Err(e)) => rsx! { p { "Error: {e}" } },
        None => rsx! { p { "Loading..." } },
    }
}
```

### Router

```rust
use dioxus::prelude::*;

fn App() -> Element {
    rsx! {
        Router::<Route> {
            route: "/", { Home }
            route: "/about", { About }
            route: "/user/{id}", { UserPage }
        }
    }
}

#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[route("/")]
    Home { },
    #[route("/about")]
    About { },
    #[route("/user/:id")]
    UserPage { id: String },
}
```

### Fullstack project structure

```
my_app/
├── Cargo.toml          # client + server features generated
├── Dioxus.toml
├── server/
│   └── main.rs         # Server entry
├── src/
│   └── main.rs         # Client entry
└── assets/
```

The CLI splits input commands into separate cargo build commands for client and server.

## Mobile FFI in depth

### Android (JNI)

```rust
#[cfg(target_os = "android")]
use jni::{JavaVM, JNIEnv, LocalFrame};

fn show_native_dialog(message: &str) {
    #[cfg(target_os = "android")]
    unsafe {
        let env = /* get JNIEnv */;
        // JNI calls to Android native code
    }
}
```

### iOS (Obj-C)

```rust
#[cfg(target_os = "ios")]
use objc2::{class, msg_send, runtime::Object, DeclaredClass};

fn show_native_dialog(message: &str) {
    #[cfg(target_os = "ios")]
    unsafe {
        let alert = class!(UIAlertController);
        // Obj-C calls to iOS native code
    }
}
```

### Web (web-sys)

```rust
#[cfg(target = "wasm32-unknown-unknown")]
use web_sys;

fn show_native_dialog(message: &str) {
    #[cfg(target = "wasm32-unknown-unknown")]
    {
        web_sys::window().unwrap().alert_with_message(message).unwrap();
    }
}
```

### Platform-specific conditional compilation

```rust
#[component]
fn PlatformButton() -> Element {
    rsx! {
        #[cfg(target_os = "android")]
        button { "Android Button" }
        
        #[cfg(target_os = "ios")]
        button { "iOS Button" }
        
        #[cfg(target_os = "linux")]
        button { "Linux Button" }
        
        #[cfg(windows)]
        button { "Windows Button" }
    }
}
```

## Asset management

### asset!() macro

```rust
use dioxus::prelude::*;

// Load static assets (images, fonts, CSS)
const ICON: Asset<'/'> = asset!("/assets/icon.png");
const CSS: Asset<'/'> = asset!("/assets/main.css");

#[component]
fn App() -> Element {
    rsx! {
        img { src: ICON }
        link { rel: "stylesheet", href: CSS }
    }
}
```

### TailwindCSS integration

Dioxus auto-compiles `tailwind.css` at the project root during `dx serve` and `dx build`. The output is placed in the `assets/` folder.

### Asset resolution per platform

| Platform | Asset root |
|----------|-----------|
| macOS | Resources directory relative to executable |
| Linux | `lib/$product_name/assets/` |
| Android | Android AssetManager (Java bridge) |

## Remote Mac builds (detailed)

### GitHub Actions

```yaml
jobs:
  build-ios:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install dioxus-cli --locked
      - run: dx serve --platform ios --release
      - run: dx bundle --platform ios
```

### SSH to remote Mac

```bash
# On remote Mac:
rustup install stable
cargo install dioxus-cli --locked
cargo build --target aarch64-apple-ios --release
xcrun codesign --sign "Developer ID" --entitlements entitlements.xml MyApp.app
tar czf MyApp.tar.gz MyApp.app

# On local machine:
rsync -avz user@mac-mini:~/MyApp.tar.gz ./
tar xzf MyApp.tar.gz
```

### CI/CD pipeline

1. Build Rust code on remote Mac
2. Use `xcrun` for signing and provisioning
3. Package with `dx bundle`
4. Upload artifacts to App Store Connect or Firebase

### Mobile architecture internals

Dioxus mobile uses a webview-based renderer:

- **iOS:** WKWebView with custom protocol `dioxus://index.html/`
- **Android:** WebView via JNI with HTTPS-based scheme `https://dioxus.index.html/` (security requirement)
- **Communication:** Binary protocol + WebSocket edit streaming
- **Thread safety:** Android requires runtime lock (`android_runtime_lock()`)
- **No menus/tray/hotkeys** on mobile platforms (stubbed)

## Cross-platform feature matrix (full)

| Feature | Web | Desktop | Mobile (iOS/Android) | Server |
|---------|-----|---------|---------------------|--------|
| RSX components | ✅ | ✅ | ✅ | ✅ |
| Signals state | ✅ | ✅ | ✅ | ✅ |
| Hot-reloading | ✅ | ✅ | ⚠️ Limited | ❌ |
| Router | ✅ | ✅ | ✅ | ✅ |
| Server functions | ✅ | ✅ | ✅ | ✅ |
| Menus/Tray | ❌ | ✅ | ❌ (stubbed) | ❌ |
| Global hotkeys | ❌ | ✅ | ❌ | ❌ |
| File dialogs | ✅ | ✅ | ⚠️ Limited | ❌ |
| Multi-window | ❌ | ✅ | ❌ | ❌ |
| Native FFI | web-sys | Full | JNI/Obj-C | Full |

## Debugging deep dive

### Log levels

```bash
RUST_LOG=debug dx serve       # Verbose logging
RUST_LOG=info dx serve        # Default level
RUST_LOG=trace dx serve       # Maximum verbosity
```

### Hot-patch debugging

```bash
# Hot-patch mode (experimental)
dx serve --hotpatch

# Check hot-patch status
dx check
```

### Common error patterns

| Error | Cause | Fix |
|-------|-------|-----|
| Feature not found | Cargo.toml missing feature flag | Add `web = ["dioxus/web"]` etc. |
| Port 8080 in use | Another process on dev server port | Use `--port` or kill existing process |
| Android ABI mismatch | Wrong target architecture | Check `--target` matches device |
| iOS provisioning | Missing/invalid provisioning profile | Install proper profile in Xcode |
| Asset 404 | Wrong asset path or not bundled | Use `asset!()` macro for static assets |
| Tailwind not compiling | Missing `tailwind.css` at project root | Create empty `tailwind.css` at root |
| Fullstack build fails | Both `web` and `server` features missing | Enable both in Cargo.toml |

## Version compatibility

- **Dioxus 0.7.x** — Current stable (latest as of 2026)
- **Dioxus CLI** — Should match major.minor with dioxus crate version
- **Template resolution** — `dx new` auto-selects template branch matching CLI version

## Key resources

- **Docs:** https://dioxuslabs.com/learn/0.7/
- **API docs:** https://docs.rs/dioxus/latest/dioxus/
- **GitHub:** https://github.com/DioxusLabs/dioxus
- **Crates:** https://crates.io/crates/dioxus
- **CLI crates:** https://crates.io/crates/dioxus-cli
- **Discord:** https://discord.com/invite/dioxus
