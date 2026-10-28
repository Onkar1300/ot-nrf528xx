# Multi-PAN radio port to ot-nrf528xx release/v2026.10.0: port notes

Branch `feature/nrf-multipan-v10`, based on `release/v2026.10.0` (`382ea3d`).
Source of the changes: `feature/nrf-multipan-radio` of the old `ot-nrf528xx` clone (5 commits on old `main` `8e10133`).
OpenThread: `feature/ot-multipan-v10` @ `cd34cb871` (the Multi-PAN port of OpenThread v2026.10.0, see its `PORT-NOTES.md`).
Plan: `H:\thread\Updates\nrf-plan.md`.

## 1. Commits

| # | Commit | Content |
|---|---|---|
| A | `[openthread] point the submodule at the Multi-PAN port` | gitlink to `cd34cb871` (R8) |
| B | `[radio] accept multiple PAN IDs in the 802.15.4 driver` | R1, R2, N2/N3, `nrf_802154.c/.h` wrappers, filter PAN record (see 4.3) |
| C | `[transport] enlarge the UART RX buffer for the Multi-PAN key map` | R3 per N5 |
| C2 | `[script] use env to find bash in bootstrap` | R9 (same blob as the old branch) |
| D | `[radio] per-PAN MAC keys for transmit and Enh-ACK` | R4 to R7, nrf52811 guard |
| E | `[docs] add Multi-PAN port notes` | this file |

Submodule setup (local only, not committed): `git config submodule.openthread.url H:/thread/Updates/new/openthread`.
`.gitmodules` keeps the upstream URL. Nothing was pushed. If `feature/ot-multipan-v10` moves, commit A has to be redone.

## 2. Decisions applied

| ID | Applied as |
|---|---|
| N1 | All `radio.c` Multi-PAN code is under `#if OPENTHREAD_CONFIG_MULTI_PAN_ENABLE` (CMake `OT_MULTI_PAN`). The driver changes are not gated; with Multi-PAN off the driver behaves as stock (section 6, T8). |
| N2 | Capacity 64 (`NRF_802154_PIB_PAN_ID_LIST_SIZE` in `nrf_802154_config.h`, array, comments; `radio.c` has an `#error` if it is smaller than `OT_RADIO_MULTI_PAN_MAX_KEYS`). `nrf_802154_pib_pan_id_set()` replaces the list with one PAN. New `nrf_802154_pib_pan_id_add()` (ignores duplicates, refuses when full), `nrf_802154_pib_pan_id_list_set()`, `nrf_802154_pib_pan_id_list_get()`, `nrf_802154_pib_pan_id_exists()`, and the `nrf_802154_pan_id_add()` / `nrf_802154_pan_id_list_set()` wrappers. With Multi-PAN on, `otPlatRadioSetPanId()` adds; `otPlatRadioSetMacKeyMultiPan()` replaces the list with the PANs of the used map entries (entry 0 first, duplicates removed). |
| N3 | `nrf_802154_pib_pan_id_get()` returns the first PAN of the list (broadcast PAN when the list is empty). The old `TODO ... UDP Security issue` comment is gone. |
| N4 | Local submodule URL override, `.gitmodules` unchanged (section 1). |
| N5 | Done as CMake definitions: `UART_RX_BUFFER_SIZE=2048` is added to `OT_PLATFORM_DEFINES` in `nrf52833.cmake` and `nrf52840.cmake` only `if(OT_MULTI_PAN)`. `transport-config.h` keeps `512` as the default. nrf52811 untouched (256) and refused for Multi-PAN (`#error` in `openthread-core-nrf52811-config-check.h`). The USB CDC transport sizes its RX buffer from the same macro, so it also grows to 2048 with Multi-PAN. |

## 3. Design of the `radio.c` port

- `otPlatRadioSetMacKey()` is unchanged (router keys `sPrevKey/sCurrKey/sNextKey`, `sPrevMacFrameCounter = sMacFrameCounter`).
- `otPlatRadioSetMacKeyMultiPan()` matches `include/openthread/platform/radio.h` of the OpenThread port. It computes the PAN list
  outside the critical region, then under `CRITICAL_REGION_ENTER/EXIT` sets `sKeyId`, copies the used part of the map into
  `sPanKeys[64]` (zeroing entries that were used before and are not now) and calls `nrf_802154_pan_id_list_set()`. It does not touch
  `sMacFrameCounter` or `sPrevMacFrameCounter`.
