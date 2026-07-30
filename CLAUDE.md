# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Scope

This is a Konnected fork of `bradsjm/hubitat-public` (a personal collection of Hubitat Groovy drivers and apps, upstream README marks it "LEGACY — NO LONGER ACTIVELY MAINTAINED").

**The only file that matters here is `ESPHome/ESPHome-API-Library.groovy`** — the Hubitat library that implements ESPHome's native protobuf API. It is what lets Konnected's ESPHome-based Alarm Panels and Garage Door Openers talk to Hubitat. Everything else (`Aqara/`, `Component/`, `LedMiniDashboard/`, `PhilipsHue/`, `ThirdReality/`, `Tuya/`, and the example drivers in `ESPHome/`) is inherited fork content — don't modify it unless explicitly asked.

`ESPHome/ESPHome-API-Library-Bundle.zip` is a generated artifact of that same file (see Bundle below).

## Build / test / lint

There is none. No Gradle, no CodeNarc config, no test suite in the repo — Groovy here is source-only, executed by the Hubitat hub's sandboxed runtime.

The only automation is `.github/workflows/update-bundle.yml`: on any push touching `ESPHome/ESPHome-API-Library.groovy`, CI copies it to `esphome.espHomeApiHelper.groovy`, `zip -u`s it into `ESPHome-API-Library-Bundle.zip`, and auto-commits the zip. **Never hand-edit the zip's contents** — edit the `.groovy` and either let CI regenerate the zip on push or run the same three commands locally. The entry name inside the zip must stay `esphome.espHomeApiHelper.groovy` (`<namespace>.<library name>`) or Hubitat's bundle importer rejects it.

Verifying a change means installing it on a real hub: paste the library into Hubitat's **Libraries** section (not Drivers/Apps), install a driver that `#include`s it, point the device at a real ESPHome node, and watch the hub logs. Enabling the driver's `logEnable` preference turns on per-message trace logging in the library and raises the ESPHome device's own subscribed log level to DEBUG.

## Library architecture

The library is a single ~1670-line Groovy file: a hand-rolled protobuf codec plus a socket state machine, compiled into every driver that declares `#include esphome.espHomeApiHelper` as its last line.

**Connection lifecycle** — `openSocket()` → raw TCP to port 6053 → `espHomeHelloRequest` → `espHomeHelloResponse` (records `state.apiVersionMajor/Minor`) → `espHomeConnectRequest` → `espHomeDeviceInfoRequest/Response` → either `espHomeListEntitiesRequest` (first connect, or when MAC/compile-time changed) or straight to `espHomeSubscribe()`. Reconnect is exponential with jitter, capped at `MAX_RECONNECT_SECONDS`; `healthCheck` pings every 60s + jitter. `closeSocket(reason)` tears everything down.

Note the function names still say `Connect` (`espHomeConnectRequest`/`espHomeConnectResponse`) while the message constants say `AUTHENTICATION` — ESPHome renamed `ConnectRequest`→`AuthenticationRequest` in the proto at 2025.10 with the wire numbers (3/4) unchanged, and upstream renamed only the constants, keeping `MSG_CONNECT_REQUEST`/`MSG_CONNECT_RESPONSE` as aliases so existing drivers still compile.

**Two `parse` methods, different directions.** `parse(String hexString)` is called *by Hubitat* with raw socket bytes — the name is fixed by the platform, as is `socketStatus(String)`; renaming either silently breaks every driver. It reassembles frames (partial packets are stashed in a per-device `ByteArrayOutputStream`), then `parseMessage` decodes and switches on message type. `parse(Map message)` is implemented *by the driver* and is how the library pushes events out. Message maps always carry a `type` of `device`, `entity`, `state`, `service`, or `complete`, plus a `platform` (`binary`, `switch`, `cover`, `sensor`, `light`, `network`, …).

**Send queue and supervision.** Two `sendMessage` overloads: the 2-arg form is fire-and-forget; the 4-arg form (`msgType, tags, expectedMsgType, onSuccess`) enqueues into a per-device `ConcurrentLinkedQueue`, dedupes by `msgType`, retries every 5s up to 5 times, and gives up by closing the socket and reconnecting. `supervisionCheck` matches an inbound message against the queue, fires the `onSuccess` callback by name via `"${e}"(tags)` (so those private methods look unused — hence the `groovylint-disable UnusedPrivateMethod` comments), and returns a boolean. That boolean is threaded into state parsers as `isDigital`: a state response that satisfied a queued request came from *our* command (digital), an unsolicited one came from the device (physical).

