# Kukirin Manager (Community, Unofficial)

An unofficial SwiftUI + CoreBluetooth iOS app styled after the Kukirin Manager
app: scan for scooters, connect over Bluetooth LE, see live speed/battery/
odometer, switch ride modes, and set the max speed for each mode.

**Target: iOS 16.0+ (tested config for 16.7.x).**

## What actually works right now

- Full BLE scanning + connect/disconnect flow (`Sources/BLE/BLEManager.swift`)
- The entire UI: scanner list, speed gauge, mode selector, battery/odometer
  tiles, lock/lights/cruise toggles, per-mode max-speed sliders
- **Demo Mode** (toggle in the top-right of the scanner screen) — simulates a
  connected scooter with live-changing speed/battery so you can see the full
  app working immediately, including in the iOS Simulator (CoreBluetooth
  doesn't work in the Simulator at all, so Demo Mode auto-enables there)
- GitHub Actions workflow that builds an unsigned `.ipa` automatically

## What you need to fill in yourself: `Sources/BLE/ScooterProtocol.swift`

I could not include Kukirin's real Bluetooth protocol (service/characteristic
UUIDs and command byte format) because it isn't publicly documented and I
have no way to verify it — shipping guessed bytes would just silently fail to
control your scooter, which is worse than being upfront about it.

That file has detailed step-by-step instructions at the top, but in short:

1. Install [Wireshark](https://www.wireshark.org).
2. Capture Bluetooth traffic while using the **real** Kukirin Manager app to
   connect and tap through mode changes / speed limit changes / lights /
   lock. Two common ways:
   - **Android**: Settings → Developer Options → "Enable Bluetooth HCI
     snoop log", reproduce the actions, then pull
     `/sdcard/android/data/btsnoop_hci.log` and open it in Wireshark.
   - **iOS + Mac**: Xcode → Open Developer Tool → "Additional Tools for
     Xcode" → download **PacketLogger**, capture while using the real app
     on a connected iPhone.
3. Filter the capture on `btatt` to see only GATT read/write/notify packets.
4. Note the service UUID, the characteristic UUID you write commands to, the
   characteristic UUID that notifies live telemetry, and the exact bytes for
   each action.
5. Paste those into the constants and `encode`/`decode` functions in
   `ScooterProtocol.swift`. The rest of the app (UI, connection flow) doesn't
   need to change at all.

Once that file reflects the real protocol, turn off Demo Mode and the exact
same screens will control your real scooter.

## Building the IPA

This repo builds itself — just push it to GitHub:

1. Create a new GitHub repository and push this entire folder to it.
2. GitHub Actions will automatically run `.github/workflows/build-ipa.yml`
   on every push to `main`/`master` (or trigger it manually from the
   **Actions** tab → "Build Unsigned IPA" → **Run workflow**).
3. When it finishes, open the workflow run → **Artifacts** →
   `KukirinManager-unsigned-ipa` → download it. That's your `.ipa`.

The workflow uses [XcodeGen](https://github.com/yonaskolb/XcodeGen) to
generate the `.xcodeproj` on the fly (so there's no fragile, hand-edited
project file to go stale) and builds with code signing turned off entirely.

## Important: installing an *unsigned* IPA on iOS 16.7

Apple does not allow a truly unsigned app to be installed on a normal
(non-jailbroken) iPhone — iOS checks for a valid signature at install time,
not just at build time. This is an OS-level restriction, not something any
Xcode project setting can get around. To actually get this onto your phone
you'll need one extra step after downloading the IPA:

- **[Sideloadly](https://sideloadly.io/) / [AltStore](https://altstore.io/)**
  (easiest, free) — re-signs the IPA with your own free Apple ID and
  installs it. Apps installed this way need re-signing roughly every 7 days
  (free account) unless you have a paid Apple Developer account (1 year).
- **TrollStore** — if your specific iOS 16.7.x build is exploitable by
  TrollStore, it can install genuinely unsigned/permanently-signed apps with
  no re-signing needed. Check https://trollstore.app for supported versions.
- A jailbroken device can install the unsigned IPA directly (e.g. via
  Filza), no re-signing needed.

## Project structure

```
KukirinManager/
├── project.yml                  # XcodeGen spec (generates the .xcodeproj)
├── Info.plist
├── Sources/
│   ├── App/KukirinManagerApp.swift
│   ├── Models/Scooter.swift
│   ├── BLE/BLEManager.swift          # CoreBluetooth scan/connect/read/write
│   ├── BLE/ScooterProtocol.swift     # ⚠️ fill in real protocol here
│   ├── Views/RootView.swift          # scanner screen
│   ├── Views/ScooterDetailView.swift # main control screen
│   ├── Views/SpeedGaugeView.swift
│   ├── Views/ModeSelectorView.swift
│   └── Views/StatTile.swift
├── Resources/Assets.xcassets/
└── .github/workflows/build-ipa.yml
```

## Building locally in Xcode (optional)

If you have a Mac with Xcode installed:

```bash
brew install xcodegen
xcodegen generate
open KukirinManager.xcodeproj
```

Then just hit Run with a real device selected (turn code signing back on
in the project settings first if you want to run it directly from Xcode
onto your own device with your own Apple ID).
