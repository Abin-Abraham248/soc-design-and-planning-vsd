# soc-design-and-planning-vsd
RTL-to-GDSII physical design flow using OpenLANE &amp; Sky130 PDK | VSD SoC Design and Planning Workshop.
# Digital VLSI SoC Design and Planning — RTL to GDSII

> A 2-week hands-on workshop on complete RTL-to-GDSII flow for digital VLSI SoC design,
> organised by **VSD (VLSI System Design)** in collaboration with **NASSCOM**.
> This repository documents my learning, lab outputs, and key takeaways from each day.

## Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK

#### Understanding the Chip Package

When we look at any embedded board and point to what we call the "chip," we're actually looking at the **package** — a protective casing around the actual silicon die. The real chip sits in the centre of this package and communicates with the outside world via **wire bonding** — tiny wires that connect the chip's pads to the package pins.

#### Inside the Chip: Core, Pads, and Die

Zooming into the chip itself, all signals between the chip and the external world pass through **pads** placed around the periphery. The region enclosed by the pads is called the **core** — this is where all the actual digital logic lives. Together, the core and the pads form the **die**, which is the fundamental unit of chip manufacturing.

- **Foundry** — the place where chips are physically manufactured
- **Foundry IPs** — IP blocks that require specialized process knowledge to implement (e.g., PLLs, SRAMs)
- **Macros** — reusable, purely digital logic blocks

#### From Software to Silicon — The ISA Bridge

A C program running on a chip goes through a multi-layer transformation:

1. The C code is compiled into **RISC-V assembly** (or another ISA)
2. The assembler converts it to **binary machine code (0s and 1s)**
3. This binary pattern needs an **RTL implementation** of the ISA
4. The RTL gets synthesized and goes through the full **PnR (Place and Route)** flow to become a physical layout

The system software stack (OS → Compiler → Assembler) acts as the bridge between what the programmer writes and what the hardware executes.

#### Why Open-Source EDA Matters

For a fully open-source ASIC design flow, three things are needed:

1. **RTL Designs** (e.g., from opencores.org)
2. **EDA Tools** (synthesis, P&R, verification)
3. **PDK Data** (process-specific design rules, standard cell libraries)

Historically, PDKs were proprietary and distributed only under NDAs, making chip design inaccessible to most people. This changed in **June 2020**, when Google collaborated with SkyWater Technology to release the **Sky130 PDK** as the world's first open-source process design kit — a massive milestone for the VLSI community.

#### OpenLANE and the Automated RTL to GDSII Flow

**OpenLANE** is an open-source flow built on top of multiple EDA tools that automates the journey from an RTL netlist all the way to the final GDSII layout file. It uses:

| Stage | Tool(s) Used |
|---|---|
| Synthesis | Yosys, ABC |
| Floorplan & PDN | OpenROAD |
| Placement | OpenROAD |
| CTS | TritonCTS |
| Routing | FastRoute, TritonRoute |
| SPEF Extraction | OpenRCX |
| GDS Streaming | Magic, KLayout |
| Timing Analysis | OpenSTA |
| DRC & LVS | Magic, Netgen |

### Lab — Running OpenLANE for `picorv32a`

#### Setting Up and Invoking OpenLANE

The very first step is to navigate to the OpenLANE working directory and launch the tool in **interactive mode**, which lets us run each stage step-by-step.

```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
```

![image alt](image/01-synth1.png))

#### Preparing the Design

Before running synthesis, we prepare the design to merge the cell LEF and technology LEF files, and set up the run directory.

```tcl
prep -design picorv32a
```

![image alt](image/02-synth2.png) -->

#### Running Synthesis

```tcl
run_synthesis
```
![image alt](image/03-synth3.png)
![image alt](image/04-synth4.png)

After synthesis completes, we can calculate the **flop ratio** — a useful sanity check:

```
Flop Ratio = (No. of D Flip-Flops) / (Total No. of Cells)
           = 1613 / 14876
           ≈ 0.108  →  ~10.8%
```
## Day 2 — Floorplanning and Introduction to Library Cells

#### Chip Floorplanning — Core Area and Utilisation

Floorplanning is about deciding where everything goes on the chip. Two key parameters drive this:

- **Utilisation Factor** = (Area occupied by Netlist) / (Total Core Area)
  - A utilisation of 0.5–0.6 is typical — you want room for buffers, routing, etc.
- **Aspect Ratio** = Height / Width of the core
  - A ratio of 1 means a square; anything else is a rectangle.

#### Pre-Placed Cells and Decoupling Capacitors

**Pre-placed cells** (like memories, PLLs, and complex IP blocks) are fixed in position before automated placement runs. Their location is determined manually based on connectivity and power intent.

**Decoupling capacitors** are placed around pre-placed cells to act as local charge reservoirs — they compensate for voltage drops caused by switching activity and ensure these blocks see clean power.

