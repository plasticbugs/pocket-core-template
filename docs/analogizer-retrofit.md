# Adding the Analogizer to a core not built from the current template

A handoff for whoever, person or agent, adds
[Analogizer](https://github.com/RndMnkIII/Analogizer) support to an existing
Pocket core that predates it in this template. A core started from the
template today already has it; `docs/analogizer.md` describes that, and
should be read first. This file is the order of work and the traps, written
from adding it to Moo Mesa (2026-10; `plasticbugs/moomesa`, branch
`analogizer`, commit `72271e8`, a complete worked example of every step below)
and from the MSX2 core, where RndMnkIII and plasticbugs added it by hand
(2026-08).

The template is the source for every file named here. Copy from it rather
than from another core: the copies in other cores drift, and RndMnkIII keeps
editing his module per core (the copy in MiraxPocket carries two lines marked
"FIX ONLY FOR XAIN'D SLEENA").

## 0. Decide before touching anything

Answer these from the core's own source, and write the answers into the
commit message:

1. **The dot rate and the whole line.** Total dots per line, blanking
   included, at the core's pixel enable. That is `LINE_LEN`. Using the visible
   width is the trap: the MSX2 core first shipped with 290 on a 684-dot line,
   and the right half of every scandoubled line was lost.
2. **Real H and V blanking.** The scandoubler times every output line from
   the H-blank edges. If the core only has a combined DE, derive
   `hblank = !visible_x` and `vblank = !visible_y` from its counters. The MSX2
   hook first tied them to 0: syncs and blanking came out garbage in every
   scandoubled mode, and RGBS lost its blanking.
3. **The clock to give the Analogizer.** It clocks the DAC through the
   cartridge pins, so keep it near 48 MHz (cores ship at 42.95-48 MHz) and at
   least 4x the dot rate.
   - `clk_sys` is already 40-50 MHz: pass it as both `clk` and `clk_src`.
   - `clk_sys` is faster (this template's is 96 MHz): use a spare PLL output at
     `clk_sys / 2`, phase 0. Check which `outclk_N` is free in
     `core_pll_0002.v`.
   - Video on its own unrelated clock: `clk_src` is that clock and `clk` any
     ~48 MHz. The hand-over is a real synchroniser, but put the two in separate
     asynchronous clock groups in the SDC.
4. **The pixel's settling time after the dot enable.** That is `PIX_LAG`
   (default 2 source clocks). If the colour reaches its final register more
   than two clocks after the enable, raise it.
5. **Where the core reads controllers.** Every place that reads
   `cont1_key..cont4_key` must read `key1..key4` instead, including direct
   reads for players 3 and 4 that bypass the `gamepad` helper.
6. **Free resources.** On Moo Mesa (LINE_LEN 512) the Analogizer cost
   **1,550 ALMs, 13 RAM blocks and 2 DSP blocks** (265 → 278 of 308 blocks).
   The scandoubler's and HQ2x's line buffers scale with `LINE_LEN`. A core
   within ~15 blocks of 308 may not fit, so check the last fit summary before
   starting.
7. **Bridge address `0xF7xxxxxx` is free.** Grep the core for anything
   decoding it, including a `casex` with `32'hFxxxxxxx`. The settings file is
   written there.
8. **Does the core already have an Analogizer?** If it does (MSX2 does), this
   is a replacement. Keep its `interact`/file scheme decision (section 4) and
   its SNAC mapping until the user says otherwise.

## 1. Copy the files

From the template into the core, unchanged:

```
target/pocket/analogizer/            RndMnkIII's module, 15 files + our analogizer.qip
target/pocket/pocket_analogizer.sv   the wrapper
sim/ps2_keyboard_stub.v              Verilator stand-in for the one VHDL file
sim/tb_analogizer.sv                 the bench (set its raster, step 7)
sim/run_analogizer.sh
tools/check_json.py                  or merge its Analogizer block, if the core's copy differs
docs/analogizer.md                   then set the core's numbers in it
```

If the core's layout differs (`src/fpga/...`), put the two Verilog pieces
next to its `core_top` and adjust the paths in `run_analogizer.sh` and the
lint. Do **not** put the module under `modules/` if `tools/gen_qip.sh` or the
lint globs `modules/*`. It would be swept into every lint and into the qip
twice. Do **not** add the upstream `sync_fix.v` either: the top file already
defines `sync_fix`.

## 2. Quartus

- `core.qip` (or the qsf): add `pocket_analogizer.sv` and
  `QIP_FILE analogizer/analogizer.qip`.
- The PLL (if step 0.3 said so): set `output_clock_frequency<N>` to
  `"48.000000 MHz"`, phase 0, in `core_pll_0002.v`. Name it `clk_ana` where
  `core_top` takes the output.
- The SDC: add that output's `general[N].gpll~PLL_OUTPUT_COUNTER|divclk` to
  the core PLL's clock group. If the core's CI checks that every constraint
  applied, a name that matches nothing fails it, which is the point.

## 3. `core_top.sv`

Take the template's `core_top.sv` as the reference and search for
`USE_ANALOGIZER`, `key1`, `ana_bridge_rd_data`, `ana_pocket_off` and
`pocket_analogizer`. In order:

1. Parameter `USE_ANALOGIZER = 1`.
2. Wrap the core's "cart is unused" `assign`s to `cart_tran_*` in
   `generate if (USE_ANALOGIZER == 0)`. The wrapper drives the pins
   otherwise, and holds them at exactly those levels until the settings file
   enables it.
3. The bridge read mux: `32'hF7xxxxxx: bridge_rd_data <= ana_bridge_rd_data;`.
4. `wire [31:0] key1..key4`, and every `cont*_key` read changed to `key*`
   (step 0.5). Joystick and trigger words are unchanged.
5. Blank the Pocket screen: bring `ana_pocket_off` into the video clock
   domain with two flops and zero the RGB *going to the Pocket's scaler only*.
   The Analogizer is fed from before that point.
6. The instance, inside `generate if (USE_ANALOGIZER == 1)`, with the
   `else` branch tying `key* = cont*_key`, `ana_bridge_rd_data = 0` and
   `ana_pocket_off = 0`. Connect:
   - `clk`, `CLK_HZ`, `clk_src`: step 0.3. `LINE_LEN`: step 0.1.
   - `rst`: the core's reset, held through the ROM load. Any domain.
   - `src_pix_ce`, `src_rgb`, `src_hs/vs/hb/vb`: the core's dot enable and
     the picture **as the Pocket gets it, before any scaler-side blanking**.
     Syncs and blanks active high; the module corrects sync polarity itself,
     but the blanks must be active high. Use the bring-up panel's output if
     the core has one, so the panel can be read off a CRT.
   - The bridge signals, `cont1..4_key`, and every `cart_*` pin straight
     through.

## 4. Settings: the file, not the menu

The Analogizer wiki's specifications page shows settings in `interact.json`
at `0xA0000000`. That is the old scheme. The module this template vendors
(1.4) reads only `analogizer.bin` at `0xF7000000`, written by Pupdate and
AnalogizerConfigurator and shared by every core on the card. Bit 5 is a
master enable, which the menu scheme never set, so menu entries would leave
it off. Do not add Analogizer entries to `interact.json`. The bit layout is
in `docs/analogizer.md`.

## 5. The package

- `core.json`: `"platform_ids": ["<shortname>", "analogizer"]`, with the
  core's own platform **first** (tools and the Pocket's library use index 0),
  and `"cartridge_adapter": 0`.
- `data.json`: a slot named "Analogizer config", an unused id (10 by
  convention), `"filename": "analogizer.bin"`, `"address": "0xF7000000"`,
  `"parameters": "0x1000000"`, `"required": false`, `"nonvolatile": false`.
  Bits 25:24 of `parameters` are the index into `platform_ids` of the folder
  the file is read from. `0x1000000` means index 1, `analogizer`, so it must
  be second. Adding a slot at the end moves no other slot id. If you insert
  one in the middle, every slot id in `core_top` moves with it.
- `info.txt`: one paragraph. The file goes in `Assets/analogizer/common/`,
  and the cartridge slot is powered, so remove any game cartridge first.
- `tools/check_json.py pkg/pocket --active WxH` fails if the slot, the
  platform index and the cart power disagree.

## 6. Credit

`CREDITS.md`, the README and `modules/VENDOR.md` (or the core's equivalent):
RndMnkIII for the adapter and module, with the authors his module carries:
Mike Simone (Y/C), Sorgelig and Ludvig Strigeus (scandoubler, HQ2x), and the
SNAC controller code he credits. Copy the template's rows and change nothing
but paths.

## 7. Prove it, in this order

1. `sim/lint.sh` with the Analogizer block from the template's `lint.sh`.
2. `sim/run_analogizer.sh`, after setting the raster at the top of
   `sim/tb_analogizer.sv` to the core's: system clocks per dot, totals,
   visible window, sync positions, `CLK_HZ`. It must print `PASS`, and the
   RGBS line must show `0 differ`, every visible dot and line, and the
   scandoubler at exactly twice the lines and half the line period. A
   scandoubler line count off by a few means `LINE_LEN` or the blanking is
   wrong (step 0.1, 0.2).
3. The core's own gates, unchanged: frozen states, memory, system bench.
   With no settings file nothing in the machine changed, and they show that.
4. `./build-local.sh compile` (or the core's build). Read the fit summary for
   RAM blocks and ALMs, and the timing summary for every corner. Put both,
   before and after, in the commit message. **Expect the SDRAM capture to
   move.** A new PLL output and 1,500 more ALMs reshuffle the fit, and
   `dram_dq -> dq_in` hold is the first thing to go negative. It did on the
   template's skeleton: -0.058 ns at fast 0C, fixed by re-balancing the
   SDRAM clock phase as the SDC describes (METHODOLOGY 5.20), not by touching
   the Analogizer. Moo Mesa kept +0.080 ns at the old phase.
5. On hardware, `docs/bringup.md`'s "With an Analogizer" order: first with
   no file (the core must be exactly as before), then RGBS, then each mode,
   then SNAC.

## 8. Commit message

Say what was measured (the bench's lines, lint, the fit and timing numbers)
and what was **not**: no hardware, no CRT, SNAC protocols only as the
adapter's module implements them. If the core has no maintainer with an
adapter, the README says so too.