- Key lookup: `findPanKeys(panId)` returns the first used entry with that PAN (an entry is unused when its PAN ID and its three keys
  are all zero, at any index including 0). `panKeyFor(panId, routerKey)` returns the key of that entry in the same slot
  (previous/current/next) as `routerKey`, and returns `routerKey` itself when there is no entry **or the entry is entry 0**.
  So an empty or not-yet-received map can never give an all-zero key.
- Transmit (R5): in `nrf_802154_tx_started()`, after the v10 `mIsSecurityProcessed` early exit and the stock
  `mAesKey = &sCurrKey`, the key is replaced by `panKeyFor(pan, &sCurrKey)`. The PAN is the destination PAN, or the source PAN when
  there is none (same rule as `SubMac::GetTxKeyTrio()`). Key ID and frame counter assignment in `otPlatRadioTransmit()` are unchanged.
- Enh-ACK (R6): `txAckProcessSecurity()` keeps the v10 key-index chain (`keyId - 1`, `(curr + 127) & 0x7f`, `(curr + 1) & 0x7f`)
  and frame counter handling unchanged; after the chain picked `sPrevKey/sCurrKey/sNextKey`, `panKeyFor()` maps it to the same slot
  of the acknowledged PAN. The old branch used `curMacKey` for all three cases, which broke key rotation; that is fixed.
- PAN registration (R7): `otPlatRadioSetPanId()` calls `nrf_802154_pan_id_add()` (inside a critical region) with Multi-PAN on and the
  stock `nrf_802154_pan_id_set()` otherwise. The old private `sPanIdList/sPanIdCount` are not carried over.

## 4. Verify items