#### Power Planning — Mesh vs Ring

A good power grid uses both **power rings** around the core and a **power mesh** across the chip. Multiple VDD and VSS rails are distributed in both metal layers so that every standard cell has a nearby power tap, minimising IR drop and electromigration risk.

#### Pin Placement and Logical Cell Blockage

Input and output pins are placed along the chip boundary. The relative placement of pins is guided by connectivity — a pin that drives logic deep in the core should be closer to that logic. The area between the core and the die boundary (I/O ring area) is blocked from automated cell placement to reserve it for pin buffers and ESD cells.

### Lab — Floorplan and Placement

#### Running Floorplan

```tcl
run_floorplan
```
![image alt](image/05-floorpl1.png)

After this completes, we can inspect the DEF file that was generated:

```bash
cd results/floorplan/
less picorv32a.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read picorv32a.def &
```
![image alt](image/06-floorpl2.png)

![image alt](image/07-floorpln3.png)

#### Running Placement

```tcl
run_placement
```
![image alt](image/08-placmnt1.png)

![image alt](image/08-placmnt1.png)

Standard cells legally placed

![image alt](image/10-plcmnt3.png)

## Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

#### CMOS Inverter — SPICE Deck

To characterise a standard cell, we write a SPICE netlist describing the PMOS and NMOS transistors along with their W/L ratios, supply voltage, input stimulus, and load capacitance.

Key parameters we extract from simulation:

- **Rise time** — 20% to 80% of output rising edge
- **Fall time** — 80% to 20% of output falling edge
- **Propagation delay** — 50% input to 50% output

#### 16-Mask CMOS Fabrication Process (Brief Overview)

The chip fabrication follows a sequence of about 16 mask steps:

1. Substrate selection (p-type, high resistivity)
2. Active region creation (field oxidation + Si3N4 mask)
3. N-well and P-well formation (ion implantation)
4. Gate oxide growth
5. Polysilicon gate deposition
6. Source/Drain implantation (LDD + halo)
7. Contacts and metal layers
8. Final passivation

### Lab — Cloning and Characterising a Custom Inverter Cell

#### Cloning the Standard Cell Repository

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
```

```bash
magic -T sky130A.tech sky130_inv.mag &
```
![image alt](image/11-inv1.png)
![image alt](image/12-inv2.png)



#### Extracting SPICE Netlist from Magic

Inside the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```
![image alt](image/13-inv3.png)
Screenshot of created spice file
![image alt](image/14-inv4.png)

Editing the spice model file for analysis through simulation.

Measuring unit distance in layout grid
Final edited spice file ready for ngspice simulation
![image alt](image/15-inv6.png)
#### Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```

```ngspice
plot y vs time a
```
![image alt](image/16-inv6.png)
Screenshot of generated plot
![image alt](image/17-inv7.png)
From the waveform, measure rise time, fall time, and propagation delay values.
Rise transition time calculation

Rise transition time = Time taken for output to rise to 80% - Time taken for output to rise to 20%

20% of output = 660 mV

80% of output = 2.64 V
Fall transition time calculation

Fall transition time = Time taken for output to fall to 20% - Time taken for output to fall to 80%

20% of output = 660 mV

80% of output = 2.64 V

![image alt](image/18-inv8.png)
![image alt](image/19-inv9.png)
![image alt](image/20-inv10.png)
![image alt](image/21-inv11.png)
![image alt](image/22-inv12.png)
![image alt](image/23-inv13.png)


Incorrectly implemented poly.9 rule no drc violation even though spacing < 0.48u
Find problem in the DRC section of the old magic tech file for the skywater process and fix them.

Link to Sky130 Periphery rules: https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html

![image alt](image/24-inv14.png)

![image alt](image/25-inv15.png)

![image alt](image/26-inv16.png)

![image alt](image/27-inv17.png)


## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

#### LEF Files and Guidelines for Standard Cell Ports

Before a custom cell can be used inside OpenLANE, it needs a proper **LEF file** describing its physical boundary, pin locations, and metal layer information. Port definitions must follow two important rules:

- All input and output ports must lie on the **intersection of horizontal and vertical routing tracks**
- The cell **width must be an odd multiple** of the track pitch, and height must be an odd multiple of the vertical track pitch

#### Static Timing Analysis (STA) Concepts

**Setup slack** = Data Required Time − Data Arrival Time (must be ≥ 0)

Key sources of uncertainty accounted for in STA:

- **OCV (On-Chip Variation)** — process/voltage/temperature variation modelled using derate factors
- **Clock Uncertainty** — jitter and skew margins added to timing paths
- **CRPR (Clock Reconvergence Pessimism Removal)** — removes artificial pessimism when launch and capture paths share clock buffers

#### Clock Tree Synthesis (CTS)

CTS builds a balanced tree of clock buffers to distribute the clock signal across the chip with minimal skew. After CTS:

