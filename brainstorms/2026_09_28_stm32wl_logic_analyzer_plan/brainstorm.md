# NUCLEO-WL55JC1 + 16-channel logic analyzer - wiring plan for 2026-09-29

**Context:** the user has a 16-channel USB logic analyzer ("DL16" -
generic name, almost certainly one of the common Cypress FX2LP-based
16CH clones that `sigrok`/`PulseView`'s `fx2lafw` driver supports; if
it turns out to need vendor software instead, the channel/signal plan
below still applies, only the capture tool changes). Plan for
2026-09-29: connect the NUCLEO-WL55JC1 (and the logic analyzer, if
useful) at home without the laptop present during the day. This note
is the wiring plan prepared in advance, written up per the user's
request so the day isn't wasted deciding what to probe on the spot.

**This is a plan, not a verified result** - none of the pin
assignments below have been probed on real hardware in this session,
per `CLAUDE.md` §5. They're read directly out of ST's real source
(high confidence on peripheral/pin *names*), not guessed from general
Nucleo-64 knowledge - but the exact physical unit's behavior is still
unconfirmed until actually captured.

## Priority 1 - settle the AT-command UART question before it's too late

This is the one genuinely time-critical item. Root `ROADMAP.md`'s
"STM32WL 'ipari' eszköz" section already flags this as open ("a board
egyetlen USB-portján... nem jött AT-parancsos kimenet... Van logikai
analizátor... szándékosan elhalasztva későbbre") and separately warns
that **once this unit is reflashed with `base_node`/`otaa/*`, the
factory firmware's live UART/AT-command behavior can no longer be
tested on it.** If tomorrow's plan includes flashing the new firmware,
probing the factory firmware's UART has to happen *first* - it's a
one-shot opportunity, not something to redo later.

### What we actually know now, that we didn't when this was first flagged

Checked directly against ST's own reference source this session
(not guessed):

- The vendored `LoRaWAN_End_Node_LBM` example (now `base_node/` in this
  repo) configures **USART2 on PA2 (TX) / PA3 (RX)** - see
  `Core/Src/usart.c`.
- ST's own **`SubGHz_Phy_AT_Slave`** application for this exact board
  family (the one already identified as matching this unit's factory
  firmware, `AT+VL` code match, see
  `Documentation/lab_notes/2026_09_19_stm32wl_firmware_identification.md`)
  configures **LPUART1 on the *same physical pins*, PA2 (TX) / PA3
  (RX)**, at **9600 baud**. Checked in ST's public `STM32CubeWL` repo,
  `Projects/NUCLEO-WL55JC/Applications/SubGHz_Phy/SubGHz_Phy_AT_Slave/Core/Src/usart.c`.
- The earlier attempt (`2026_09_09_stm32wl_industrial_first_look.md`)
  tried 9600/19200/57600/115200 on "the board's own USB port (onboard
  ST-Link)" and got nothing. That's the ST-Link's **USB-CDC virtual COM
  port**, which bridges to these same PA2/PA3 pins through the
  on-board ST-Link chip - so the previous attempt should have worked if
  the firmware really is stock `SubGHz_Phy_AT_Slave` at 9600 baud.

**This mismatch is exactly what the logic analyzer should resolve.**
Probing PA2 directly with the logic analyzer bypasses the ST-Link's USB
bridge entirely - if there's real UART activity on the pin at boot/reset
but the VCP still shows nothing, the ST-Link bridge (or the terminal
settings used against it) is the problem, not the firmware. If there's
*no* activity on the pin either, the firmware genuinely isn't using
PA2/PA3, and the search moves to other candidates (below).

### Channel assignment - Priority 1 (2 channels, doubled up for reliability)

| LA channel | Signal | Where |
|---|---|---|
| D0 | PA2 (LPUART1_TX / USART2_TX) | CN10 morpho header - pin is silkscreen-labeled "PA2" directly on the board |
| D1 | PA3 (LPUART1_RX / USART2_RX) | CN10, silkscreen-labeled "PA3" |
| D2 | PA2 again (second clip, different contact point if possible) | redundancy in case D0's clip has a bad connection - this capture can't be redone once the board is reflashed |
| D3 | PA3 again (second clip) | same reasoning |

Capture settings: 200 kHz sample rate is already >20x oversampled for
9600 baud (plenty of margin even if the real baud turns out to be
higher) and keeps hours of capture small. UART idle-high framing means
long idle stretches compress well in `sigrok`'s session file format, so
sample rate is the thing to keep modest, not capture duration.

