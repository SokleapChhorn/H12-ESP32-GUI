# NPICTECH H12 — native TR remote app for the Skydroid H12

A native Android (Kotlin + Jetpack Compose) port of NPICTECH v2.3 for the Android screen on
the Skydroid H12. Same layout and features: field map with tap-to-target, sky-block buttons, two
sticks, MODE/SPEED toggles, D-pad jog with step size, face buttons, AUTO/MANUAL, RED/BLUE side
(double-tap to confirm), TR/BR (KTB) select, day/night theme and the robot pose and trail.

There is no WebView and no ESP32: the app talks to the robot through the H12's own radio.

```
App ──UDP 14551──► H12 radio ~~2.4 GHz~~► R12 receiver TX2 ──► STM32 UART RX
App ◄──────────────────────────────────── R12 receiver RX2 ◄── STM32 UART TX
```

## Two GUIs in one APK

Installing the APK adds **two launcher icons**. Both use the same radio link, protocol and robot firmware.

| Icon | GUI | Source |
|---|---|---|
| **NPICTECH H12** | Native Kotlin/Compose port | `MainActivity.kt`, `Screen.kt`, `Controls.kt`, `Field.kt`, `TrViewModel.kt` |
| **NPICTECH H12 Web** | Your v2.3 page in a WebView | `assets/index.html`, `WebActivity.kt`, `WebBridge.kt` |

The web version keeps the v2.3 page almost untouched. `WebBridge` registers as `TRBridge` with the
same `usbConnected()` / `sendLine()` methods the page already calls. Behind those methods it:

- adds the `*XX` checksum on the RC link;
- sends over UDP 14551 (or USB);
- turns the robot's compact `S…` status into the JSON that `window.trStatus()` expects, with rx,
  bad and Hz computed in the app.

Changes made to `index.html`, each marked `H12:` in the file:

- `buzz()` vibrates through the bridge, because the WebView ignores `navigator.vibrate`;
- the link label shows **RC** or **USB**, and both show green;
- holding the link label for 0.8 s switches between RC and USB;
- "no link to ESP" messages now read "no link to robot".

To change the web GUI, edit `app/src/main/assets/index.html` and rebuild. You can still test the
page in a desktop browser, where it falls back to its Wi-Fi fetch path.

The two GUIs keep separate settings for side, theme and toggles. Only one of them can use port
14551 at a time, so close one before opening the other.

## ESP32 relay mode ("ESP+RC"): keep the ESP32, lose its UART wires

The H12 radio can only be fed from Android (UDP 14551), so the ESP32 can't push its UART straight
into the radio. In this mode the app relays the ESP32's bytes instead:

```
App ──USB lines──► ESP32 (TR_GUI_BRIDGE, unchanged logic) ──"@R<hex>" over USB──► App ──UDP──► H12 radio ──► R12 TX2 ──► STM32
STM32 ──► R12 RX2 ──► H12 radio ──UDP──► App ──"@T<hex>" over USB──► ESP32 telemetry parser ──JSON status──► App page
```

**The STM32 firmware doesn't change.** It receives exactly the bytes the ESP32's UART used to send.

- **ESP32:** add `esp32/RadioPort.h` and use `RadioPort` in place of the UART object that talked
  to the STM32. Feed `@T` lines to `RadioPort.feedHex()` and call `RadioPort.poll()` every loop.
  The comments at the top of the file show the exact lines to change.
- **App:** hold the link label to cycle through **RC → USB → ESP+RC**.
- **Wiring:** connect the STM32 UART to the receiver's TX2/RX2 (crossed) instead of the ESP32.
  The receiver's data-port baud rate must match the STM32 UART.
- The STM32 must keep sending telemetry regularly, because the app learns the radio link's address
  from it.

## 1. Build the APK

1. Install Android Studio. Use **File → Open** on this folder, not "Import".
2. Wait for Gradle sync. It downloads Gradle 8.7, AGP 8.5 and the libraries, so the first sync
   takes a while and needs internet.
3. Use **Build → Build App Bundle(s) / APK(s) → Build APK(s)**. The APK is written to
   `app/build/outputs/apk/debug/app-debug.apk`.

The H12 runs an older Android, so `minSdk` is set to 21. Check the version under Settings → About
on the remote if an install fails.

## 2. Install it on the H12

You can install it in either of two ways:

- **USB debugging:** on the H12, go to Settings → About and tap the build number 7 times. Then turn
  on Developer options → USB debugging, connect the H12's Type-C to your PC, and press **Run** in
  Android Studio.
- **Copy the APK:** copy the APK file to the H12 and open it with the file manager.

> If the PPM-jack USB-TTL converter is still wired onto the Type-C D+/D- lines, it shares
> those lines with the PC. Disconnect it while installing or debugging.

## 3. Wire the robot

| R12 receiver (data port) | STM32H723 |
|---|---|
| TX2 | UART RX |
| RX2 | UART TX |
| GND | GND |
| 5V | 5 V, only if the receiver isn't powered from another connector |

