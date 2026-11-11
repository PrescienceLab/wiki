# Working with BittWare FPGA cards

A practical guide to the BittWorks II toolkit, flash, boot source, and Vivado
integration.

Written against a BittWare XUP-P3R (Xilinx Virtex UltraScale+ VU9P),
BittWorks II 2024.1, Vivado 2023.2. Also available as a typeset PDF:
[`bittware-fpga-guide.pdf`](bittware-fpga-guide.pdf).

---

## The one idea that explains most failures

**An FPGA's configuration is volatile. The board management controller is not.**

Almost every confusing symptom on these cards comes from the same place: the
fabric is blank while the BMC is perfectly healthy. That combination makes a
card look dead over PCIe and alive over USB at the same instant, which is
exactly the situation that sends people looking for hardware faults that are
not there.

Hold on to the consequences:

- A blank FPGA presents no PCIe endpoint, so the link never trains, so firmware
  hides the root port. The card is absent from `lspci` entirely, not merely
  driverless.
- The BMC answers over USB regardless of what the fabric contains. That is your
  diagnostic channel and your recovery channel.
- A configuration loaded by hand survives warm reboots while the slot keeps
  power. It disappears the first time power is actually cut, which is why a
  card can appear healthy for months and then vanish.

---

## 1. Getting the toolkit talking to the board

```bash
BW=/opt/bwtk/2024.1          # version varies; check: ls /opt/bwtk
export LD_LIBRARY_PATH=$BW/lib
cd $BW/bin
```

`bwconfig` reports *There are no mapped devices in your system* until you map
one. Do this at the start of every session, because it is not persistent in the
way you would expect:

```bash
./bwconfig --add=usb      # BMC over USB; works even with a blank FPGA
./bwconfig --add=pci      # needs the FPGA configured AND bwpci loaded
./bwconfig --list
```

**Prefer `--add=usb`.** The USB path reaches the board management controller,
an independent always-on microcontroller. It works no matter what the fabric
contains, which is precisely what you need when something is wrong. The PCI
path only works when the thing you are trying to diagnose is already healthy.

Useful queries:

```bash
./bwconfig --dev=0 --get=device                   # part number, serial, revision
./bwconfig --dev=0 --get=resource --type=fpga     # size, boot source
./bwconfig --dev=0 --get=resource --type=flash --index=N
./bwmonitor  --dev=0 --type=Sensor --read         # rails, currents, temperatures
./bwconfig --help=<scan|add|get|set|load|start|erase|reset>
```

---

## 2. When the card is missing from lspci

This is the most common and most alarming failure, and it is usually not a
hardware fault. Diagnose in this order rather than guessing.

### Step 1: is the board powered?

```bash
./bwmonitor --dev=0 --type=Sensor --read
```

If you get sensor readings at all, the BMC is alive. Check
`12v Connector Current`: near zero with voltage present means the auxiliary
cable is seated but the board is running on slot power alone, which is normal
for a blank FPGA.

### Step 2: is the fabric configured?

This is the decisive measurement, and it is far more reliable than any status
register.

| Sensor            | Blank                      | Configured   |
|-------------------|----------------------------|--------------|
| FPGA Core Current | about 3 A (static leakage) | 5 A or more  |
| FPGA Core Temp    | about 35 C                 | 40 C or more |

Watch for bus renumbering as corroboration. If devices that used to sit on bus
02 are now on bus 01, a bridge ahead of them disappeared.

### Step 3: check the boot source

```bash
./bwconfig --dev=0 --get=resource --type=fpga --prop=HIL_FPGA_BOOT_SOURCE
```

A value of `0` means **no boot source**: nothing loads at power-on and the FPGA
comes up blank after every cold boot. Setting a boot source is the real fix.
Reloading by hand only papers over it.

---

## 3. Boot source numbering is 1-based

This trips people up, and getting it wrong leaves the card unconfigured at
power-on. Flash resources are indexed from 0, but `BOOT_SOURCE` counts from 1.

