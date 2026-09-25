Here's your handoff document — everything confirmed, everything still open, spelled out plainly enough that even a fresh Claude Code session with no memory of this saga can pick it up without me having to hold its hand. You're welcome.

---

# SIDE-KEYBOARD Rust Remapper — Project Spec

## Device Identification

| Field | Value |
|---|---|
| Manufacturer/Name | SDINNOVATION SIDE-KEYBOARD |
| USB Vendor:Product ID | `6d7f:dcfe` |
| Serial | `535308426172` |
| Connection | 2.4GHz wireless dongle, `hotplug` |
| USB port | `1-4` |
| USB interfaces | `{ 03:00:00  03:01:01  03:00:00 }` — three HID interfaces, no mass storage, no network/CDC |

## Security Validation (completed, do not need to redo)

- ✅ USBGuard interface audit: all three interfaces are class `03` (HID). No `08` (mass storage) or `02` (network) interfaces present — rules out the classic "keyboard that's secretly a flash drive" or "phones home" attack shapes.
- ✅ Idle test: device produced zero events over multiple minutes with no physical interaction.
- ✅ Timing test: keystroke/knob-turn intervals showed irregular, human-paced gaps (0.6–4.7s between actions), not the uniform sub-50ms bursts characteristic of an injected payload.
- Conclusion: hardware is **not malicious**, just cheap. Firmware takes lazy shortcuts (see below) rather than doing anything hostile.

## Control Inventory

### Knobs (×2)
- Both currently mapped by firmware to the same redundant behavior: `KEY_VOLUMEUP` (115) / `KEY_VOLUMEDOWN` (114).
- Confirmed emitting node: `/dev/input/event24` (evdev name contains `"...Keyboard"`).
- **Open TODO:** confirm whether the second knob shares this same node or has its own — isolate each knob physically, one at a time, with `evtest` on each candidate node (`event24`, `event25`) and log which one lights up per knob.

### Macro buttons (×9)
- Do **not** send single key codes — firmware sends full modifier+key chords (cheaper to implement in firmware than custom HID usages).
- Confirmed node: `/dev/input/event27` (bare evdev name `"SIDE-KEYBOARD"`).
- Confirmed signature so far:
  - One button → `KEY_LEFTCTRL (29)` + `KEY_A (30)` (i.e., "Select All" as a side effect of lazy chord-mapping).
- **Open TODO:** isolate and log the raw chord for the remaining 8 buttons individually via `sudo evtest /dev/input/event27`, one press at a time.

### Other enumerated nodes (role unclear — monitor, don't ignore)
- `/dev/input/event25` — `"...Mouse"`, reports `scroll-nat`, `scroll-button`, `left`. Possibly the second knob riding a different usage page. Needs isolation test above.
- `/dev/input/event26` — `"...Wireless Radio Control"`. Should only ever fire on the physical RF pairing button. Any event here outside of an intentional pairing action is worth a second look.

## Project Goal

Build a Rust userspace remapper (not a kernel driver — `hid-generic` already handles enumeration correctly) that:

1. Locates the device portably by **vendor:product ID (`0x6d7f:0xdcfe`)** at runtime via `evdev::enumerate()`, never by a hardcoded `/dev/input/eventN` path — those numbers are not stable across reboots or machines.
2. `.grab()`s the relevant real device node(s) exclusively, so raw firmware output (duplicate volume events, unwanted Ctrl+A, etc.) never leaks to the desktop.
3. Detects the 9-button chords via a buffered held-key `HashSet`, matched against a signature table built from the isolation testing above.
4. Emits clean, user-chosen output through a virtual device created via `/dev/uinput`.
5. Always emits matching key-**up** events for anything intercepted, to avoid stuck modifiers.
6. Eventually runs unattended as a `systemd --user` service.

## Environment Setup

```bash
sudo usermod -aG input $USER
```
```bash
# /etc/udev/rules.d/99-uinput.rules
KERNEL=="uinput", MODE="0660", GROUP="input", OPTIONS+="static_node=uinput"
```
Log out/in after the group change before testing.

## Dependencies

```toml
# Cargo.toml
[package]
name = "side-keyboard-remapper"
version = "0.1.0"
edition = "2021"

[dependencies]
evdev = "0.13"
```

## Starter Skeleton (portable device lookup, chord-ready)

```rust
use evdev::{enumerate, Device, EventType, InputEvent, Key, AttributeSet};
use evdev::uinput::VirtualDeviceBuilder;
use std::collections::HashSet;
use std::error::Error;

const VENDOR_ID: u16 = 0x6d7f;
const PRODUCT_ID: u16 = 0xdcfe;

fn find_device(name_hint: &str) -> Option<Device> {
    for (_, device) in enumerate() {
        let id = device.input_id();
        if id.vendor() == VENDOR_ID && id.product() == PRODUCT_ID {
            if let Some(name) = device.name() {
                if name.contains(name_hint) {
                    return Some(device);
                }
            }
        }
    }
    None
}

fn main() -> Result<(), Box<dyn Error>> {
    // TODO: confirm exact name_hint strings once knob isolation is done —
    // "Mouse" and bare "SIDE-KEYBOARD" are best guesses from initial enumeration.
    let mut knob_device = find_device("Keyboard").expect("knob device not found");
    let mut macro_device = find_device("SIDE-KEYBOARD").expect("macro pad not found");

    knob_device.grab()?;
    macro_device.grab()?;

    let mut keys = AttributeSet::<Key>::new();
    keys.insert(Key::KEY_VOLUMEUP);
    keys.insert(Key::KEY_VOLUMEDOWN);
    // TODO: add whatever keys the 9 remapped macro buttons should emit.

    let mut virt = VirtualDeviceBuilder::new()?
        .name("side-keyboard-remapper")
        .with_keys(&keys)?
        .build()?;

    let mut held: HashSet<u16> = HashSet::new();

    loop {
        // TODO: poll both devices (e.g. via separate threads or an async
        // executor) rather than blocking sequentially on one.
        for event in macro_device.fetch_events()? {
            if event.event_type() == EventType::KEY {
                let code = event.code();
                if event.value() == 1 { held.insert(code); }
                if event.value() == 0 { held.remove(&code); }

                // TODO: match `held` against the full 9-button signature
                // table once all chords are logged, and emit the desired
                // output via `virt`. Always emit matching up-events.
            }
        }
    }
}
```

## Outstanding TODOs Before Implementation

- [ ] Physically isolate and log which node the second knob actually uses (`event24` vs `event25`)
- [ ] Isolate and log the raw chord signature for all 9 macro buttons individually
- [ ] Decide final target behavior for each knob and each button (current firmware mapping is redundant/lazy and should be replaced)
- [ ] Confirm `event25`/`event26` roles definitively before deciding whether they need grabbing too
- [ ] Multi-device event loop design (threads vs. async) once both device paths are grabbed simultaneously
