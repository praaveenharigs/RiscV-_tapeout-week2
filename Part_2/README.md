
## Cloning the VSDBabySoC Repository:
Before cloning the repository, choose a directory where you wants to store your works.
```bash
cd ~/vcd/photos
```

Clone the required sources from the [VSDBabySoC Repository](https://github.com/manili/VSDBabySoC):

```bash
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC
ls
```
`ls` used for to list out the files inside the `VSDBabySoC folder` and verify its contents.

---

### Analyse the contents of VSDBabySoC

After cloning the repository, it is important to understand the structure of the files and directories.
This helps in navigating the design, identifying modules, and preparing for simulation.

---

### TLV to Verilog Conversion for RVMYTH

Initially, you will see only the rvmyth.tlv file inside src/module/, since the RVMYTH core is written in TL-Verilog.

To convert it into a .v file for simulation, follow the steps below:

#### TLV to Verilog Conversion Steps
```bash
# Step 1: Install python3-venv (if not already installed)
sudo apt update
sudo apt install python3-venv python3-pip

# Step 2: Create and activate a virtual environment
cd ~/vcd/photos/VSDBabySoC/
python3 -m venv sp_env
source sp_env/bin/activate

# Step 3: Install SandPiper-SaaS inside the virtual environment
pip install pyyaml click sandpiper-saas

# Step 4: Convert rvmyth.tlv to Verilog
sandpiper-saas -i ./src/module/*.tlv -o rvmyth.v --bestsv --noline -p verilog --outdir ./src/module/
```

After running the above command, rvmyth.v will be generated in the src/module/ directory.

You can confirm this by listing the files:
```
cd ~/vcd/photos/VSDBabySoC
ls src/module
```
#### Note
To use this environment in future sessions, always activate it first:
```bash
source sp_env/bin/activate
```
To deactivate
```bash
deactivate
```

--- 

## Simulation Steps
### Pre-Synthesis Simulation
Run the following command to perform a pre-synthesis simulation:
```bash
mkdir -p output/pre_synth_sim
iverilog -o ~/vcd/photos/VSDBabySoC/output/pre_synth_sim/pre_synth_sim.out -DPRE_SYNTH_SIM -I ~/vcd/photos/VSDBabySoC/src/include -I ~/vcd/photos/VSDBabySoC/src/module ~/vcd/photos/VSDBabySoC/src/module/testbench.v
```
then run,
```bash
cd output/pre_synth_sim
./pre_synth_sim.out
```

#### Viewing Waveform in GTKWave
After running the simulation, open the VCD file in GTKWave:
```bash
cd ~/vcd/photos/VSDBabySoC
gtkwave output/pre_synth_sim/pre_synth_sim.vcd
```
<img width="1812" height="837" alt="image" src="https://github.com/user-attachments/assets/b8363d39-910e-4769-904d-ec28294e7dd4" />

---

### Verilog Source Files (*.v):
These files hold the RTL implementation of the BabySoC, defining its core modules and functional behavior.

`avsddac.v` – Implements the DAC (Digital-to-Analog Converter) module of the SoC.
```bash
module avsddac (
   OUT,
   D,
   VREFH,
   VREFL
);

   output      OUT;
   input [9:0] D;
   input       VREFH;
   input       VREFL;
   

   reg  real OUT;
   wire real VREFL;
   wire real VREFH;

   real NaN;
   wire EN;

   wire [10:0] Dext;	// unsigned extended

   assign Dext = {1'b0, D};
   assign EN = 1;

   initial begin
      NaN = 0.0 / 0.0;
      if (EN == 1'b0) begin
         OUT <= 0.0;
      end
      else if (VREFH == NaN) begin
         OUT <= NaN;
      end
      else if (VREFL == NaN) begin
         OUT <= NaN;
      end
      else if (EN == 1'b1) begin
         OUT <= VREFL + ($itor(Dext) / 1023.0) * (VREFH - VREFL);
      end
      else begin
         OUT <= NaN;
      end
   end

   always @(D or EN or VREFH or VREFL) begin
      if (EN == 1'b0) begin
         OUT <= 0.0;
      end
      else if (VREFH == NaN) begin
         OUT <= NaN;
      end
      else if (VREFL == NaN) begin
         OUT <= NaN;
      end
      else if (EN == 1'b1) begin
         OUT <= VREFL + ($itor(Dext) / 1023.0) * (VREFH - VREFL);
      end
      else begin
         OUT <= NaN;
      end
   end
endmodule
```
- Inputs:
        - D: A 10-bit digital input from the processor.
        - VREFH: Reference voltage for the DAC.
- Output:
        - OUT: Analog output signal.

## RTL Simulation of modules:

`avsddac.v`
```bash
gedit avsddac.v
gedit tb_avsddac.v
```

> Testbench ported from [rvmyth_avsddac_interface](https://github.com/vsdip/rvmyth_avsddac_interface/blob/main/iverilog/Pre-synthesis/avsddac_tb_test.v) repository

```bash
iverilog -o ~/vcd/photos/VSDBabySoC/src/module/avsddac.vvp ~/vcd/photos/VSDBabySoC/src/module/avsddac.v ~/vcd/photos/VSDBabySoC/src/module/tb_avsddac.v
vvp avsddac.vvp
gtkwave avsddac_tb_test.vcd
```
<img width="1238" height="790" alt="image" src="https://github.com/user-attachments/assets/659c212f-0b44-49a2-a612-02aa4deb35d7" />

`avsdpll.v`

View the design file
```bash
module avsdpll (
   output reg  CLK,
   input  wire VCO_IN,
   input  wire ENb_CP,
   input  wire ENb_VCO,
   input  wire REF
);
   real period, lastedge, refpd;

   initial begin
      lastedge = 0.0;
      period = 25.0; // 25ns period = 40MHz
      CLK <= 0;
   end

  // Toggle clock at rate determined by period
   always @(CLK or ENb_VCO) begin
      if (ENb_VCO == 1'b1) begin
         #(period / 2.0);
         CLK <= (CLK === 1'b0);
      end
      else if (ENb_VCO == 1'b0) begin
         CLK <= 1'b0;
      end 
      else begin
         CLK <= 1'bx;
      end
   end
   
   // Update period on every reference rising edge
   always @(posedge REF) begin
      if (lastedge > 0.0) begin
         refpd = $realtime - lastedge;
         // Adjust period towards 1/8 the reference period
         //period = (0.99 * period) + (0.01 * (refpd / 8.0));
         period =  (refpd / 8.0) ;
      end
      lastedge = $realtime;
   end
endmodule
```
> Testbench ported from [rvmyth_avsdpll_interface](https://github.com/vsdip/rvmyth_avsdpll_interface/blob/main/verilog/pll_tb.v) repository

```bash
iverilog -o ~/Documents/Verilog/Labs/avsdpll.vvp ~/Documents/Verilog/Labs/VSDBabySoC/src/module/avsdpll.v ~/Documents/Verilog/Labs/tb_avsdpll.v
vvp avsddac.vvp
gtkwave avsddac_tb_test.vcd
```
<img width="993" height="628" alt="image" src="https://github.com/user-attachments/assets/f574df61-0762-404c-9919-20b4d6abf754" />

---

## Post-synthesis Simulation of VSDBabySoc

### Synthesis :

Synthesis requires the header files essential for the rvmyth module,
these are
- sp_verilog.vh – includes core Verilog macros and parameter definitions
- sandpiper.vh – defines integration-specific settings used by SandPiper
- sandpiper_gen.vh – contains tool-generated parameters and configuration values

These files need to be present in the working directory of yosys in order to ensure error free synthesis. This is done using the following commands,

cd ~/Documents/Verilog/Labs/VSDBabySoC
cp -r src/include/sp_verilog.vh .
cp -r src/include/sandpiper.vh .
cp -r src/include/sandpiper_gen.vh .

Now inside the `../VSDBabySoC` folder, run yosys,
```bash
yosys
```
In yosys, run 
```bash
read_verilog src/module/vsdbabysoc.v 
read_verilog -I ~/vcd/photos/VSDBabySoC/src/include/ ~/vcd/photos/VSDBabySoC/src/module/rvmyth.v
read_verilog -I ~/vcd/photos/VSDBabySoC/src/include/ ~/vcd/photos/VSDBabySoC/src/module/clk_gate.v
```
This is performed to read the verilog files.
then, the library files
```bash
read_liberty -lib ~/vcd/photos/VSDBabySoC/src/lib/avsdpll.lib 
read_liberty -lib ~/vcd/photos/VSDBabySoC/src/lib/avsddac.lib 
read_liberty -lib ~/vcd/photos/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

Synthesize `vsdbabysoc`, specifying it as the top module,
```bash
synth -top vsdbabysoc
```

Convert D Flip-Flops into equivalent Standard Cell instances by,
```bash
dfflibmap -liberty ~/vcd/photos/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
Perform Optimization and Technology mapping using the following commands,
```bash
opt
abc -liberty ~/vcd/photos/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib -script +strash;scorr;ifraig;retime;{D};strash;dch,-f;map,-M,1,{D}
```
Then, conduct final optimisations and clean-up through,
```bash
flatten
setundef -zero
clean -purge
rename -enumerate
```

- `flatten`: Remove hierarchy, make a flat netlist
- `setundef -zero`: Replace undefined signals with 0
- `clean -purge`: Delete unused/duplicate logic
- `rename -enumerate`: Systematically rename nets and cells

To check the statistics of the synthesised design run,
```bash
stat
```
Statistics:

<pre>
   === vsdbabysoc ===

   Number of wires:               3881
   Number of wire bits:           6279
   Number of public wires:        3881
   Number of public wire bits:    6279
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:               5933
     avsddac                         1
     avsdpll                         1
     sky130_fd_sc_hd__a2111oi_0      9
     sky130_fd_sc_hd__a211oi_1       9
     sky130_fd_sc_hd__a21boi_0       6
     sky130_fd_sc_hd__a21o_2         4
     sky130_fd_sc_hd__a21oi_1      563
     sky130_fd_sc_hd__a221oi_1     149
     sky130_fd_sc_hd__a22o_2         3
     sky130_fd_sc_hd__a22oi_1      162
     sky130_fd_sc_hd__a2bb2oi_1      3
     sky130_fd_sc_hd__a311oi_1       3
     sky130_fd_sc_hd__a31o_2         4
     sky130_fd_sc_hd__a31oi_1      265
     sky130_fd_sc_hd__a32o_1        20
     sky130_fd_sc_hd__a41oi_1       10
     sky130_fd_sc_hd__and2_2        14
     sky130_fd_sc_hd__and3_2         1
     sky130_fd_sc_hd__clkinv_1     464
     sky130_fd_sc_hd__dfxtp_1     1144
     sky130_fd_sc_hd__mux2i_1       13
     sky130_fd_sc_hd__nand2_1      925
     sky130_fd_sc_hd__nand3_1      121
     sky130_fd_sc_hd__nand3b_1       1
     sky130_fd_sc_hd__nand4_1       51
     sky130_fd_sc_hd__nor2_1       699
     sky130_fd_sc_hd__nor3_1        33
     sky130_fd_sc_hd__nor4_1         4
     sky130_fd_sc_hd__o2111ai_1     25
     sky130_fd_sc_hd__o211a_1        2
     sky130_fd_sc_hd__o211ai_1      20
     sky130_fd_sc_hd__o21a_1        13
     sky130_fd_sc_hd__o21ai_0      985
     sky130_fd_sc_hd__o21bai_1      13
     sky130_fd_sc_hd__o221ai_1       8
     sky130_fd_sc_hd__o22ai_1      103
     sky130_fd_sc_hd__o2bb2ai_1      2
     sky130_fd_sc_hd__o311ai_0       4
     sky130_fd_sc_hd__o31ai_1        1
     sky130_fd_sc_hd__o41ai_1        3
     sky130_fd_sc_hd__or2_2         17
     sky130_fd_sc_hd__or3_2          1
     sky130_fd_sc_hd__xnor2_1       13
     sky130_fd_sc_hd__xor2_1        41
</pre>

Then finally write the netlist using the command, 
```bash
write_verilog -noattr ~/vcd/photos/vsdbabysoc_synth.v
```

<img width="1394" height="244" alt="image" src="https://github.com/user-attachments/assets/33204399-e8be-409f-bd2a-a63cbef52b0a" />

Simulation :

Ensure the following files are in the working directory (`photos` in my case) before compilation by running,
```bash
cp -r ~/vcd/photos/VSDBabySoC/src/module/avsddac.v .
cp -r ~/vcd/photos/VSDBabySoC/src/module/avsdpll.v .
cp -r ~/vcd/photos/VSDBabySoC/src/gls_model/sky130_fd_sc_hd.v .
cp -r ~/vcd/photos/VSDBabySoC/src/gls_model/primitives.v .
```
to check the below given files
```
vsdbabysoc_synth.v
avsddac.v
avsdpll.v
primitives.v
sky130_fd_sc_hd.v
```

Compilation of the netlist with the testbench must be done, of course through `iverilog` using the following command,
```bash
iverilog -o ~/vcd/photos/vsdbabysoc_synth.vvp -DPOST_SYNTH_SIM -DFUNCTIONAL -DUNIT_DELAY=#1 -I ~/vcd/photos/VSDBabySoC/src/include -I ~/vcd/photos/VSDBabySoC/src/module -I  ~/vcd/photos/VSDBabySoC/src/gls_model ~/vcd/photos/VSDBabySoC/src/module/testbench.v
```

- `-DPOST_SYNTH_SIM`: Defines the macro POST_SYNTH_SIM to enable post-synthesis simulation mode.
- `-DFUNCTIONAL`: Defines the macro FUNCTIONAL to select functional simulation mode. \
- `-DUNIT_DELAY=#1`: Defines the macro UNIT_DELAY with value #1 for unit delay parameterization in simulation.

Then, to view the waveform,
```bash
vvp vsdbabysoc_synth.vvp
gtkwave post_synth_sim.vcd
```
_Waveform:_

<img width="1593" height="794" alt="image" src="https://github.com/user-attachments/assets/d55cd9ce-8487-4a33-9009-497b8f995801" />

---
## Pre-Synthesis vs Post-Synthesis Simulation

<img width="1912" height="886" alt="image" src="https://github.com/user-attachments/assets/abc54ff4-a328-4620-b6af-7aba189d4c61" />

---

## Summary

A little system-on-a-chip called VSDBabySoC showcases essential SoC principles and functional modeling.  It demonstrates full data flow from compute to output by integrating a CPU, clock control with a PLL, and a DAC.  The design has a strong emphasis on timing, signal behavior, and module integration, allowing for waveform analysis using GTKWave and system functionality simulation and verification with Icarus Verilog.  Simulations conducted before and after synthesis verify the design's functionality, demonstrating how functional modeling connects abstract ideas with real-world SoC implementation.