| Flash resource | Label  | BOOT_SOURCE value |
|----------------|--------|-------------------|
| `--index=0`    | User   | 1                 |
| `--index=1`    | Backup | 2                 |
| no boot source |        | 0                 |

The authority for this is BittWare's own
`$BW/examples/load_fpga/load_fpga.c`, which fetches `HIL_RESOURCE_FLASH, 0`,
calls it *Flash partition 1* in its own error string, and sets
`HIL_FPGA_BOOT_SOURCE` to 1. On an unfamiliar board, read that file rather than
guessing.

```bash
./bwconfig --dev=0 --set=resource --type=fpga --index=0 \
           --prop=HIL_FPGA_BOOT_SOURCE --newvalue=1
```

Read it back to confirm it took, then confirm *functionally* with
`--start --type=fpga`, which loads from the boot source using the same path a
power-on uses. That turns "I set a property" into "the boot path works".

---

## 4. Loading your own bitstream

**Direct volatile load is not supported on these boards.** `--load --type=fpga`
returns `Not supported`. Flash is the only way in, which also matches how
BittWare's own example does it.

**The FPGA must be unconfigured before flash is writable**, because the
configured design owns the SPI pins. Skipping this gives:

```
The FPGA must be erased (unconfigured) before the Flash can be accessed.
```

Before erasing, remove the PCI device so the kernel is not touching the card
when the link drops:

```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000:XX:00.0/remove
```

Then the sequence itself:

```bash
./bwconfig --dev=0 --erase --type=fpga                  # frees the SPI pins
./bwconfig --dev=0 --load=my.bit --type=flash --index=1 # minutes over USB
./bwconfig --dev=0 --set=resource --type=fpga --index=0 \
           --prop=HIL_FPGA_BOOT_SOURCE --newvalue=2
```

**Keep a known-good image in the other partition.** Overwriting the one
partition that holds a working design, with no copy on disk, is how a board
becomes a paperweight. Recovery is always setting `BOOT_SOURCE` back to the
good partition over USB.

The recorded `HIL_FLASH_FILESIZE` will be about 127 bytes smaller than your
`.bit` file. That is the bitstream header being stripped, not corruption.

---

## 5. The two-step reboot, and why one reboot is not enough

This is the single most time-consuming trap in the whole workflow, and it
follows directly from the volatile-fabric idea at the top.

Flashing *erases* the FPGA to free the SPI pins. A warm `reboot` is not a power
cycle, so the card is still erased when the BIOS enumerates PCIe. There is
nothing to train against, the slot comes up empty, and the card looks bricked
even though the flash write succeeded perfectly.

**The sequence that works:**

```bash
# 1. flash (this leaves the FPGA erased)
./bwconfig --dev=0 --erase --type=fpga
./bwconfig --dev=0 --load=my.bit --type=flash --index=1
./bwconfig --dev=0 --set=resource --type=fpga --index=0 \
           --prop=HIL_FPGA_BOOT_SOURCE --newvalue=2

# 2. configure the fabric from flash, so there is something to enumerate
./bwconfig --dev=0 --start --type=flash --index=1
sleep 8
echo 1 | sudo tee /sys/bus/pci/rescan

# 3. NOW reboot, for a clean enumeration by firmware
sudo reboot
```

Step 2 is the one people skip. Without it you get a card that is absent from
`lspci` after a successful flash, which reads as a failed flash and sends you
chasing the wrong problem.

---

## 6. Verify you flashed what you think you flashed

Build a version register into your design and read it back after every flash.
Without one you cannot distinguish "my change did not work" from "my change is
not actually on the card".

A worked example: a control-plane register block exposing a magic value and a
version, read by the host driver at probe time.

```verilog
// in the RTL
localparam [31:0] VERSION = 32'h0001_0006;
```