- The receiver LED must be **solid green**. Solid red means the port is in C.Bus mode.
- The UART baud rate must match the receiver's data port. Find it with a USB-TTL adapter first;
  57600 is a common default. Changing it needs the password from Skydroid.
- Check that TX2 idles at about 3.3 V before connecting it to the STM32.

## 4. Robot firmware

Add `stm32/h12_link.c` and `stm32/h12_link.h`, then follow `stm32/example_main_snippet.c`:

- `H12Link_Init(&huartX)` once at start-up.
- `H12Link_Task()` from the main loop.
- `H12Link_RxEvent()` and `H12Link_Error()` from your HAL callbacks.
- Implement `H12Link_OnPad()` and `H12Link_OnCmd()`. They are where the ESP32 bridge's data used
  to arrive, so connect them to your existing `tr_pad` and command code.
- Send `H12Link_SendStatus(...)` at 5–10 Hz. **This is required:** the app learns the radio
  link's address from these packets, and it doesn't transmit until the first one arrives.
- Stop the chassis when `H12Link_Alive()` returns 0.

H7 DMA note: the receive buffer must be in AXI SRAM or D2 SRAM, not DTCM. Set `H12_DMA_ATTR` if
your linker places `.bss` in DTCM. This applies to Keil's default scatter file in particular.

`h12_link.c` was tested on a PC with a stubbed HAL: pad lines, commands, checksum rejection, the
link timeout, and a status line whose checksum matches the app's.

## 5. Protocol

The commands are identical to v2.3. The differences are that every line gets a `*XX` checksum
(XOR of all bytes before `*`) and that a compact status line replaces the ESP32's JSON.

| Direction | Line | Meaning |
|---|---|---|
| app → robot | `J<lx>,<ly>,<rx>,<ry>,<l2>,<r2>,<btn>,<flags>*XX` | 20 Hz. Sticks −100..100; l2/r2 0/50/100; btn = tr_pad.h bits (△0 ○1 ✕2 □3 ↑4 ↓5 ←6 →7 ↺8 ↻9); flags: bit0 AUTO, bit1 BR, bits2–3 SPEED, bits4–5 MODE |
| app → robot | `C<code>,<seq>*XX` | 1–8 targets, 9–11 sky targets, `0x40\|dir<<2\|step` jog, 112/113 side red/blue, 127 STOP |
| app → robot | `P*XX` | Heartbeat every 200 ms |
| robot → app | `S<fl>,<tgt>,<x>,<y>,<yaw×10>*XX` | fl: bits0–3 state, bit4 adjust, bit5 warn, bit7 side; x/y in mm in the MAP frame |

Retries work as in v2.3. A target command is re-sent until the status shows `tgt` equal to that
target (up to 4 tries), and a side change is re-sent until bit 7 of `fl` matches.

## 6. Link modes

A **long-press on the link label** (top left of the field) switches between:

- **RC** (default): the H12 radio over UDP 14551, with checksums.
- **USB**: USB serial with no checksum. Use it with the old ESP32-C3 bridge (it understands the
  same lines and its JSON status is still parsed), or with the PPM-jack converter.

Settings such as the port, baud rate and checksum are in `app/src/main/java/com/npic/trh12/Config.kt`.

## 7. First test, step by step

1. Flash the robot with `h12_link`, so that it sends status even when the chassis is off.
2. Open the app. The label should turn green and show **● RC**, and the pose should appear on the field.
3. If it stays **● no link (RC)**, the app isn't receiving status. Check the wiring, the baud rate
   and the receiver LED, and make sure no other app (H12Tool, QGC, a UDP terminal) is holding
   port 14551.
4. Move a stick and check that `H12Link_OnPad` gets the values. Then tap a target in AUTO.
5. Measure the joystick lag on the real field before relying on it for driving.

## Files

```
app/src/main/java/com/npic/trh12/
  MainActivity.kt   full-screen landscape, keeps the screen on
  Screen.kt         layout: left controls | field | right controls
  Controls.kt       multi-touch hold buttons, sticks (dead zone + 12° axis snap), 3-state toggles
  Field.kt          field geometry and drawing, targets (must match tr_routes.h)
  TrViewModel.kt    all logic: sending, retries, side handling, jog repeat, status
  Links.kt          UdpLink (H12 radio) and UsbLink (usb-serial-for-android)
  Protocol.kt       line format, checksum, status parsing
  Config.kt         port, baud rate, checksum switch
  WebActivity.kt    WebView host for assets/index.html
  WebBridge.kt      TRBridge for the page: RC/USB link, checksum, status -> JSON
app/src/main/assets/index.html   v2.3 web GUI (small H12 patches)
stm32/              h12_link.c/.h and an integration example (direct RC mode)
esp32/RadioPort.h   drop-in for the ESP32 bridge (ESP+RC relay mode)
```