**Shared static state.** `espReceiveBuffer` and `espSendQueue` are `@Field static final ConcurrentHashMap`s — one library instance is shared across *all* devices using it, so every entry is keyed by `device.id`. Never add unkeyed mutable static state.

**`@CompileStatic` everywhere it can be.** Methods touching only parameters/statics are annotated for speed; methods touching `device`, `settings`, `state`, `log`, `runIn`, or `sendEvent` cannot be and are left dynamic. `logWarning()` exists solely so `@CompileStatic` methods can log.

**Protobuf codec.** `protobufDecode` returns `Map<Integer, List>` — field number → list of values (repeated fields stack). Read them with the `getXxxTag(tags, index, default)` helpers, never directly. `protobufEncode` takes `[fieldNumber: [value, WIRETYPE_*]]` and **skips entries whose value is falsy**, which is why optional fields are encoded as a `has_x` varint flag plus the value (e.g. `4: [tags.position != null ? 1 : 0, VARINT], 5: [tags.position as Float, FIXED32]`).

### Adding support for a new ESPHome message

1. Add `MSG_*` constants at the bottom of the file (numbers come from [`api.proto`](https://github.com/esphome/aioesphomeapi/blob/main/aioesphomeapi/api.proto)).
2. Add a `private static Map espHomeXxx...(Map<Integer, List> tags)` decoder returning a map with `type`/`platform`/`key`; entity list responses start from `parseEntity(tags, <device_id field number>) + [...]`.
3. Add a `case` in `parseMessage`'s switch — `parse espHomeXxx(tags)` to forward to the driver.
4. For outbound, add a public `espHomeXxxCommand(Map tags)` using `sendMessage(...)` with the matching state response as `expectedMsgType` when the device acks with state.

## Driver contract

A driver using this library must:

- End with `#include esphome.espHomeApiHelper`
- Declare `attribute 'networkStatus', 'enum', ['connecting', 'online', 'offline']` — the library both writes it and reads it back via `isOffline()`, so omitting it breaks the send path
- Define preferences named exactly `ipAddress` (required), `password` (optional), `logEnable` (optional bool) — the library reads `settings.*` directly
- Call `openSocket()` from `initialize()` and `closeSocket(reason)` from `uninstalled()`
- Implement `void parse(Map message)`
- Set `singleThreaded: true` in `metadata.definition` (all 19 bundled drivers do)

The library owns these `state` keys: `reconnectDelay`, `requireRefresh`, `services`, `apiVersionMajor`, `apiVersionMinor`, `noiseDetected`. It also writes device data values (`MAC Address`, `Compile Time`, `ESPHome Version`, `Board Model`, …) and **overwrites `device.deviceNetworkId`** with the device's MAC on every DeviceInfo response.

## Known constraints

- **No encryption.** Frames starting with `0x01` (Noise) trigger `handleNoiseProtocolDetected()`, which logs an error with fix instructions, closes the socket, and throttles reconnects to `MAX_RECONNECT_SECONDS`. ESPHome YAML must use `api:` without an `encryption:` key.
- **No mDNS on Hubitat** — the IP address must be hardcoded (or DNS-resolvable).
- API major version > 1 is refused at handshake.
- `MSG_SUBSCRIBE_HA_STATES_REQUEST` and a few others are declared but marked `// TODO` and unimplemented. Climate *is* implemented as of v1.3.
- `PING_INTERVAL_SECONDS = 60`, deliberately. Hubitat's raw socket layer drops idle TCP connections at ~120s; the ping is scheduled at `interval + rand(interval/2)`, so anything above ~80 risks firing after the socket is already dead. Don't raise it.

## Fork state vs upstream

`origin` = `konnected-io/hubitat-public`, `upstream` = `bradsjm/hubitat-public`.

`main` is rebased onto upstream v1.3 (`API_HELPER_VERSION = '1.3'`, ESPHome 2026.2.0 support). The fork delta is now one commit, ~53 lines, each change marked with a `KONNECTED:` comment:

1. **`case MSG_AUTHENTICATION_RESPONSE` in `parseMessage`** → `espHomeUnsupervisedAuthenticationResponse()`. Upstream's `espHomeConnectRequest()` gates on the negotiated API version: API ≤ 1.11 waits for a reply, API ≥ 1.12 advances immediately without supervising. That gate is correct and must be kept — password auth was *removed* in ESPHome 2026.1.0, so those devices auto-authenticate on Hello and never reply. But a 2025.10–2025.12 device built with `USE_API_PASSWORD` still does reply, and without this case that reply is logged as an unhandled message type while an invalid password goes undetected. The handler deliberately does *not* call `espHomeConnectResponse()`, which would queue a duplicate `DeviceInfoRequest` on the fast path.

2. **Per-entity `device_id` field numbers.** Upstream shipped a single global `ENTITY_DEVICE_ID_PROTO_FIELD = 26` with a `*** VERIFY THIS VALUE ***` box in the source that never got verified. There is no global number — each `ListEntitiesXxxResponse` gives `device_id` its own next-free field (26 is correct only for climate), and it's a `uint32` varint, not a string. Replaced with a `parseEntity(tags, deviceIdField)` parameter read via `getIntTag`, defaulting to `0`.

3. **`protobufDecode()` generics restored to `Map<Integer, List>`.** Upstream reduced it to a raw `Map` while leaving the method `@CompileStatic`. That makes `tags.computeIfAbsent(tag){...}` return `Object`, so the following `.add(val)` is a static type-check error. Groovy 2.4 lets it through; Groovy 3+ rejects it and the *entire library* fails to compile, taking every driver with it. Don't re-strip these — see the `KONNECTED:` comment on the method.

Device handshake behavior, if you need to reason about it again:

| ESPHome | API | No password | Password set |
|---|---|---|---|
| ≤ 2024.12 | 1.10 | replies | replies |
| 2025.10–2025.12 | 1.12–1.13 | auto-auth on Hello, no reply | replies |
| 2026.1+ | 1.14 | auto-auth, password auth removed | n/a |

We still send `AuthenticationRequest` unconditionally; 2026.1+ has no `case 3:` in its dispatcher and the `default:` branch ignores it, so this is safe.

**Verify protocol claims against sources, not changelogs.** Upstream's v1.3 commit message is a long, confident changelog that is wrong in places (its climate field mapping was broken and had to be fixed by the *next* commit; FIX-9's stated premise was already true in the baseline; FIX-6's claim that API ≥ 1.12 never replies "even with a password set" is false for 2025.10–2025.12). Ground truth:

- `https://raw.githubusercontent.com/esphome/aioesphomeapi/main/aioesphomeapi/api.proto` — field numbers and wire types
- `https://raw.githubusercontent.com/esphome/esphome/<tag>/esphome/components/api/api_connection.cpp` — actual device behavior, per release tag
- `.../api/api_pb2_service.cpp` — which message types the device dispatches at all

### Checking a change without a hub

There's no Groovy toolchain installed, but two useful checks are cheap to set up. Fetch `groovy-2.4.21.jar` and `groovy-3.0.21.jar` from Maven Central and drive them with a small Java `CompilationUnit` harness. Run it under JDK 17 (`/opt/homebrew/opt/openjdk@17`) — Groovy 3's bundled ASM can't read the default JDK 26's class files.

1. **Parse check** — compile the whole file to `Phases.CONVERSION`. Catches syntax errors only. Drivers need their `#include` line stripped first; it's a Hubitat preprocessor directive, not Groovy.
2. **Static type check** — extract every `@CompileStatic private static` method whose body doesn't touch `device`/`settings`/`state`/`log`/`runIn`/`sendEvent`/`interfaces`, plus all the `@Field` constants, into one class and compile *that* to `Phases.CLASS_GENERATION` (stub `HexUtils`). This is what caught the `protobufDecode` regression above, and it covers the codec and every entity decoder — roughly 34 methods. Full compilation of the real file is impossible because Hubitat's injected symbols don't exist off-hub.

**Run the type check against both 2.4 and 3.x.** They disagree, and the disagreement is the interesting part: 2.4 is what Hubitat runs today, 3.x is the canary for anything a platform upgrade would break.

Note the library is plain LF. `file` reports it as "Unicode text ... with escape sequences" because of box-drawing characters in comments, not line endings — a raw `git diff` against upstream looks enormous due to reformatting, so use `--ignore-all-space` when comparing.

Note that `importUrl`, `ESPHome/packageManifest.json`, and `repository.json` all still point at `raw.githubusercontent.com/bradsjm/...`, not at this fork — so a Hubitat Package Manager install or a driver's "Import" button pulls upstream's code, not the code in this repo. Anything Konnected ships has to reference this fork's raw URLs explicitly.

## Style

Match the existing file: 4-space indent, single quotes for strings, no semicolons, Groovy-idiomatic `.with {}` blocks, `groovylint-disable-next-line` comments where CodeNarc would complain (upstream develops with IntelliJ + the CodeNarc plugin). Keep new helpers `@CompileStatic` unless they need `device`/`settings`/`state`.