```bash
# read back after flashing and rebooting
dmesg | grep -i "control plane"
  capio 0000:01:00.0: control plane ok: MAGIC=0x43415049 VERSION=0x00010006
```

Bump it on every behavioural change and have the driver warn on mismatch. This
single habit catches the failure mode in section 7.3, which is otherwise
invisible.

---

## 7. Vivado: the traps that cost the most time

### 7.1 Keep the vendor's top-level port names

The most expensive mistake available here is using a block-design wrapper as
the top-level module. Vivado's generated wrapper names ports after the BD's
externals (`sys_clk_0`, `pcie_mgt_0_rxn`), while the vendor's XDC references
*their* top-level names (`sys_clkp`, `sys_reset_l`). Every `get_ports` then
matches nothing, every constraint silently does not apply, and synthesis
reports success with the clock entirely unconstrained. Any timing number you
get is meaningless rather than failing.

Write your own board-level top with a port list identical to theirs and
instantiate your BD inside it. You also inherit board housekeeping that is not
optional:

```verilog
IBUFDS_GTE4 refclk_ibuf (.O(sys_clk_gt), .ODIV2(sys_clk),
                         .I(sys_clkp), .CEB(1'b0), .IB(sys_clkn));
assign QSFP_CTL_EN       = 1'b1;   // board requires this high to operate
assign FPGA_I2C_MASTER_L = 1'b0;
```

Verify afterwards, because the failure is silent:

```bash
grep -c "No ports matched" <run>/runme.log     # must be 0
grep -c "CRITICAL WARNING" <run>/runme.log
```

### 7.2 PCIe block placement is a board-wiring fact

`pcie_blk_locn` and `select_quad` come from the board, not from preference.
Read them out of the reference design's `.bd` file rather than guessing.

### 7.3 Out-of-context IP runs do not rebuild on their own

If your logic is packaged as a module-reference IP inside a block design, it is
synthesised *out of context* and cached. Resetting the top-level run does not
touch it:

```tcl
reset_run synth_1            # NOT enough: the IP's cached netlist survives
```

Implementation then stitches in the *old* netlist and produces a bitstream that
looks new and behaves old. The tell is suspicious: timing comes back
*identical* to the previous build, which is easy to read as a good sign.

Reset the IP's own run as well, and disable the IP cache so a stale hit cannot
be served:

```tcl
set_property IP_CACHE_PERMISSIONS {} [current_project]
config_ip_cache -disable_cache
reset_run <bd>_<your_ip>_0_synth_1
reset_run synth_1
launch_runs <bd>_<your_ip>_0_synth_1 -jobs 8
wait_on_run <bd>_<your_ip>_0_synth_1
launch_runs impl_1 -to_step write_bitstream -jobs 8
```

**Gate the build on evidence that your change is physically present**, rather
than trusting that it rebuilt:

```tcl
open_run impl_1
set n [llength [get_cells -hier -filter {NAME =~ *my_new_signal*}]]
puts "GATE: $n cells"
if {$n == 0} { puts "GATE FAILED: the change is NOT in the netlist" }
```

Corroborate with the netlist timestamp and the bitstream size, both of which
should change when the design does:

```bash
find . -name "*<your_ip>*.dcp" -printf "%Tc %p\n"
ls -la <run>/impl_1/*.bit
```

### 7.4 Reading synthesis results correctly

Report utilisation per block rather than for the whole design. On a card like
this the vendor's PCIe and DMA infrastructure dominates, and a whole-design
number tells you almost nothing about your own logic. Each out-of-context run
writes its own utilisation report:

```
<run>/<bd>_<ip>_0_synth_1/<bd>_<ip>_0_utilization_synth.rpt
```

Remember that the PCIe hard block and the transceivers are dedicated silicon.
They cost zero LUTs, so the fabric cost of adding logic next to a PCIe endpoint
is just your own logic.

---

## 8. Driver

`bwpcidrv.ko` is built against a specific kernel and silently stops matching
after a kernel upgrade. This is a frequent cause of "it worked last week":

