# Bluetooth connections: a plan

A TECHO5 device already forwards the Bluetooth Low Energy advertisements it hears to Home Assistant
(`feature/bluetooth`), which is half of what an ESPHome Bluetooth proxy does: Home Assistant can see
a sensor broadcasting its temperature. The other half is **connections**: Home Assistant asks the
device to open a GATT connection to a device in range, discover its services, and read, write and
subscribe to characteristics. That half does not exist anywhere in this tree — not on the Show, not
on the Spot, not on the Dot — and it is what most of the interesting BLE devices need: SwitchBot
bots and curtains, stock-firmware Xiaomi/Mijia sensors, locks, scales, toothbrushes and anything
that has to be paired, written to or polled.

This is the plan for building it. It is written to be argued with: every choice below names what it
was chosen over, and why.

**Status (2026-10-17): a plan, nothing built.** Written from a read-only survey of the code at
`ba1b7bc`, and checked against ESPHome's own `components/bluetooth_proxy/bluetooth_proxy.h` and
BlueZ's GATT API documentation. Every claim about this tree carries a file reference. The protocol
half is checked against the messages this repository's ESPHome library already carries
(`go-esphome-device` at `v0.0.15-0.20261013225817-bf214844c4f5`), which is what Home Assistant
actually speaks to.

## What the device does today

Two halves of a proxy, one of them missing.

**Advertising works.** `feature/bluetooth` batches what the radio hears and sends it as
`BluetoothLERawAdvertisementsResponse` (`bluetooth.go`), drops rather than blocks when Home
Assistant is slow (`queued`, 512), and reports the scanner's state. The feature bitmask is:

```go
// feature/bluetooth/bluetooth.go:41
const bluetoothFeatures = esphome.BluetoothPassiveScan |
    esphome.BluetoothRawAdvertisements |
    esphome.BluetoothStateAndMode
```

`BluetoothActiveConnections` is not in it, so Home Assistant never offers this device for a
connection in the first place.

**Connections do not, and cannot as the code stands.** Three separate things stand in the way:

1. The library's proxy answers the one message that advertises capacity with an empty reply —
   `BluetoothConnectionsFreeResponse{}`, which reads as *no free connections* — and ignores every
   GATT message. Its own comment says why:
   `// Answered even though connections are not offered: silence leaves Home Assistant waiting.`
   That is a deliberate stub, not a bug.