- Hold timing must be re-checked (CTS inserts buffers that add real delay)
- Setup timing should be re-verified post-CTS as clock paths have changed

### Lab — Custom Cell Integration and STA with OpenSTA
Screenshot of tracks.info of sky130_fd_sc_hd
![image alt](image/28-inv18.png)
#### Get syntax for grid command
help grid

#### Set grid values accordingly
grid 0.46um 0.34um 0.23um 0.17um
![image alt](image/28-inv18.png)
![image alt](image/30-inv20.png)
Generate lef from the layout.

Command for tkcon window to write lef
![image alt](image/31-inv21.png)

Screenshot of newly created lef file
![image alt](image/32-inv22.png)

Copy the newly generated lef and associated required lib files to 'picorv32a' design 'src' directory.

Commands to copy necessary files to 'picorv32a' design 'src' directory
![image alt](image/33-inv23.png)

#### Editing `config.tcl` to Include Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```
![image alt](image/34-inv24.png)
![image alt](image/35-inv25.png)
![image alt](image/36-ivn26.png)
![image alt](image/37-inv27.png)

#### Running OpenSTA (Pre-CTS Timing)
Newly created pre_sta.conf for STA analysis in openlane directory
Newly created my_base.sdc for STA analysis in openlane/designs/picorv32a/src directory based on the file openlane/scripts/base.sdc

```bash
sta pre_sta.conf
```

#### Running CTS

```tcl
run_cts
```


#### Command to run OpenROAD tool
openroad

Reading lef file:
read_lef /OpenLane/designs/picorv32a/runs/24-03_10-03/tmp/merged.nom.lef

Reading def file:
read_def /OpenLane/designs/picorv32a/runs/24-03_10-03/results/cts/picorv32a.def

Creating an OpenROAD database to work with:
write_db pico_cts.db

Loading the created database in OpenROAD:
read_db pico_cts.db

Read netlist post CTS:
read_verilog /OpenLane/designs/picorv32a/runs/24-03_10-03/results/synthesis/picorv32a.v

Read library for design:
read_liberty $::env(LIB_SYNTH_COMPLETE)

Link design and library:
link_design picorv32a

Read in the custom sdc we created:
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc

Setting all cloks as propagated clocks:
set_propagated_clock [all_clocks]

Check syntax of 'report_checks' command:
help report_checks

Generating custom timing report:
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4



Exit to OpenLANE flow
exit
---

## Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA

#### Routing — Global vs Detailed

Routing happens in two stages:

1. **Global Routing (FastRoute)** — divides the chip into routing regions and finds approximate paths for each net, respecting layer and congestion constraints
2. **Detailed Routing (TritonRoute)** — takes the global routing guides and assigns exact wire segments, vias, and metal tracks while adhering to DRC rules

#### SPEF and Post-Route STA

After routing, parasitics (resistance and capacitance of actual wires) are extracted into a **SPEF (Standard Parasitic Exchange Format)** file. These parasitics are then back-annotated into the netlist and STA is re-run for final sign-off timing.


### Lab — Power Distribution, Routing

#### Generating Power Distribution Network

```tcl
gen_pdn
```
#### Running Routing

```tcl
run_routing
```





Common violations to look out for:

- *Min spacing violations* – two wires too close on the same layer
- *Antenna violations* – long metal segments accumulating charge during etch (can damage gate oxide)
  - Fix: insert antenna diodes or use jumper vias to a higher layer

-----

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, Placement, CTS, Routing |
| **Magic** | Layout editor, DRC, LVS |
| **OpenSTA** | Static Timing Analysis |
| **ngspice** | SPICE simulation |
| **TritonRoute** | Detailed routing |
| **Netgen** | LVS (Layout vs Schematic) |
| **Sky130 PDK** | SkyWater 130nm open-source PDK |

---

## Key Learnings

- Understood how a chip moves from an idea (RTL) to a manufacturable file (GDSII) using a fully open-source toolchain
- Got hands-on with floorplanning, placement, CTS, and routing for the `picorv32a` RISC-V core
- Learned how to characterise custom standard cells and integrate them into an existing flow
- Gained practical experience with STA concepts — setup/hold slack, OCV, CRPR — using OpenSTA
- Understood how parasitics from post-route SPEF extraction affect timing sign-off

---


## Acknowledgements

A huge thank you to *Kunal Ghosh* (Co-founder, VSD Corp. Pvt. Ltd.) and *Nickson P Jose* (Physical Design Engineer, Intel) for putting together such a well-structured and genuinely practical workshop. Running a real CPU from RTL to GDSII using nothing but open-source tools is something I didn’t expect to be possible — and yet here we are.
- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design)
- **Nickson Jose** — for the `vsdstdcelldesign` repository used in Day 3 labs
- **NASSCOM** — for facilitating this workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)


Documented by Abin Abraham