| Item | Result |
|---|---|
| 5.1 prototype | Matches `radio.h:821` of the OpenThread port exactly; builds with `-Werror`. `otPlatRadioSetMacKeySingle`, `otPanIdKeyMaterialMap` as parameter and `kMaxPanKeys` are not used. |
| 5.2 key type | No nRF config enables `OPENTHREAD_CONFIG_PLATFORM_KEY_REFERENCES_ENABLE`, so keys are literal. `assert(aKeyType == OT_KEY_TYPE_LITERAL_KEY)` kept. |
| 5.4 ACK PAN source | **Confirmed problem, resolved.** The Enh-ACK generator (`nrf_802154_enh_ack_generator.c`) copies the PAN ID compression bit of the acknowledged frame and uses no source address. For the usual Thread frame (both PANs equal, at least one short address: 802.15.4-2015 rows 12-14, PAN ID compression 1) the ACK therefore has **no PAN ID**. The old branch then used PAN 0, i.e. map entry 0. Fix: the receive filter remembers the destination PAN of the frame it checks (`nrf_802154_filter_last_dst_pan_id_get()`, reset at the first filter stage of every frame). An ACK is only sent for a frame that passed the filter, and no other frame is filtered before its ACK, so in `nrf_802154_tx_ack_started()` this is the PAN of the acknowledged frame. `radio.c` uses the ACK's own destination PAN when present (that is what goes on air: the frame's source PAN, else its destination PAN), else the remembered PAN; with neither (e.g. 2015 ext/ext frame with compression, which OpenThread does not send for unicast data) the router keys are used. The ACK generator is unchanged. |
| 5.4 ISR context | `nrf_802154_tx_ack_started()` and `nrf_802154_tx_started()` run in the RADIO IRQ (`irq_address_state_tx_ack()` / `transmit_started_notify()` in `nrf_802154_core.c`). Added work: one MHR parse and a scan of at most 64 entries (2-byte compare each); no logging, no blocking, no critical section. For the router's own PAN the scan stops at entry 0. Cost not measured (open item). |
| 5.6 list swap vs ISR | `CRITICAL_REGION_ENTER()` is `__disable_irq()` in builds without SoftDevice (`ot-rcp`), so the RADIO IRQ (and its receive filter) cannot see a half-written list or map. With `SOFTDEVICE_PRESENT` it is `sd_nvic_critical_region_enter()`, which does not mask SoftDevice-owned priorities; this is the same protection the stock `otPlatRadioSetMacKey()` has. The interrupt-off time grows with the map: about 52 bytes per used entry (worst case 3,328 bytes, estimated at about 50 us at 64 MHz). |
| 7 build environment | The build clone needs `openthread/third_party/mbedtls/repo` and its nested `framework` submodule (nrf52833/nrf52811 build OpenThread's mbedTLS; configure fails without `framework`). Both were initialised in the WSL clone only, from the local copies in `new\openthread`. No project change. |
| 8 `nm` | Stock nrf52840 `ot-rcp`: no Multi-PAN symbol (`sPanKeys`, `panKeyFor`, `otPlatRadioSetMacKeyMultiPan`, ...). Symbols new versus stock `release/v2026.10.0`: `nrf_802154_pib_pan_id_exists`, `m_last_dst_pan_id`, `m_last_dst_pan_id_present` (driver, commit B), plus a renumbered compiler switch table. |
| Risk 5 SubMac early exit | **Not real.** `SubMac::CommitPanMode1MacKeys()` exits only when `ShouldHandle(kCapTransmitSec)` is true. `ShouldHandle()` returns false first when the radio supports the capability, and nRF reports `OT_RADIO_CAPS_TRANSMIT_SEC` (Thread >= 1.2). `OPENTHREAD_CONFIG_MAC_SOFTWARE_TX_SECURITY_ENABLE=1` only matters for radios without it. RCP path: `ncp_base_radio.cpp` -> `otLinkRawSetMacKeyMultiPan()` -> `LinkRaw::SetMode1MacKeysMultiPan()` -> `SubMac::CommitPanMode1MacKeys()` -> `otPlatRadioSetMacKeyMultiPan()`. Note: the key index passed is the RCP `SubMac`'s own index (from the last single-key call), not the index field of the Spinel message; they are equal in normal operation. |

## 5. End-device transparency contract (plan section 9)

| # | Result | Evidence |
|---|---|---|
| T1 | Pass | `dst_pan_id_check()` accepts the broadcast PAN and every listed PAN; the auto-ACK path is unchanged, so listed PANs are ACKed. The list holds the router PAN, every PAN of the map, and every PAN added since the last map. |
| T2 | Pass, with the limits below | Unlisted PANs are rejected. Every map replaces the list, so a PAN that leaves the host's list is dropped at the next map, which the host sends on every PAN-list change (`KeyManager::SetPanIds()` sends the map before re-adding the remaining PANs), PAN change and key change. See limits L2 to L4 and finding F1. |
| T3 | Pass (code), with limit L1 | The per-PAN slot is chosen from all three keys, with the v10 index arithmetic unchanged. |
| T4 | Pass | `nrf_802154_enh_ack_generator.c` and `nrf_802154_imm_ack_generator.c` are unchanged. Their only PIB read is the PAN fallback for an ACK that needs a PAN when the frame has none: only a frame with no source address and no PAN ID (802.15.4-2015 table row 4), which OpenThread does not send. There the primary PAN is used, which is the router's PAN after the first map, as stock. |
| T5 | Pass | One global `sMacFrameCounter`/`sPrevMacFrameCounter` as stock; `otPlatRadioSetMacKeyMultiPan()` never writes them. |
| T6 | Assumption | `sKeyId` is global; the map call sets it from the same key index the single-key call uses. All PANs share the router's key sequence (the host derives every PAN's keys from `mKeySequence`; end devices learn the sequence through Joiner Entrust, `joiner_router.cpp`). Not verifiable in this tree. |
| T7 | Pass | Diff review: no change in beacon, broadcast, CSL IE or timing code. The beacon rule of the filter also accepts beacons when the broadcast PAN is anywhere in the list (needed for the router's own active scan with add semantics); beacons are never ACKed, so there is no on-air change. |
| T8 | Pass | With Multi-PAN off, `radio.c` preprocesses byte-for-byte the same at D as at C (`gcc -E -P`, nrf52840 `ot-rcp` flags). Driver: `pan_id_set` keeps replace semantics; a one-entry list filters exactly like the old single PAN; init is one broadcast PAN as before. Cost: +144 B flash, +132 B RAM (nrf52840 `ot-rcp`, same code on every platform). |
| T9 | Pass | No new frames or IEs. Timing: only the ISR work noted in 4 (5.4 ISR) and the interrupt-off time of a map push (5.6). |

**Finding F1 (stopped, not changed; on-air difference only during the router's own active scan).**
Stock: `Mac` sets the radio to the broadcast PAN for an active scan (`mLinks.SetPanId(kPanIdBroadcast)`), so frames addressed to
the router's PAN are not received or ACKed until the scan ends. With the OpenThread port's contract (every `otPlatRadioSetPanId()`
adds a PAN, N2), the radio keeps its PANs during the scan. A stock end device sending to the router on the PAN channel during the
router's active scan therefore gets an ACK where a stock router would give none (OpenThread processes such frames when the scan
channel is the PAN channel, so they are delivered rather than lost). The radio cannot tell "scan" from "add" without changing N2
(for example by treating `otPlatRadioSetPanId(0xffff)` as "replace"). Active scans are operator-initiated (`scan`,
`otLinkActiveScan()`). Decision needed from Onkar: accept, or change N2/the OpenThread contract.

## 6. Build matrix

Toolchain: WSL Ubuntu, GNU Arm Embedded 9-2020-q2 (gcc 9.3.1), cmake 3.28.3, ninja. Clone `~/nrf-latest-build` of the committed
branch, build trees in `~/nrf-builds` (nothing inside `nrf-latest`). `script/build` sets `-DOT_COMPILE_WARNING_AS_ERROR=ON`.

| Config | Command | Result |
|---|---|---|
| nrf52840 stock, commits A, B, C, D | `./script/build nrf52840 USB_trans -DOT_BOOTLOADER=USB` | Pass (all 6 targets) |
| nrf52840 Multi-PAN, commit B | same + `-DOT_MULTI_PAN=ON` | Compiles; link fails on `otPlatRadioSetMacKeyMultiPan` (by design of the OpenThread port, no default implementation) until D |
| nrf52840 Multi-PAN, commit D | same + `-DOT_MULTI_PAN=ON` | Pass (all 6 targets, `ot-rcp` links `otPlatRadioSetMacKeyMultiPan`) |
| nrf52833 stock / Multi-PAN, commit D | `./script/build nrf52833 UART_trans [-DOT_MULTI_PAN=ON]` | Pass / Pass |
| nrf52811 stock, commit D | `./script/build nrf52811 UART_trans` | Pass |
| nrf52811 Multi-PAN, commit D | same + `-DOT_MULTI_PAN=ON` | Fails as intended: `#error "Platform nrf52811 doesn't support configuration option: OPENTHREAD_CONFIG_MULTI_PAN_ENABLE (not enough RAM)"` |
| N5 check | configure nrf52840/nrf52833 with `-DOT_MULTI_PAN=ON` | `transport/uart.c` gets `-DUART_RX_BUFFER_SIZE=2048`; stock builds get no definition (512) |

Commit C (CMake only) was built for nrf52840 stock; C2 and E do not touch built files.

`arm-none-eabi-size` of `ot-rcp` (text / data / bss / dec):

| Build | text | data | bss | dec |
|---|---|---|---|---|
| nrf52840 stock, `release/v2026.10.0` + A | 69232 | 204 | 21624 | 91060 |
| nrf52840 stock, D | 69376 | 204 | 21756 | 91336 |
| nrf52840 Multi-PAN, D | 70928 | 204 | 37884 | 109016 |
| nrf52833 stock, D | 52180 | 168 | 11664 | 64012 |
| nrf52833 Multi-PAN, D | 53716 | 168 | 27792 | 81676 |
| nrf52811 stock, D | 52080 | 168 | 10568 | 62816 |

nrf52840 Multi-PAN versus stock: +1,552 B flash, +16,128 B RAM. Largest RAM items: NCP/HDLC buffers +7,680 (4096-byte Spinel
frames), OpenThread instance +3,584 (`SubMac` per-PAN key trios), `sPanKeys` 3,328, USB CDC RX buffer +1,536.

## 6.1 Deviations from the plan

1. **Filter records the destination PAN (commit B).** Forced by verify item 5.4: the ACK often has no PAN. Added in the filter, not in
   the ACK generator, so T4 holds. Two static bytes and a flag, written for every filtered frame.
2. **`txAckProcessSecurity()` keeps its stock signature.** The plan passes the PAN as a parameter from `nrf_802154_tx_ack_started()`.
   The PAN is instead looked up inside, under the guard, after the security checks pass, so unsecured ACKs pay nothing and the
   stock code path is unchanged.
3. **Map entry 0 uses the router keys.** The plan treats entry 0 as a normal entry. By the API contract entry 0 holds the same keys
   as `otPlatRadioSetMacKey()`, but the host sends the single key first and the map after it (`KeyManager::UpdateKeyMaterial()`),
   so the router keys are never older than entry 0. Using them keeps the router's own PAN exactly stock, also during a rotation.
4. **The "unused entry" test applies to entry 0 too** (plan: "beyond entry 0"), so a frame with PAN 0 can never match the all-zero
   start-up map.
5. **Beacon rule** also accepts beacons when the broadcast PAN is anywhere in the list, not only as the primary PAN (otherwise the
   router's own active scan would lose foreign beacons with add semantics). Identical to stock for a one-entry list.
6. **Extra commit C2** for R9 (allowed by the plan as "optionally its own tiny commit").
7. **`#error` for Multi-PAN with Thread < 1.2** in `radio.c` (all transmit security code is 1.2-only).

## 7. Known limits

- L1 **Key rotation window for non-router PANs.** The host sends `otPlatRadioSetMacKey()` (new `sKeyId`) and then the map as two
  Spinel commands. Between them, frames and Enh-ACKs on the *other* PANs use the old map with the new key index (wrong key). The
  router's own PAN is not affected (deviation 3). Length: one map transfer (a ~3.2 KB Spinel frame: milliseconds over USB, about
  0.3 s at 115200 baud UART), once per key rotation. Fix belongs to the API (one atomic call, or the map sent first with the RCP
  switching on the key index).
- L2 All PANs share the router's key sequence; a device whose sequence drifts more than +/-1 cannot talk to the router (same as
  stock Thread).
