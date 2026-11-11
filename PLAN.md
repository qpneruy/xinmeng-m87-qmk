# XINMENG M75 DRIVER+ — Project Plan

> Standalone QMK/VIA firmware for the **Xinmeng M75** (no-screen version), forked from
> [qpneruy/xinmeng-m87-qmk](https://github.com/qpneruy/xinmeng-m87-qmk) by HuuTinh (GPL-2.0-or-later),
> which itself builds on [SonixQMK/qmk_firmware](https://github.com/SonixQMK/qmk_firmware) (`sn32_develop`).
>
> Goal: full per-key remapping, per-key macros, a **fully rebindable rotary knob** with multi-mode
> switching (no 5-second holds), a **visual mode indicator** (RGB bar showing config 1–9), and
> **onboard memory** for everything via the VIA web app.

---

## 0. Fact-check results (verified before planning)

| Question | Verified answer | Consequence |
|---|---|---|
| Is the M75 MCU known? | **Yes** — platform chip is HFD2201KBA = rebranded **Sonix SN32F248BF** (Cortex-M0, 64KB flash, 8KB RAM, LQFP-64). Our keyboard enumerates as `VID 0C45 / PID 8006` in normal mode — same as the M87. Bootloader = `0C45:7040` masked-ROM ISP. | Not a "customized SOC". SonixQMK supports it natively. |
| Does QMK encoder support work on SN32? | **Yes** — `encoder_quadrature` driver + `ENCODER_MAP_ENABLE` work on `sn32_develop`. **Risk:** some OEM boards multiplex the encoder A/B into the switch matrix instead of dedicated GPIOs. Must trace the PCB. | Encoder pin tracing is a hard gate before firmware bring-up. |
| Does VIA save to the keyboard (not browser)? | **Yes** — VIA web (usevia.app, Chromium/WebHID) writes keymaps, macros, encoder modes to **onboard EEPROM** instantly. Draft JSON loads via the Design tab; no upstream repo merge required. | "Web-based memory flashing" = stock VIA behavior. We just need the VIA JSON definition. |
| Does SN32 have EEPROM? | **No hardware EEPROM** — must use `EEPROM_DRIVER = wear_leveling` + `WEAR_LEVELING_DRIVER = embedded_flash` (upper flash pages, 1–2KB logical). Without it, VIA settings are RAM-only and lost on unplug. The M87 port has `EEPROM_ENABLE = yes` — verify which driver it resolves to. | EEPROM verification is an explicit regression test. |
| Can Sonix Flasher back up stock firmware? | **NO** — neither Sonix Flasher nor SonixFlasherC can read flash back. The ISP protocol has no read command; stock firmware also enables readout protection. | ⚠️ No software backup of stock firmware is possible. Mitigations below. |
| Can a bad flash brick the keyboard? | **Practically no** — the ISP bootloader lives in mask ROM and is always reachable by shorting the **BOOT pin (P2.2, LQFP-64 pin 3) to GND** at power-up. Bad firmware → re-enter bootloader → re-flash. | Worst realistic case: keyboard temporarily unusable until re-flash, never permanently dead. |
| Wireless lost with QMK? | **No** (outdated claim) — the M87 port drives the WCH CH582F coprocessor (`ch582f.c`); BT 5.3 + 2.4G + wired all work. | We inherit tri-mode. |

**Backup reality (important):** the stock firmware `.bin` is *not* in the Xinmeng driver installer
(`app/firmware/` is empty) and cannot be dumped over USB. Before first flash we will:
1. Ask Xinmeng support (www.szxinmeng.com.cn) for the M75 stock firmware image; and/or
2. Search community dumps (SonixQMK Discord, GitHub) for the M75/M75-adjacent `0C45:8006` image; and/or
3. Proceed *without* stock backup only after explicit sign-off — recovery to QMK is always possible
   via BOOT-pin; recovery *to stock* is the only thing that needs a `.bin`.

---

## 1. Product definition — "XINMENG M75 DRIVER+"

### 1.1 Inherited from the M87 port (keep)
- Tri-mode: wired USB (native 500Hz), BT 5.3 (3 profiles), 2.4GHz dongle
- RGB matrix engine (SN32F2xx driver) + side strips / lightbar control
- VIA v3 support, dynamic keymaps, macros, bootmagic, NKRO
- Sonix Flasher flashing workflow (offset `0x00`)

### 1.2 Changed / added for the M75
1. **Rebrand**: repo + keyboard name `Xinmeng M75 Driver+`, `keyboard_name = "Xinmeng M75 Driver+"`,
   VID `0x0C45` / PID `0x8006` (unchanged — hardware identity), maintainer updated. Keep upstream
   attribution in README (GPL-2.0 obligation).
2. **75% layout (81 keys + knob)**: derive the physical layout from the stock driver's
   `layouts/rgb-keyboard.xml` (6 rows × 19 cols, all 81 keys with codes + pixel positions —
   we extracted this from the driver installer; it is our ground truth for the layout JSON).
3. **Knob mode engine** (see §2).
4. **Per-key macros everywhere**: every key — including the entire F-row, `Del`, `PgUp/PgDn`,
   navigation cluster — is macro-assignable via VIA. Requires ≥3 dynamic layers and enlarged
   VIA macro memory (`VIA_MACRO_NUM` / dynamic macro area sized within EEPROM budget).
5. **Onboard persistence** for keymaps, macros, knob modes, RGB config.

### 1.3 Non-goals (v1)
- TFT screen features (M75 non-screen model)
- Upstream merge into official QMK/VIA repos (optional, later)
- Re-implementing the Xinmeng proprietary driver (VIA web replaces it)

---

## 2. Knob mode engine (the headline feature)

### 2.1 UX design
| Gesture | Action |
|---|---|
| **Rotate** | Performs the *current mode's* action |
| **Knob tap** (< 400ms) | **Cycle to next mode** (wraps 1→9→1) |
| **Knob tap with Fn held** | Cycle to *previous* mode |
| **Knob hold** (> 600ms) | Legacy-safe default: Mute (configurable) |
| **`Fn + ↑` / `Fn + ↓`** | Also cycle modes (redundant input path) |

No 5-second holds anywhere. Mode changes are **instant** and **persisted** to EEPROM.

### 2.2 Modes (v1 ships 5, framework supports 9)
| # | Mode | CW | CCW | Notes |
|---|---|---|---|---|
| 1 | Volume | `KC_VOLU` | `KC_VOLD` | default |
| 2 | Brightness | `RM_VALU` | `RM_VALD` | replaces the stock 5s-hold toggle |
| 3 | Scroll | `KC_WH_U` | `KC_WH_D` | page/mouse-wheel scroll |
| 4 | Media | `KC_MNXT` | `KC_MPRV` | press = play/pause |
| 5 | Zoom | `KC_LCTL` + `KC_EQUAL` / `KC_MINUS` | | pinch-zoom substitute |
| 6–9 | **User slots** | any keycode/macro via VIA | | bound with `QK_USER` custom keycodes; assignable in VIA |

### 2.3 Mode indicator (RGB bar)
On every mode switch, the **top F-row LEDs render a progress bar**: mode N of M lights
`N` consecutive LEDs across the F-row (mode 3 of 9 → first 3 F-row LEDs bright, rest dim),
holds ~1.5s, then returns to the normal RGB effect. Implemented as a small
`rgb_matrix` hook (`housekeeping_task_kb` + one-shot animation state) — no extra hardware.

### 2.4 Implementation sketch
- `ENCODER_MAP_ENABLE = yes` with `ENCODERS_PAD_A/B` (pins from PCB trace, §4)
- Custom keycodes `KNB_MODE_NEXT`, `KNB_MODE_PREV`, `KNB_USER(n)` in `keymap.c`
- Mode table + active-mode byte stored in `EECONFIG` (via `user_eeprom` block / VIA `custom_config_value`)
- VIA custom menu (advanced keyboard definition JSON) exposes mode slots 6–9 as dropdowns/keycode pickers
- Encoder press: short vs long distinguishable in `matrix_scan_kb` timer logic

### 2.5 Known technical risks
| Risk | Likelihood | Mitigation |
|---|---|---|
| Encoder A/B multiplexed into key matrix (not dedicated GPIO) | Medium | Detect during PCB trace; fall back to `matrix_scan_kb`-based encoder read synchronized with the SN32 RGB/matrix timer ISR |
| Encoder press shares matrix row/col | Medium | Same as above — treat press as a matrix key in `keyboard.json` |
| Flash budget: 64KB total, wear-leveling reserves 2–4KB | Certain | `LTO_ENABLE = yes`, strip unused RGB animations, assert `.bin < 56KB` in CI (size check is skipped upstream for SN32F248BF — we add our own) |
| Wear-leveling flash writes stall RGB briefly | Certain (cosmetic) | Accept; writes are rare (mode change / VIA save only) |
| M75 wireless coprocessor firmware differs from M87's CH582F | Unknown | Phase 6; test wired-only first; wireless stays stock-untouched until then |

---

## 3. Safety & backup plan (anti-brick)

1. **No flash before Phase 4 sign-off.** All compile/UI work happens on PC.
2. **Stock firmware image** sourced per §0 before first flash (support ticket / community dump).
3. **Recovery documented & rehearsed**: BOOT (LQFP-64 pin 3) → GND at USB plug-in → `0C45:7040`
   appears → Sonix Flasher re-flash. This works with *any* flash content — the ISP ROM cannot be
   overwritten.
4. **First flash = M87-derived build in wired mode only**, wireless coprocessor untouched.
5. **Per-phase commit tags** — every flashable artifact is traceable to a git tag + CI build.
6. Physical BOOT-pin access verified (case opened, pin located, shorting tool ready) *before* the
   first flash attempt.

---

## 4. Hardware verification checklist (user-assisted, one-time)

- [ ] Open case (clip retention; photos at each step)
- [ ] Photograph PCB front/back; read MCU top marking (expect `HFD2201KBA` / `SN32F248B`)
- [ ] Locate BOOT test pad/pin near MCU
- [ ] Continuity-trace encoder A, B, and press pins → MCU pins (candidate pins: `P3.7–P3.9`,
      `P0.8/P0.9`, upper `P2.x`, or matrix-multiplexed)
- [ ] Verify matrix row/col count against driver XML (6 rows × 19 cols, COL2ROW expected)
- [ ] Count RGB LEDs (81 keys + side strips + logo lightbar)
- [ ] Record all findings in `docs/m75_hardware_notes.md`

## 5. Regression tests

### Automated (CI, `.github/workflows/build.yml`)
- `qmk compile` for `xinmeng_m75_driver_plus` keymaps `default` + `via` on every PR
- **Binary size gate**: fail if `.bin > 56,320 bytes` (protects wear-leveling flash region)
- keyboard.json validity (QMK's JSON schema check via `qmk generate-rgb-matrix-h` dry run)
- Keymap round-trip: `qmk json2c`/`c2json` idempotency on the default keymap

### Manual hardware gates (checklist doc, executed per milestone)
- [ ] M1: boots, types all 81 keys correctly (use keyboard tester), no ghosting
- [ ] M2: encoder rotate/tap detected (QMK console via `qmk console`)
- [ ] M3: knob modes cycle; RGB bar indicator shows correct N-of-M
- [ ] M4: VIA web detects keyboard, per-key remap + macro works, **unplug 10s → settings persist**
- [ ] M5: RGB effects + mode bar coexist; no flicker regression
- [ ] M6: BT pairing + 2.4G still functional after `ch582f` adaptation; battery report works
- [ ] M7: full flash → unplug → replug → bootmagic safe-test; BOOT-pin recovery rehearsal

## 6. Phases

| Phase | Deliverable | Depends on |
|---|---|---|
| **P0** ✅ | Fork renamed `xinmeng-m75-driver-plus`; PLAN.md; CI wired | — |
| **P1** | `keyboards/xinmeng_m75_driver_plus/` skeleton: `keyboard.json` (81-key layout from driver XML), default keymap, compiles in CI | P0 |
| **P2** | Hardware verification (§4) complete; pin map filled in | P1 + case opening |
| **P3** | Wired bring-up: matrix correct, RGB basic, safe to flash test | P2 |
| **P4** | Knob mode engine + RGB mode-bar indicator | P3 |
| **P5** | VIA definition (incl. custom knob-mode menu), wear-leveling EEPROM, persistence test | P4 |
| **P6** | Wireless: adapt `ch582f.c`, BT/2.4G bring-up | P5 |
| **P7** | Docs, release `.bin` artifacts, firmware-update guide, (optional) upstream PRs | P6 |

## 7. Immediate next actions

1. P1: generate `keyboard.json` from the extracted driver layout XML (offline, zero risk)
2. User: open the case & do §4 tracing (or decide to open later and gate P3)
3. User: email Xinmeng support requesting the M75 stock firmware `.bin`