```bash
# Start a capture, then power-cycle or reset the board a few seconds in
# (the AT-slave app's boot banner, if any, only happens once at reset -
# an all-day blind capture isn't necessary, a short one bracketing a
# manual reset is).
sigrok-cli --driver fx2lafw --config samplerate=200k \
  --channels D0=PA2,D1=PA3,D2=PA2b,D3=PA3b \
  --time 5m -o /home/nagyszabolcs/stm32wl_uart_hunt.sr
```

(Exact channel-naming syntax depends on what `sigrok-cli --driver
fx2lafw --show` reports for this specific device - run that first to
confirm channel count/names match a 16-channel fx2lafw device before
trusting the command above verbatim.)

Decode offline in PulseView (`File > Open`, then add a UART decoder on
D0/D1, try 9600 first) whenever the laptop is available again - doesn't
need to happen live.

## Priority 2 - backup UART candidates, in case PA2/PA3 is genuinely silent

**Lower confidence - not verified against ST source, listed only as a
hedge given 12 spare channels.** If Priority 1 shows no UART activity
at all on PA2/PA3, this unit's firmware may differ from mainline
`SubGHz_Phy_AT_Slave` (the existing lab notes already hedge this
possibility - "korábbi verzió vagy egyedi kiterjesztés lehet"). Rather
than guess a second specific pin pair blind, use spare channels to
probe a handful of other commonly-broken-out morpho-header GPIOs
(**physically locate these from the silkscreen labels on the board
itself, not from a number guessed here**) and decode all of them
offline - this is the actual advantage of having 16 channels: cast a
wide net once, since re-probing later isn't an option.

| LA channel | Signal | Note |
|---|---|---|
| D4–D7 | 4 more morpho-header GPIO pins, picked by eye from the board's silkscreen (avoid ones already known to be SWD/radio-related) | decode as UART @ 9600/19200/57600/115200 offline, one at a time |

## Priority 3 - SWD, for timing correlation

The prior investigation already found the factory firmware disables
the debug port *at runtime* (software-level, not RDP) - see
`2026_09_09_stm32wl_industrial_first_look.md`. Capturing SWD alongside
UART lets you see *when* in the boot sequence that disable happens
relative to any UART output, which the ST-Link-only approach used
previously couldn't show.

| LA channel | Signal | Where |
|---|---|---|
| D8 | SWCLK | CN2 SWD header (or morpho, both broken out on Nucleo boards) |
| D9 | SWDIO | CN2 |

## Priority 4 - base_node bring-up, if reflashing happens the same day

Only relevant *after* Priority 1 is done and the board is reflashed.
`base_node`'s vendored `lora_app.c` drives `LED1` (blue, RX indicator)
and `LED3` (red, join-status) via GPIO - see the `SMTC_MODEM_EVENT_*`
handlers. Probing these lets join success/failure be read off the
capture later without needing a live serial terminal watching in real
time.

| LA channel | Signal | Where |
|---|---|---|
| D10 | PB15 (`LED1_Pin`/`LED1_GPIO_Port`, blue) | confirmed from `Core/Inc/main.h` |
| D11 | PB11 (`LED3_Pin`/`LED3_GPIO_Port`, red) | confirmed from `Core/Inc/main.h` |

Remaining channels (D12–D15) are spare - either leave unconnected or
double up a Priority-1/3 signal a third time if clip count allows.

## What this session can and can't do while the user is away

**Clarified 2026-09-28: the device and logic analyzer will be connected
to a *different* laptop at home, not the one this session runs on.**
This session was confirmed to run on `t580` (checked via `hostname` +
`lsusb` - real machine, not a sandbox), so in principle it *could* drive
already-wired hardware attached to *that* machine (start/stop
`sigrok-cli` captures, trigger an ST-Link reset, etc. - none of that
needs hands once the physical wiring is done). But none of that
transfers to a separate machine this session has no access to.

So, concretely, for 2026-09-29:

- The physical wiring (attaching probe leads to PA2/PA3/SWD/LEDs) needs
  hands regardless of which machine is involved - no way around that.
- Starting/stopping the capture and triggering resets on the home
  laptop needs either the user doing it manually there, or a Claude
  Code session actually running *on that machine* (not this one) if
  autonomous driving is wanted - that would need to be set up and
  confirmed separately, not assumed from this note.
- Once a capture file exists (e.g. the `.sr` file from the command
  above) and is reachable from a session with access to it - this one
  once the laptop is back, or a session on the home machine - decoding
  and analysis can happen then. That part doesn't need to be live.
