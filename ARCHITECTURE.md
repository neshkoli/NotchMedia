# NotchMedia architecture

This repository is a fork of [TheBoredTeam/boring.notch](https://github.com/TheBoredTeam/boring.notch), rebranded as **NotchMedia** (bundle id `media.notch.NotchMedia`). The upstream project is a macOS menu-bar app that draws a live “notch” experience (music, shelf, calendar, HUDs, and more) using SwiftUI and AppKit.

Use this document as a map when you add features or refactor. Class names still use the `Boring*` prefix in many places from upstream; targets and product names use **NotchMedia**.

---

## High-level layout

| Area | Path | Role |
|------|------|------|
| Main app | `NotchMedia/` | SwiftUI + AppKit UI, coordinators, managers, media controllers |
| XPC helper | `NotchMediaXPCHelper/` | Privileged operations (accessibility checks, brightness, keyboard backlight) isolated from the main process |
| Xcode project | `NotchMedia.xcodeproj/` | Targets `NotchMedia` (app) and `NotchMediaXPCHelper` (XPC service) |
| Now Playing bridge | `mediaremote-adapter/` | Ships **MediaRemoteAdapter.framework** and a small Perl helper; used for system “Now Playing” on newer macOS versions |
| Sparkle / releases | `updater/` | `appcast.xml` and release metadata (must be regenerated with your own signing keys if you ship updates) |
| DMG tooling | `Configuration/dmg/` | Scripts and settings to build a distributable disk image |

---

## Build targets and runtime

- **NotchMedia** — macOS application; product `NotchMedia.app`. Entry: `@main struct NotchMediaApp` in `NotchMedia/NotchMediaApp.swift`, which hosts a `MenuBarExtra` and delegates lifecycle to `AppDelegate`.
- **NotchMediaXPCHelper** — XPC service embedded under `Contents/XPCServices/`. Bundle id `media.notch.NotchMedia.NotchMediaXPCHelper`. The app connects using the Mach service name `media.notch.NotchMedia.NotchMediaXPCHelper` (see `NotchMedia/XPCHelperClient/XPCHelperClient.swift`).

Changing XPC behavior requires updating **both** the protocol in `NotchMedia/XPCHelperClient/NotchMediaXPCHelperProtocol.swift` (client copy) and `NotchMediaXPCHelper/NotchMediaXPCHelperProtocol.swift` / implementation, keeping `@objc` selector compatibility in mind.

---

## Application flow

1. **Launch** — `NotchMediaApp` creates `SPUStandardUpdaterController` (Sparkle) and wires `SettingsWindowController` to it. `AppDelegate` sets up status items, screen observers, and **per-screen notch windows**.
2. **Notch windows** — `AppDelegate` creates `NotchMediaSkyLightWindow` (subclass of `NSPanel`) per display via `createNotchMediaWindow`. Content is SwiftUI driven by `BoringViewModel` / `ContentView` / tab views under `NotchMedia/components/`.
3. **Coordinator** — `BoringViewCoordinator` (`NotchMedia/BoringViewCoordinator.swift`) is the central `@MainActor` observable: current notch tab (`NotchViews`), sneak-peek HUD state, onboarding flags, and related UI mode. Most feature UI reads or writes through this type or `Defaults` keys from **SwiftyUserDefaults** / `@AppStorage`.
4. **Space layering** — `NotchSpaceManager` uses private CoreGraphics space APIs (`NotchMedia/private/CGSSpace.swift`) to keep the notch layer above normal windows.

---

## Major feature modules

### Music and “Now Playing”

- **`MusicManager`** (`NotchMedia/managers/MusicManager.swift`) — Singleton façade: artwork, playback state, transport commands, lyrics. It selects a **media controller** conforming to `MediaControllerProtocol` (`NotchMedia/MediaControllers/`): Apple Music, Spotify, YouTube Music, or `NowPlayingController` for system Now Playing (with MediaRemoteAdapter on supported OS builds).
- **Adding a new music source** — Implement `MediaControllerProtocol`, register it in `MusicManager`’s controller selection logic, and expose it in onboarding/settings UI under `NotchMedia/components/Onboarding/` and `Settings/`.

### Shelf (files, AirDrop, Quick Look)

- **Views** — `NotchMedia/components/Shelf/Views/`.
- **State** — `ShelfStateViewModel`, selection models, persistence under `Shelf/Services/` (e.g. `ShelfPersistenceService` uses Application Support under a `NotchMedia/Shelf` path segment).

### Calendar

- **Managers** — `CalendarManager.swift`; models in `NotchMedia/models/CalendarModel.swift` / `EventModel.swift`; UI in `components/Calendar/BoringCalendar.swift`.

### HUD / volume / brightness / backlight

- **Managers** — `VolumeManager`, `BrightnessManager`, and related observers (`MediaKeyInterceptor`, fullscreen detection). Sneak-peek UI is coordinated through `BoringViewCoordinator.toggleSneakPeek`.
- **Keyboard backlight** — Often delegated to **NotchMediaXPCHelper** over XPC because of entitlement / TCC boundaries.

### Settings and onboarding

- **`SettingsWindowController`** / **`SettingsView`** — AppKit window hosting SwiftUI settings.
- **Onboarding** — `components/Onboarding/*`; permissions and controller choice live here.

---

## Dependencies (Swift Package Manager)

Declared in the Xcode project; resolved versions live in `NotchMedia.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved`. Notable packages include **Sparkle** (updates), **Defaults**, **KeyboardShortcuts**, **Lottie**, **SwiftUIIntrospect**, **AsyncXPCConnection**, **MacroVisionKit**, **SkyLightWindow**, **LaunchAtLogin**, and **Collections**.

---

## Configuration you will likely change for your fork

| Concern | Where |
|--------|--------|
| Bundle identifier | Xcode → target **NotchMedia** → Signing, or `PRODUCT_BUNDLE_IDENTIFIER` in `NotchMedia.xcodeproj/project.pbxproj` |
| Display name | `INFOPLIST_KEY_CFBundleDisplayName` in the project file (currently **NotchMedia**) |
| Sparkle feed URL | `NotchMedia/Info.plist` → `SUFeedURL` (points at this repo’s `updater/appcast.xml` via raw GitHub URL). You must publish a matching **EdDSA** public key (`SUPublicEDKey`) and sign items in `appcast.xml` when you ship your own builds. |
| OAuth / server paths | YouTube Music client still calls `…/auth/boringNotch` on the upstream server path; do not rename unless you control that backend. |
| CI scheme | `.github/workflows/cicd.yml` — builds scheme **NotchMedia** |
| Crowdin | `crowdin.yml` — paths under `/NotchMedia/` |

---

## Tests

There is **no XCTest target** in the project today. Validation is done by **building** (Debug/Release) in Xcode or CI. Adding a small unit-test target for pure Swift logic (e.g. models or networking parsers) would be a natural first improvement.

---

## Syncing with upstream

```bash
git fetch upstream
git merge upstream/main   # or rebase, per your workflow
```

Remote `upstream` should reference `https://github.com/TheBoredTeam/boring.notch.git`. Resolve conflicts especially in `project.pbxproj`, `Package.resolved`, and any files you heavily customized.

---

## Mental model for new features

1. Decide whether the feature is **global UI state** (prefer extending `BoringViewCoordinator` + `Defaults` keys), **per-window** (thread through `BoringViewModel` / environment), or **privileged** (extend XPC protocol + helper implementation).
2. Add SwiftUI in `NotchMedia/components/` following existing folder patterns (`Notch/`, `Music/`, `Shelf/`, etc.).
3. If you need new entitlements, edit `NotchMedia/NotchMedia.entitlements` and mirror any sandbox or XPC assumptions in the helper target.

This should be enough context to navigate the codebase without reading every file.