```bash
modinfo $BW/drivers/bwpci/bwpcidrv.ko | grep vermagic
uname -r                                     # must match
```

Rebuild out of tree. The Makefile expects headers at `../../include`, so mirror
that layout rather than copying the driver alone:

```bash
cp -r $BW/include       ~/bw/include
cp -r $BW/drivers/bwpci ~/bw/drivers/bwpci
cd ~/bw/drivers/bwpci && make && sudo make install   # install also runs depmod
```

`bwpci.service` is usually already enabled; once the module is in
`/lib/modules`, it loads at boot on its own.

---

## 9. Identifying what is actually on a board

Decode the part number rather than trusting the numeric class property:

```bash
./bwconfig --dev=0 --get=device --prop=HIL_PART_NUMBER_STR
# XUPP3R-0U-F09VP2E-R4R4R4R4-S0X410150RS-6
#   |         |         |
#   |         |         four banks of DDR4
#   |         F09 = size-9 device, VP = Virtex UltraScale+, so VU9P
#   board type
```

Cross-check the board type against the `BT_*` enum in `$BW/include/hil.h`,
which carries a descriptive comment per board.

Be careful with properties: `HIL_FPGA_CLASS` is marked deprecated in
`$BW/include/resources/hil_fpga.h` and can return a value that contradicts the
part number. Its replacements (`HIL_FPGA_CLASS_STR`, `HIL_FPGA_GRADE_STR`,
`HIL_FLASH_DESIGNNAME_STR`) are documented but not implemented by all BMC
firmware, and return "could not resolve resource property". Treat the header as
a catalogue of what *might* exist, and the part number as truth.

Flash partitions often record the full build path of whatever was written,
which identifies the exact reference design and FPGA part for free:

```
/home/bite/bite/binaries/full/xupp3r/2.36.0.0.0/FPGA/vu9p/pcie_base_gen3x16/pcie_base.bit
```

---

## 10. Working safely on a shared machine

These cards usually live in lab servers other people use.

- Run `who` before rebooting, and prefer `shutdown -r` over `reboot` so
  logged-in users get a broadcast.
- Check for running jobs. On a Slurm node: `sinfo -N -n $(hostname -s)` and
  `squeue`.
- **Never write to a live device's doorbell or control registers to see what
  happens.** Ringing a transmit doorbell on an unconfigured queue can make a
  NIC fetch a descriptor from a garbage ring base. If the platform reports
  `_OSC: platform does not support AER`, there is no graceful recovery and the
  machine can reset. Reads are safe; writes are not.
- Long Vivado runs over SSH should be detached
  (`setsid nohup ... > log 2>&1 &`) and polled. A dropped session otherwise
  loses results from work that completed fine on the remote side.
- Restore anything you change system-wide. Moving IRQ affinities or process
  affinities to isolate a core is fine for a measurement, but put them back.

---

## Quick reference

| Goal               | Command                                                  |
|--------------------|----------------------------------------------------------|
| map the board      | `bwconfig --add=usb`                                     |
| is it alive        | `bwmonitor --dev=0 --type=Sensor --read`                 |
| is fabric loaded   | core current above 5 A, temp above 40 C                  |
| boot source        | `--get=resource --type=fpga --prop=HIL_FPGA_BOOT_SOURCE` |
| what is flashed    | `--get=resource --type=flash --index=1`                  |
| free the SPI pins  | `--erase --type=fpga`                                    |
| write flash        | `--load=my.bit --type=flash --index=1`                   |
| configure from it  | `--start --type=flash --index=1`                         |
| rescan the bus     | `echo 1 > /sys/bus/pci/rescan`                           |
| recover a bad load | set `BOOT_SOURCE` to the known-good partition            |

---

If a card vanishes from `lspci`: the fabric is almost certainly blank, the BMC
is almost certainly fine, and USB is the way back in.
