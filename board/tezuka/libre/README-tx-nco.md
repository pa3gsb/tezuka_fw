# LibreSDR fixed TCXO: automatic TX-NCO

`overlay/usr/sbin/libresdr-tx-nco` corrects the TX frequency using the FPGA NCO.
The script runs on BusyBox `/bin/sh`, `awk` and `devmem`. It does not write to
the TX-LO, sample rate, or `xo_correction`, and does not start any transmit
signal itself. There is no check for the old XO script; make sure yourself
that it is no longer adjusting things.

## Preparing the LibreSDR

1. Load the modified bitstream with NCO registers `0x43c00020` through
   `0x43c00028`. The bundled bitstream is not replaced by this change.
2. Use a fixed 40 MHz TCXO and connect the stable 10 MHz reference.
3. Make sure beforehand that `ad9361-phy/xo_correction` is set to `40000000`.
   Restoring it can interrupt the radio, so the controller does not do this
   itself.
4. Set `FPGA_NCO_SUPPORTED=1` in `/etc/libresdr-tx-nco.conf`. The FPGA has no
   capability register; this setting explicitly confirms the correct
   bitstream is present.
5. Check that `out_voltage_sampling_frequency` of `ad9361-phy` is the rate on
   the TX-NCO data path. For a different integration, you can set
   `TX_RATE_ATTR` to the correct converter rate attribute, after
   interpolation.

Start as root:

```sh
libresdr-tx-nco
```

This starts the background control loop and opens the terminal screen. `q`
only closes the screen; the control loop keeps running, even after closing
SSH. `d` stops the control loop and disables the NCO. The screen uses ANSI
codes and BusyBox ash `read -n -t`. A terminal of at least about 80 columns
is recommended.

```sh
libresdr-tx-nco start    # background, no screen
libresdr-tx-nco status  # one-off status
libresdr-tx-nco stop    # stop, last NCO correction stays active
libresdr-tx-nco disable # stop and disable the NCO
libresdr-tx-nco run     # controller in the foreground, for diagnostics
libresdr-tx-nco self-test # calculation test with the local awk, no hardware writes
```

`/run/libresdr-tx-nco/status` holds the latest status, `events` the last
eight events, and `daemon.log` startup errors. After stopping, the status is
a final snapshot. The PID/lock only prevents duplicate instances of this
controller. After a hard process kill, a leftover lock must be removed
manually, after checking that the corresponding PID is no longer running.

## Control behavior

- Every second: read reference status, TX-LO, and converter rate.
- By default every two seconds: sample the signed TCXO error. The FPGA
  provides no sequence counter, so individual new measurement windows cannot
  be provably distinguished. Two seconds is a conservative default value;
  identical or zero measurements are not wrongly flagged as faulty.
- After start or reference recovery, first wait two seconds and collect at
  least five samples. The window then grows to sixteen samples.
- The average of the sliding window determines the correction. With a
  spread above 5 Hz or an absolute measurement error above 10000 Hz, no new
  estimate is applied.
- From five samples: `TRACKING`; with a full valid window: `STABLE`. This
  status means a filled valid filter, not proven thermal stability or PLL
  lock.
- Only apply with at least a 2 Hz difference on the TX output, on first
  enable, or on a changed LO/rate. No five-minute wait time.
- Write new TX-FTW and RX-FTW=0, then toggle the apply bit. No phase reset.
  Register readback checks the written configuration, not independently the
  processing in the data path. On a write error, further writes stay
  blocked until a restart; the state may then be partially applied.
- Without a reference: `WAITING`, or `HOLDOVER` if a valid estimate exists.
  In holdover, the FTW is retained; only an LO/rate change recalculates it
  using the last valid TCXO error. After recovery, it waits for new valid
  samples.
- On a changed `xo_correction`, further applies are blocked until restart.

The formula uses `e = measured TCXO - 40000000`:

```text
actual_fs = tx_fs * (40000000 + e) / 40000000
shift     = -tx_lo * e / 40000000
tx_ftw    = round(shift * 2^32 / actual_fs)
```

The correction targets the TX center. An NCO does not restore the physical
sample rate: for signals away from the center, the small proportional
timing and frequency error remains. The internal AXI-DDS bypasses this NCO;
use the DMA/DVB path instead. RX stays untouched. The old analog lock bit is
not a requirement; `ref_present` is. Do not change the clock source/DAC
control via other software while this is in use.

## Firmware and boot

The Libre defconfig adds `board/tezuka/libre/overlay` after the shared
overlays from `board/tezuka/common`. The Libre post-build step sets the
execute permissions, including for builds from a Windows checkout. For an
existing Buildroot output, the updated defconfig must be reloaded before
building the firmware.

To start at boot: also set `AUTOSTART=1` in the configuration. `S95tx-nco`
starts the background control loop; open the screen later over SSH. Boot
start is off by default because Libre boards can also have other
oscillators/bitstreams. Change the configuration in the source overlay for
settings that should be part of the firmware image; changes in a volatile
rootfs do not survive a reboot.

For testing without a new firmware build, you can copy the script and the
configuration to `/tmp` on the board:

```sh
export NCO_CONFIG=/tmp/libresdr-tx-nco.conf
sh /tmp/libresdr-tx-nco
```

## Validation

From the repository root, in a POSIX environment:

```sh
sh board/tezuka/libre/tests/tx-nco-smoke.sh
```

The test uses a temporary fake register bank and IIO files. On hardware,
sign direction, correct TX rate, cold start, LO/rate changes, 10 MHz
loss/recovery, and phase continuity during transmission still need to be
verified. Use the same 10 MHz timebase for the RF measurement. The filter
and threshold are starting values; 2 Hz is an apply threshold, not a
guaranteed absolute accuracy.