- L3 Stale-PAN window: `otPlatRadioSetPanId()` only adds; a PAN is removed at the next map. The broadcast PAN added for an active
  scan stays in the list until then (no on-air effect, see T7).
- L4 PANs listed by the host **without a PAN key** are registered by `otPlatRadioSetPanId()` but are not in the map, so each map
  removes them again (`KeyManager::GetNumKeyedPans()`). Such a PAN cannot be served anyway (no MAC or MLE key), but whether its
  frames are ACKed depends on whether the last event was a PAN-list update (added) or a map push (removed).
- L5 A Multi-PAN RCP driven by a host without Multi-PAN never receives a map, so every PAN the host ever set stays in the list (up
  to 64, then `pan_id_add` refuses silently). Use a stock RCP with a stock host.
- L6 Interrupt-off time of a map push grows with the number of used entries (section 4, 5.6).

## 8. Open items for later experiments

- Secured Enh-ACK on air to a stock device on a listed (non-router) PAN, with the device on the previous, current and next key index.
- Enh-ACK for a child whose PAN is not the router's, for frames with and without PAN ID compression (checks the filter PAN record).
- Key rotation: length and effect of window L1 on another PAN, over USB and over UART.
- Stale-PAN window (L3, L4) and the active-scan behaviour of finding F1.
- CSL receiver/transmitter and `ReceiveAt` together with per-PAN keys.
- ISR timing: cost of the PAN parse and map scan in `tx_started`/`tx_ack_started`, and the interrupt-off time of a 64-entry map push
  while ACKs are being sent.
- nrf52833 over USB and UART with the 2048-byte RX buffer.
- A real Spinel key-map transfer (4096-byte frame) to the RCP, including RCP reset and restoration.