2. On a Show the proxy goes through BlueZ (`hardware/ble/bluez.go`, `//go:build !dot`), because
   `bluetoothd` owns the controller for the earbuds and phone-audio features. That scanner rebuilds
   an advertisement out of the fields BlueZ parsed, and cannot advertise at all ("advertising is not
   available through bluez, skipping the beacon") — the same ownership that makes connections
   possible is what stopped advertising being offered.
3. On a Dot the scanner is the raw HCI radio (`hardware/ble/default_dot.go`), which is faster for
   scanning, but when `btbridge` has put the controller behind `hci0` it scans through a raw HCI
   socket (`hardware/ble/kernel.go`, `HCI_CHANNEL_RAW`). That channel can *send* `LE Create
   Connection`; what it cannot do is keep the resulting connection, because the kernel has no
   connection object for a connection it did not initiate and discards the LE Connection Complete
   event. Scan and advertise, yes; connect, no.

So the honest summary: **scanning is done on every board, connections are done on none, and the
constraint that shapes the design is that the Bluetooth stack which owns the controller is
`bluetoothd`.**

## What Home Assistant needs of a proxy

This is the contract, and it has to be met exactly or Home Assistant simply will not use the
device. From ESPHome's `bluetooth_proxy.h`:

```
// Version 1: Initial version without active connections
// Version 2: Support for active connections
// Version 3: New connection API
// Version 4: Pairing support
// Version 5: Cache clear support
static constexpr uint32_t LEGACY_ACTIVE_CONNECTIONS_VERSION = 5;
enum BluetoothProxyFeature : uint32_t {
  FEATURE_PASSIVE_SCAN = 1 << 0, FEATURE_ACTIVE_CONNECTIONS = 1 << 1,
  FEATURE_REMOTE_CACHING = 1 << 2, FEATURE_PAIRING = 1 << 3,
  FEATURE_CACHE_CLEARING = 1 << 4, FEATURE_RAW_ADVERTISEMENTS = 1 << 5,
  FEATURE_STATE_AND_MODE = 1 << 6, FEATURE_CONNECTION_PARAMS_SETTING = 1 << 7,
};
```

The flags travel in `DeviceInfoResponse.BluetoothProxyFeatureFlags`, which this daemon fills from
`bluetooth.Get().Features()` (`feature/api/api.go:211`). The five things that have to be true:

1. **The flag is set** — `FEATURE_ACTIVE_CONNECTIONS`, and `FEATURE_PAIRING` and
   `FEATURE_CACHE_CLEARING` if those are to work. A proxy that claims nothing new keeps behaving
   exactly as today.
2. **Capacity is reported honestly.** Home Assistant decides whether it may send a connection
   request to this proxy from `BluetoothConnectionsFreeResponse`, which carries `Free`, `Limit` and
   `Allocated` (the addresses currently held). It is sent when Home Assistant subscribes
   (`SubscribeBluetoothConnectionsFreeRequest`) and again whenever it changes. Replying zero means
   "use somebody else", which is what the stub reply does today.
3. **The GATT conversation is answered**, message for message. All of it is already in the API
   package this repository vendors: `BluetoothDeviceRequest` (connect, disconnect, pair, unpair,
   clear cache), `BluetoothSetConnectionParamsRequest`, `BluetoothGATTGetServicesRequest` /
   `...Response` / `...DoneResponse`, `BluetoothGATTReadRequest`, `BluetoothGATTWriteRequest`,
   `BluetoothGATTNotifyRequest`, `BluetoothGATTReadDescriptorRequest`,
   `BluetoothGATTWriteDescriptorRequest`, the matching responses, `BluetoothGATTNotifyDataResponse`
   for data pushed by the device, and `BluetoothGATTErrorResponse` for anything that fails.
4. **Failures are answered too.** Every request gets a response or an error; silence makes Home
   Assistant wait for a timeout and, worse, keep a connection slot marked busy.
5. **Attributes are addressable by handle.** Read, write and notify carry `uint32 Handle` plus the
   device address, not a UUID, so the device must present a stable handle for every characteristic
   and descriptor it discovers. This is the one place where the proxy's model and BlueZ's model do
   not line up, and it gets its own section below.

## The choice that shapes everything: who makes the connection

Four ways to get a GATT connection on this hardware. Everything else in the plan is a consequence
of this choice.

| | **A. BlueZ over D-Bus** | **B. Raw HCI takeover** | **C. Own the controller** | **D. Second controller** |
|---|---|---|---|---|
| Who connects | `bluetoothd` | our code, raw HCI | our code, kernel BT stack | a second virtual controller |
| Works for connections | yes | **no** — no kernel `hci_conn`, the event is dropped | yes | no (one radio) |
| Audio (earbuds, phone-as-source) keeps working | yes | yes, if we only ever scan | **no** — bluez-alsa and `bluetoothd` are the audio path | yes |
| Pairing and bonding | BlueZ, done | manual, kernel-managed, fragile | kernel + our agent | n/a |
| Effort | medium: a D-Bus GATT client and a translation layer | small, and pointless | very large: A2DP/AVRCP/HFP as well | n/a |
| Fidelity | service discovery and notification latency are D-Bus-bound | n/a | native | n/a |

**Decision: A, BlueZ over D-Bus.** The argument is not that BlueZ is nicer; it is that the three
things Home Assistant needs from a connection — the link itself, pairing and bonding, and the
attribute database — are exactly the three things `bluetoothd` already does on these devices, and
exactly the three things a hand-rolled HCI path would have to reimplement against a vendor
controller. Option B is ruled out by the kernel rather than by taste: an LE connection not
initiated by the kernel's own connection management has no `hci_conn` to attach to, which is why
the Dot's fast raw socket is fine for scanning and useless for this. Option C is the only one that
is technically better — no D-Bus in the path, no translation, full speed — and it is priced in
Bluetooth audio: `bluez-alsa` and `bluetoothd` are how the earbuds, the phone-as-source and AVRCP
features work at all, so choosing C means owning A2DP and HFP too, or making the device choose
between being a speaker and being a proxy. Option D is a non-starter: one radio, one firmware node
(`/dev/stpbt`), one stack user.

The fifth option is not on the device at all: **buy an ESP32 Bluetooth proxy**, which is what people
do today and what the project would recommend to anybody who only wants coverage in one room. It is
a good answer and it is why this feature is not urgent. It is not *this* answer because the radio,
the power supply and the position in the room are already bought and paid for in every TECHO5
device, because the presence work in `multiroom-followme-plan.md` already leans on these devices as
proxies, and because a device that is already in the room is the cheapest proxy anyone will ever
install. Anyone who disagrees with that arithmetic should stop here; nothing else in the tree
depends on this plan.

**All three boards take the same choice.** `bluetoothd` is running on every one of them (the audio
features need it), so the GATT client is written once, in `internal/lib/bluez`, and only the
*scanning* path stays per-board, which is how the code is already arranged
(`hardware/ble/default_dot.go` versus `bluez.go`, with a shared `Scanner` interface in
`scanner.go`). The Dot keeps its faster raw-HCI advertisements; connections are BlueZ's on every
board. This also means the Show is not a special case to be fixed — it is where the two paths are
already most different, and where the work will land first because the real hardware is on the
bench.

## Where the translation lives

Three places the ESPHome-message-to-D-Bus mapping could go:

1. **In `feature/bluetooth`, in this repository.** The feature already owns the protocol
   conversation: its `Handle` is in the daemon's handler chain
   (`feature/api/api.go:218`, `esphome.Chain(Describers…, entities…, Handlers…)`, and `Chain` runs
   *every* handler, not the first match), so it sees every message with the connection in hand and
   can answer or delegate one message at a time.
2. **In a fork of `go-esphome-device`.** The library is the natural home for "the ESPHome side of
   the conversation", and its `BluetoothProxy` already owns subscribe/set-mode.
3. **Upstreamed to `go-esphome-device`** as a real GATT client, which every Go ESPHome device would
   get.

**Decision: 1, with 3 as a follow-up.** The library's own doc comment sets the boundary — "carries
what the radio found and never touches hardware" — and everything hard about this feature is
hardware: BlueZ, slots, the shared antenna, pairing agents. Keeping it in the daemon also keeps the
release cycle in one repository and avoids carrying a fork of a dependency. The one wrinkle is the
stub: the library answers `SubscribeBluetoothConnectionsFreeRequest` with an empty response, and
this feature must answer it with the real one. That is easy here — the delegation point is our own
`Proxy.Handle`, so the new messages are handled before the library is asked, and the library keeps
the three it is good at (subscribe, unsubscribe, set-mode). The honest follow-up is a small upstream
patch that stops the library from answering that message at all, so a future version cannot race
ours.

## Handles: the one thing BlueZ will not give us

Home Assistant addresses attributes by numeric handle (`BluetoothGATTReadRequest.Handle`,
`.WriteRequest.Handle`, `.NotifyRequest.Handle` — all `uint32`), and BlueZ's GATT client objects are
keyed by UUID and object path. Its `Handle` property does not close the gap: BlueZ documents it as
`uint16 Handle [read-write, optional] (Server Only)`, so on a client connection — which is what we
are — it may simply not be there.

So the device must **assign handles itself, once per connection**, and keep the mapping for as long
as the connection lives:

- enumerate the services, characteristics and descriptors through `GetManagedObjects` on the
  device's path (the same call `bluez.Adapter.load` already uses for devices);
- walk the tree in a fixed order (service object path, then characteristic object path or index)
  and number what is found, skipping handles that would collide with the reserved 0 and 1;
- keep `handle → object path + interface` in memory for the connection, and drop it when the
  connection does.

Two alternatives were considered and are worse. **Using BlueZ's `Handle` when it is present and our
numbering otherwise** is the tempting middle; this plan allows it only as a hint inside one map,
never as two numbering schemes in the same tree, because a characteristic numbered by BlueZ and a
descriptor numbered by us would not agree on "next free" and Home Assistant caches the tree for the
connection's life. **Reconnecting to re-discover on every read** would make handles meaningless
(BlueZ rebuilds the tree) and cost a round trip per attribute.
The stability requirement is the whole reason the map is per-connection and not per-device: after a
reconnect the numbers may differ, and that is fine as long as Home Assistant is told the new tree
before it reads anything.

The UUID encoding has a wrinkle of its own: `BluetoothGATTCharacteristic` carries `Uuid []uint64`
(two words, little-endian) and, for short UUIDs, `ShortUuid uint32`. The Bluetooth base UUID's
16-bit form has to be recognized and sent as a short UUID, the same way the Show's advertisement
rebuild already does it (`uuid16` in `hardware/ble/bluez.go`). Units and endianness are worth a
table-driven test on their own; getting it wrong produces a device whose service list looks
plausible and whose reads all fail.

## Slots, limits and the radio that is already busy

How many connections to offer is a policy decision with a physical answer behind it.

- **The antenna is shared.** Wi-Fi and Bluetooth are on one radio, and this repository has measured
  the cost twice: scanning was given a duty cycle of 18.75 ms in every 200 ms precisely so Wi-Fi
  keeps the antenna (`scanInterval` / `scanWindow` in `hardware/ble/ble.go`), and an idle A2DP
  stream was measured dropping the Wi-Fi link from 72 Mbit/s to 6.5 Mbit/s (porting-plan.md, "A2DP
  and Wi-Fi share the antenna"). A live GATT connection is a *scheduled* link, which is a
  different and heavier load than a scan.
- **Audio already loses to the proxy today.** `docs/setup.md:243` carries a user-visible warning:
  "Turn the Bluetooth proxy off on a device a phone plays to: the proxy keeps scanning on the same
  radio, and the phone's sound stutters or drops while it does (#105)." Connections make that worse,
  not better.
- **BlueZ has its own ceiling.** BlueZ accepts only a handful of simultaneous LE connections, and
  each one is a `bluetoothd` client.

**Decision: offer three, refuse the fourth, and refuse while audio is streaming.** Three because it
is enough for the use that justifies the feature (a lock, a sensor, a bot in one room) and small
enough that the failure mode of a busy radio is a "no" Home Assistant can route elsewhere rather
than a stutter someone has to diagnose. A larger number costs nothing to *declare* and everything
to *live with*: an honest `Limit` is what makes Home Assistant spread devices across proxies instead
of piling them onto one.

The refusal is the interesting half. Two policies were possible for audio: **refuse new
connections while something streams** (earbuds connected or a phone playing), or **allow them and
accept the stutter**. The plan takes the first: `feature/btaudio` already knows whether a sink is
live (`speaker.Get().SinkSounding()` is used to reopen audio in `btaudio.go`), so the check is a
call away, and refusing is what Home Assistant's slot model is for — it will try another proxy, or
try again later. What it must *not* do is silently drop an established connection when a phone
starts playing: Home Assistant holds that connection for a lock or a command, and pulling it out
from under a write is worse than a little stutter. So: refuse new ones, keep existing ones, and
report the capacity change so Home Assistant stops choosing this proxy while the radio is busy.

Three smaller policies come with it:

- **Scanning continues while connected**, at the existing duty cycle. Stopping it would blind the
  presence features that need advertisements; raising it would starve Wi-Fi. If the bench says a
  connection plus the scan is too much, the knob is the scan duty cycle, not the connection.
- **A connection that goes away is reported**, not mourned: `BluetoothDeviceConnectionResponse` is
  the message Home Assistant expects when a link drops (the remote went to sleep, walked out of
  range, or `bluetoothd` lost it), and it is what frees the slot.
- **A slot held by a wedged peer is released by BlueZ**, not by us: its own supervision timeout
  ends the connection and fires the same response. We do not time out connections ourselves; a
  proxy that drops a slow device's link on its own schedule is a proxy that breaks pairing.

## Pairing, bonding and cache

`FEATURE_PAIRING` and `FEATURE_CACHE_CLEARING` are the two flags that come free with BlueZ, and the
two that carry the most risk, so both get an explicit policy.

- **Pairing happens only when Home Assistant asks**, on `BluetoothDeviceRequest` with the pairing
  call — never because a device in range started pairing with us. BlueZ's agent machinery is
  already here (`RegisterAgent` in `lib/bluez`, with the pin/passkey/confirm callbacks used for
  audio); the proxy registers the same agent and answers every request by the rule that the proxy
  only pairs when a paired Home Assistant says so. The alternative — auto-accepting whatever asks —
  is how a device on the other side of a wall bonds to a speaker and is briefly trusted.
- **Bonding is remembered by BlueZ and not by us.** `Device1.Paired` / `Bonded` and the trust flag
  stay BlueZ's business; the daemon only asks. This keeps one source of truth with the audio side
  (a device forgotten for audio is forgotten for the proxy) and avoids a second key store on
  userdata.
- **Cache clearing is `Adapter1.RemoveDevice`,** which is right in spirit and needs one check
  before it is promised: on BlueZ, removing a device also forgets its bond, and Home Assistant's
  cache-clear is meant to fix a stale service list rather than to unpair. If the check shows that,
  the flag is dropped rather than faked — a proxy that silently unpairs a lock is worse than one
  that declines to clear a cache.
- **The beacon comes back for free.** With `LEAdvertisingManager1.RegisterAdvertisement`, the Show's
  scanner can finally advertise the iBeacon it currently skips (`hardware/ble/bluez.go`), which is
  the "no beacon" half of the known gap in `overview.md`. This is deliberately *not* bundled into
  the connections work — it is a separate, much smaller change with its own test — but the same
  D-Bus extension unlocks it, and the plan notes it so the work is not done twice.

## Settings, surfaces and what a user sees

Two switches, and the reason for two is that they are two different risks.

- **Bluetooth proxy** (exists, `config.Bluetooth.Proxy`, off by default): the radio scans and
  advertisements go to Home Assistant. `docs/setup.md:243` warns what it costs.
- **Bluetooth connections** (new, off by default): Home Assistant may connect, read, write, pair
  and forget. Proposed `config.Bluetooth.Connections`, object id `bluetooth_connections`, name
  "Bluetooth connections".

Why not one switch? Because somebody can reasonably want advertisements forwarded and not want a
program on the network to be able to write to a lock. Why not three (pairing separate again)?
Because pairing without connections is useless and connections without pairing still cover most
devices; a third toggle would be a knob nobody sets correctly. **Connections imply the proxy**: the
connections switch cannot be on while the proxy switch is off, because connections are requested
over the same subscription Home Assistant opens for advertisements. Turning the proxy off turns
connections off with it (and logs that it did), which is the safe direction.

Where it appears:

- **The screen**, on the Connections card beside the proxy row: the same shape as
  `settingRow{id: "btproxy"}` (`display/sheet.go:339`, handled at `sheet.go:1107`), with a
  sub-line saying what it lets Home Assistant do and how many connections are free. The Dot has no
  screen, which is the next point.
- **Home Assistant**, as the switch entity, so an automation can turn it on for a week of
  troubleshooting and off again.
- **The setup page**, which the proxy row does *not* have today. This is a deliberate addition: the
  page exists for "settings that are miserable or impossible to set on the device itself", and a
  Dot is exactly that device — four buttons and no screen, where a connection switch is otherwise
  unreachable. It goes on Privacy & Security with the other radio switches.
- **Diagnostics**: connections free, connections held (count only), and how many advertisements
  have been dropped, in the existing bundle.

Flipping either switch has to do what the proxy switch already does when it changes the advertised
features — `component.Reconnect.Emit(struct{}{})` (`bluetooth.go:113`) — because Home Assistant
reads the feature flags once per connection and will otherwise keep believing the old ones.

## Security: what this actually is

This is the first thing on the device that lets something else on the network make the radio talk
to arbitrary devices in the room, including writing to them and bonding with them. Stated plainly,
and designed around:

- **Nothing new listens.** The only way in is the ESPHome API link that already exists, already
  encrypted, already paired. No port, no mDNS, no unauthenticated path. This is the single strongest
  argument for doing it this way rather than as a separate service.
- **Off by default, on deliberately**, and only while on. A device handed to a relative has it off.
- **A connection can be read and written, not trusted.** Pairing is a separate, explicit request
  (above), and the agent never auto-accepts.
- **Every connection is logged**: address, name if known, when, how long, and why it ended. The log
  is the only place this is visible on a Dot.
- **Device names and addresses in the diagnostics bundle stay redacted** by the existing rule; the
  counts are not.
- **It cannot run a command, change a setting, or reach anything but the radio.** The write surface
  is GATT characteristics on a device the link asked for.

## Tests, and what "done" means

The pattern in this tree is that the protocol conversion is testable without hardware and the radio
is tested on the bench. Both halves here have precedents to follow:

- **Unit, no hardware.** Put the GATT operations behind an interface in `feature/bluetooth` (the
  same shape as `ble.Scanner` in `hardware/ble/scanner.go`), and test: the message-to-operation
  mapping for every request type; the UUID encoding (short and long, both endiannesses); the handle
  map's stability across a rediscovery and its per-connection lifetime; slot accounting (free,
  limit, allocated) as connections come and go; and the error mapping from BlueZ's D-Bus errors to
  ESPHome's error codes, so a failed read is a failure Home Assistant can see. The existing tests to
  copy from are `feature/bluetooth/bluetooth_test.go`, `retry_test.go`, and
  `hardware/ble/addr_bluez_test.go` (which already pins "what BlueZ prints is what Home Assistant
  receives").
- **Bench, real devices.** One connection-only device that Home Assistant actually supports
  (SwitchBot or a stock Xiaomi/Mijia sensor are the cheapest to buy for this), one phone, and one
  speaker. On a Show, a Spot and a Dot: connect, discover, read, notify, disconnect, reconnect; then
  with earbuds connected; then with a phone playing to the device (the #105 case); then measuring
  Wi-Fi throughput during a connection, daemon CPU, and discovery and notification latency. Those
  numbers go in this document when they exist, the way the cast plan's step 0 numbers did.
- **The lie test.** With the switch off, the device must answer exactly as it does today: no new
  flags, and a connections-free reply of zero. The stub in the library is a useful oracle here.

## Order of work

Each step lands on main with tests and gets a bench check before the next one starts, as the other
plans do.

1. **`lib/bluez` grows a GATT client.** Services, characteristics, descriptors from
   `GetManagedObjects`; `ReadValue`, `WriteValue` (with the `type` option for
   write-without-response), `StartNotify`, `StopNotify`; `PropertiesChanged` on `Value` for
   notifications; `Adapter1.RemoveDevice`. Unit tests where the shapes are ours (the number of
   messages this library can be tested against is limited by D-Bus; the value-encoding and object
   walking are not). A few days.
2. **The lie is made honest: real capacity.** Intercept
   `SubscribeBluetoothConnectionsFreeRequest` in `feature/bluetooth`, answer with `Free`/`Limit`/
   `Allocated`, and add the `FEATURE_ACTIVE_CONNECTIONS` flag behind the new switch. Home Assistant
   now believes the device can connect. Nothing connects yet — and this is the step that is worth
   shipping on its own, because it is what makes Home Assistant start *choosing* the device and is
   where a wrong feature mask shows up immediately. A day.
3. **Connect and disconnect.** `BluetoothDeviceRequest` to `Device1.Connect` / `Disconnect`;
   `BluetoothDeviceConnectionResponse` on success, failure and drop; slot accounting wired to it.
   The first bench test: point Home Assistant at a device and watch it hold a link. Two to three
   days.
4. **Service discovery and handles.** `BluetoothGATTGetServicesRequest` answered from the BlueZ
   tree, in batches, with the synthetic handle map and the UUID encoding. This is the step where a
   wrong handle shows up as every read failing, so it lands with the unit tests from the section
   above. Two to three days.
5. **Read and write.** `ReadValue`/`WriteValue` with the descriptor variants and the write
   response mode; `BluetoothGATTErrorResponse` for every failure path. A day or two.
6. **Notify.** `StartNotify`/`StopNotify` and `PropertiesChanged` → `BluetoothGATTNotifyDataResponse`,
   including the unsubscribe-on-disconnect case. A day.
7. **Pairing, bonding and cache clear**, with the policy above, and the check on what
   `RemoveDevice` forgets. A day, plus whatever the check turns up.
8. **Policy and surfaces**: the audio coexistence rule, the three-slot limit, the switch on the
   screen, the setup page and Home Assistant, the diagnostics counts, the logs, and the reconnect
   on a feature change. Two days.
9. **Docs and numbers**: `docs/setup.md` (what the second switch does, and the warning that goes
   with it), `docs/overview.md` (the known gap closes; the beacon note stays until step 10),
   `actions.md` if an action is wanted for automations, and the measured numbers written into this
   file. A day.
10. **Separate and small: the beacon on the Show** (`LEAdvertisingManager1`), which the same client
    unlocks. Half a day, its own release.

Rough total: two to three weeks of evenings, most of it in steps 1, 4 and 8, plus whatever the bench
says about the radio being shared.

## Decisions

1. **Connections are made by `bluetoothd`, over D-Bus, not by our own HCI code.** The stack that
   owns the controller is the stack that can hold a connection, and it is the same stack the audio
   features depend on. Raw HCI cannot hold an LE connection at all (no `hci_conn`); taking the
   controller means owning A2DP and HFP; a second controller is not possible with one radio.
2. **One implementation, all boards.** The GATT client lives in `lib/bluez` and serves the Show, the
   Spot and the Dot. Only scanning stays per-board, as it already is.
3. **The translation lives in `feature/bluetooth`**, not in a fork of the ESPHome library, because
   everything hard about it is hardware. The library's `Handle` is our delegation point, so the
   messages we want are ours before it sees them; an upstream patch to remove its stub response is
   the follow-up, not a prerequisite.
4. **Handles are assigned by us, once per connection.** BlueZ's `Handle` is documented as server-only
   and optional, and a mixed numbering would not agree with itself; stability for the life of a
   connection is what Home Assistant needs.
5. **Three connections, and refused while audio streams.** The radio is shared with Wi-Fi
   (measured), scanning already has a duty cycle to protect it, and the #105 warning is already in
   the user documentation. Refusing is a shape Home Assistant understands; dropping an established
   connection is not.
6. **A second switch, off by default, that implies the proxy.** Forwarding advertisements and
   allowing writes to a lock are different risks, and the device's owner should be able to choose
   between them.
7. **Pairing only on an explicit request; bonds belong to BlueZ.** One source of truth with the
   audio side, and no path by which a device in the room pairs itself to the speaker.
8. **Cache clearing is promised only if it does not unpair.** If `RemoveDevice` turns out to forget
   the bond, the flag stays off.
9. **Nothing new listens.** The ESPHome link is the only way in, which is the strongest security
   property this feature has.

## Open questions

- **What exactly Home Assistant does with `Limit` and `Free`** when choosing a proxy, and whether it
  re-reads the response on every change. The protocol says it is subscribed; the client-side
  behavior in the current Home Assistant is worth re-checking at step 2, since it decides whether
  refusing needs a re-send or whether the subscription handles it.
- **Whether BlueZ 5.86 on the image exposes `Handle` on client-side GATT objects** despite the
  documentation. It does not change the design (we synthesize either way) but it decides whether the
  preference inside the map is worth writing.
- **What `Adapter1.RemoveDevice` does to a bond on 5.86**, and whether Home Assistant's cache clear
  arrives with the device connected or disconnected.
- **Whether a connection plus the existing scan duty cycle is too much on a Dot.** The Dot has the
  least CPU and the busiest radio; it may end up advertising its capacity honestly at one or two
  rather than three.
- **Which Home Assistant integrations are worth testing first**, in the order they are likely to be
  used and complained about: SwitchBot, Xiaomi/Mijia, a lock, a scale. The plan's benchmark devices
  should be bought with that order in mind.

## What this plan is not

- **Not a local BLE client for our own features.** The BLE button in `standalone-plan.md` ("a BLE
  button as local control") is a different consumer of the same machinery, and this plan is written
  so that it gets easier rather than harder — a GATT client in `lib/bluez` is exactly what that
  needs. It does not build it.
- **Not a Bluetooth audio change.** Earbuds, the phone-as-source and AVRCP are untouched except for
  the one rule that a live audio stream refuses new connections.
- **Not the iBeacon.** The Show's missing beacon is unlocked by the same client and is left as its
  own small step, so the connections work can land without waiting on it.
- **Not a reason to turn the proxy on for everybody.** It stays off by default, with the same
  warning about the shared antenna that is already in the user documentation.
