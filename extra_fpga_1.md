# FPGA / RTL / Verification Tooling — Interview-Ready Notes
## PART 1: Tcl → ModelSim → Quartus

> **Goal of these notes:** Not deep mastery — enough solid, correct understanding to speak confidently in interviews and technical discussions.
>
> **Part 1 covers:** Tcl, ModelSim
> **Part 2 (next) covers:** Quartus, UVM (intro), CDC, Reset Synchronizers, plus a final consolidated interview cheat-sheet for all six topics.

---

# 1. Tcl (Tool Command Language)

## 1.1 What It Is

**Tcl** is a simple, interpreted **scripting language**. It is not a hardware description language (not RTL) — it's a general-purpose scripting language, similar in spirit to Python or shell scripting, but much older and simpler in syntax.

## 1.2 Why It's Used in FPGA / EDA Flows

Almost every major EDA tool (Quartus, Vivado, ModelSim, Synopsys tools like PrimeTime/DC, Cadence tools) uses **Tcl as its command/scripting interface**. Instead of clicking through a GUI every time, engineers write Tcl scripts to:
- Automate repetitive tasks (compile, run, generate reports)
- Set design constraints (SDC files are literally Tcl syntax!)
- Batch-run regressions (run 100s of test cases overnight, unattended)
- Control simulation (ModelSim's `do` files are Tcl scripts)
- Query/modify tool databases (get list of nets, pins, timing paths, etc.)

> **Key real-world fact:** SDC (Synopsys Design Constraints) files — used everywhere in STA/synthesis — are literally written in Tcl syntax (`create_clock`, `set_input_delay`, etc. are all Tcl commands).

## 1.3 Basic Syntax

Tcl is **word-based**: everything is a command followed by arguments, space-separated, one command per line (or separated by `;`).

```tcl
# This is a comment
puts "Hello World"          ;# print to console
```

### Variables

```tcl
set x 10                 ;# assign
set name "FPGA_Design"
puts $x                   ;# use $ to READ a variable's value
puts "Value is $x"        ;# variable substitution inside a string
```

### Basic Arithmetic (needs `expr`)

```tcl
set a 5
set b 3
set c [expr {$a + $b}]    ;# square brackets = command substitution
puts $c                    ;# prints 8
```

### Lists

```tcl
set mylist {10 20 30 40}
puts [lindex $mylist 2]    ;# prints 30 (0-indexed)
lappend mylist 50          ;# append to list
```

### Conditionals

```tcl
set freq 100
if {$freq > 50} {
    puts "High frequency"
} elseif {$freq == 50} {
    puts "Medium frequency"
} else {
    puts "Low frequency"
}
```

### Loops

```tcl
# for loop
for {set i 0} {$i < 5} {incr i} {
    puts "i = $i"
}

# while loop
set i 0
while {$i < 3} {
    puts "count $i"
    incr i
}

# foreach loop (very common in EDA scripts, e.g. looping over ports/files)
foreach file {top.v alu.v reg_file.v} {
    puts "Compiling $file"
}
```

### Procedures (functions)

```tcl
proc add {a b} {
    return [expr {$a + $b}]
}
puts [add 4 7]     ;# prints 11
```

## 1.4 Practical Example (Realistic EDA-style script)

```tcl
# A simple script that could appear in a ModelSim "do" file or Quartus Tcl script
set filelist {top.v alu.v reg_file.v control_unit.v}

foreach f $filelist {
    puts "Compiling: $f"
    vlog $f            ;# vlog = ModelSim's Verilog compile command
}

vsim work.tb_top       ;# start simulation
run 1000ns             ;# run simulation for 1000 ns
```

## 1.5 Minimum Core Concepts You Must Know

- Tcl is the **automation/scripting glue** behind almost every EDA tool.
- `set` assigns a variable; `$var` reads it; `[cmd]` runs a command and substitutes its result.
- Everything in Tcl is technically a **string** — even numbers — parsed as needed.
- **SDC constraint files = Tcl syntax.** Commands like `create_clock`, `set_false_path`, `set_input_delay` are Tcl commands provided by the STA/synthesis tool.
- Tcl scripts are used for: batch automation, constraint files, regression running, tool queries/reports.

## 1.6 Connection to RTL / Simulation / Synthesis / Verification

| Stage | How Tcl Is Used |
|---|---|
| RTL / Synthesis | Constraint files (SDC) are Tcl; synthesis scripts (`compile_ultra`, `read_verilog`) are Tcl commands |
| Simulation (ModelSim) | `.do` files that compile, elaborate, and run testbenches are Tcl scripts |
| Physical Design / STA | PrimeTime/Tempus commands (`report_timing`, `set_clock_uncertainty`) are Tcl |
| Verification (UVM) | Regression scripts that launch hundreds of simulations, collect logs, and generate coverage reports are usually Tcl (or Python) automation around the simulator |

**Analogy:** If the EDA tool (Quartus, ModelSim, PrimeTime) is the "engine," Tcl is the **steering wheel and dashboard controls** — a common, tool-agnostic way to drive many different engines.

---

# 2. ModelSim

## 2.1 What It Is

**ModelSim** (by Mentor/Siemens) is an **HDL simulator** — a tool that simulates the functional behavior of your Verilog/VHDL/SystemVerilog design **without needing real hardware**. It lets you verify that your RTL behaves correctly before ever touching an FPGA or silicon.

## 2.2 Why It's Used in FPGA/RTL Flows

- RTL design is worthless if it doesn't do what you intend — simulation is the **primary way to verify functional correctness** before synthesis.
- Debugging in simulation (with waveforms, breakpoints, print statements) is vastly easier and faster than debugging on real hardware.
- Almost every RTL bug is caught in simulation, **long before** synthesis, place & route, or programming a real FPGA.

## 2.3 How Simulation Works — The 3-Step Flow

```
1. COMPILE  →  2. ELABORATE / OPTIMIZE  →  3. SIMULATE (RUN)
```

1. **Compile:** Parses your HDL source files (design + testbench) into an internal database, checking syntax.
2. **Elaborate (optimize):** Builds the actual simulation model by connecting all modules/instances together (resolving hierarchy, parameters, generate blocks). ModelSim calls this step creating an **optimized design unit (`vopt`)** in newer flows, or directly loading via `vsim`.
3. **Simulate (Run):** Actually executes/steps through time, applying stimulus from the testbench and evaluating the design's response.

## 2.4 Common ModelSim Commands (GUI or `.do` script / command-line)

| Command | Purpose |
|---|---|
| `vlib work` | Create a working library (where compiled design is stored) |
| `vlog file.v` | Compile a Verilog/SystemVerilog file |
| `vcom file.vhd` | Compile a VHDL file |
| `vsim work.tb_top` | Load/start simulation of top-level testbench module `tb_top` |
| `run 100ns` | Run simulation for 100 ns |
| `run -all` | Run until no more events / `$finish` is called |
| `add wave *` | Add all signals in current scope to the waveform viewer |
| `add wave -r /*` | Add all signals recursively (full hierarchy) |
| `restart -f` | Restart simulation from time 0 (force restart) |
| `quit -sim` | Exit simulation |

## 2.5 Practical Example — A Minimal Simulation Flow

```tcl
vlib work                      ;# 1. create library
vlog dut.v tb_dut.v            ;# 2. compile DUT and testbench
vsim work.tb_dut                ;# 3. load simulation
add wave -r /*                  ;# 4. add all signals to waveform
run 500ns                       ;# 5. run simulation
```

You'd typically put all these lines into a single `run.do` file and execute it with:
```
vsim -do run.do
```

## 2.6 Testbench Basics (What ModelSim Is Actually Running)

A simple testbench structure (conceptual, Verilog-style):

```verilog
module tb_dut;
  reg clk = 0;
  reg rst_n;
  reg [7:0] data_in;
  wire [7:0] data_out;

  // instantiate Design Under Test (DUT)
  dut u_dut (.clk(clk), .rst_n(rst_n), .data_in(data_in), .data_out(data_out));

  always #5 clk = ~clk;   // generate a clock: 10 ns period

  initial begin
    rst_n = 0; data_in = 0;
    #20 rst_n = 1;          // release reset after 20 ns
    #10 data_in = 8'hAA;    // apply stimulus
    #100 $finish;           // end simulation
  end
endmodule
```

ModelSim compiles this, elaborates the DUT+testbench hierarchy, then **runs** it — advancing simulated time and evaluating signal changes at each event.

## 2.7 Waveform Viewing

After running, engineers open the **Wave window** to visually inspect signal transitions over time — checking clock edges, reset behavior, data values, handshake signals, etc. This is the single most-used **debug tool** in RTL verification — almost every bug hunt starts with "let's look at the waveform."

**Analogy:** If simulation is like running an experiment, the waveform viewer is your **oscilloscope** — letting you see exactly what every signal did, at every point in time.

## 2.8 Minimum Core Concepts You Must Know

- ModelSim simulates HDL code — it does **not** synthesize or program real hardware.
- 3-step flow: **Compile → Elaborate → Simulate**.
- `vlib`, `vlog`/`vcom`, `vsim`, `run`, `add wave` are the essential commands.
- A **testbench** is itself HDL code that generates stimulus (clock, reset, inputs) and (in simple cases) just displays outputs — in advanced verification (UVM), the testbench becomes a full structured environment (Part 2).
- **Waveform viewer** is the primary debug tool.
- `.do` files are **Tcl scripts** that automate the compile-run-view sequence (this is exactly where Tcl and ModelSim connect).

## 2.9 Connection to RTL / Synthesis / Verification

| Stage | Relationship |
|---|---|
| RTL | ModelSim's entire job is to verify RTL behaves as intended, before synthesis |
| Synthesis | Simulation should match the eventual synthesized gate-level netlist's behavior (functional equivalence) — a mismatch (simulation vs synthesis mismatch) is a serious RTL coding bug (e.g., incomplete sensitivity lists, latch inference) |
| Verification | ModelSim is the **execution engine** underneath structured verification methodologies like UVM (Part 2) — UVM testbenches are still just HDL/SystemVerilog code that ModelSim compiles and runs |
| FPGA flow | Simulation happens **before** Quartus synthesis/place-and-route — catching functional bugs early is far cheaper than debugging on real hardware |

---

**End of Part 1.** Say **"next"** when ready for **Part 2**: Quartus, UVM (intro), CDC, Reset Synchronizers — plus the final consolidated interview talking-points list and must-remember commands/concepts for all six topics.
