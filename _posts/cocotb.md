---
title: "Cocotb : Project steup & AXI verification"
date: 2024-11-21 15:00:00 +0800
categories: [Tutorials, Testbench]
tags: [fpga, testbench, design, cocotb, verilator]
pin: false
math: false
mermaid: false
image:
  path: https://opengraph.githubassets.com/4ce9506b9a67956f2c9c95991677ef577cc341615adc2f4eb2847bb66a2040da/cocotb/cocotb
  alt: Cocotb github page
---
<!-- FINAL NAME : 2024-11-21-TIPS_FOR_COCOTB -->

## Using cocotb to test basic deisgn and AXI interface

Cocotb is a testbench framework blablabla.. If you are here you already know that and don't want a generic intro that looks like it was "written" using a LLM.

But here is something very generic you need to understand before starting :

- **Cocotb** is used to **write** tesbenches
- **Cocotb** is **not** a simulator, but rather uses a already established simulator in the backend

This is great because you just just write the tesbench in python and then use whatever simulator you want, cocotb will handle the simulating part for you.

In this post's examples, I'll use Verilator and I will show you how to

- Setup a nice project (*HDL & tesbenches*).
- Write basic testbenches
- Use [cocotbext-axi](https://github.com/alexforencich/cocotbext-axi) to verify more complex interfaces without too much of a struggle.

## Pre-requesites

- Use linux
- Knowledge of
  - HDL (SystemVerilog)
  - Python
  - Basic Makefile
- [Install cocotb](https://docs.cocotb.org/en/stable/install.html)
- [Install a compatible verilator verision](https://verilator.org/guide/latest/install.html)
- For AXI interface : Install the [corresponding cocotb extension](https://github.com/alexforencich/cocotbext-axi)
- Have a wave visualizer to open the dumped waveforms (in ```.vcd``` format), I use **GTKWave**

## Basic project setup

> Note that the following setup is something I came up with myself and is **NOT** standard by any mean. But it works and I find it quite nice.
{: .prompt-warning }

When starting an HDL project, you first want to create some HDL files, let's do exactly that  in a ```src``` folder :

```txt
.
├── src              
│   └── logic1.sv
```

We'll write very basic logic here :

```sv
module logic1 (
    input logic clk,
    input logic reset_n,
    input logic data_in,
    output logic data_out
);

always @(posedge clk) begin
    if(~reset_n) begin
        data_out <= 1'b0;
    end else begin
        data_out <= data_in;
    end
end
    
endmodule
```

And now coes the time to **test it**. To do so we add a tb folder :

```txt
.
├── src              
│   └── logic1.sv
├── tb              
│   └── logic1
│       ├── Makefile
|       └── test_logic1.py
```

Cocotb works by :

- Writing the tb in python (we'll se that in a minute)
- Writing a Makefile to tell cocotb how to operate these tests by giving it
  - some source files
  - a similator (we'll use **Verilator**)
  - and other directives ...

You can find guidilines for the **Makefile** [here](https://github.com/alexforencich/cocotbext-axi). In our case, here is the one I wrote :

```makefile
# Makefile

# defaults
SIM ?= verilator
TOPLEVEL_LANG ?= verilog
EXTRA_ARGS += --trace --trace-structs
WAVES = 1

VERILOG_SOURCES += $(PWD)/../../src/logic1.sv
# use VHDL_SOURCES for VHDL files

# TOPLEVEL is the name of the toplevel module in your Verilog or VHDL file
TOPLEVEL = logic1

# MODULE is the basename of the Python test file
MODULE = test_logic1

# include cocotb's make rules to take care of the simulator setup
include $(shell cocotb-config --makefiles)/Makefile.sim
```

Most of these are pretty self-explainatory. the ```EXTRA_ARGS += --trace --trace-structs``` line allows us to get waveforms for Verilator. And ```WAVE = 1``` is for icarus verilog (I also used it so I keep that line here for good measure, you can delete it if you want).

Once this is done, all that remains to do is to run ```make``` in the ```tb/logic1``` folder to test this module's logic ! **But** for that we have to write a testbench of course !

You can also add as many HDL files for modules as you want and as many sub-folders in ```tb/``` as you want to test them out !

## Writing a basic testbench

When it come to the tesbench, the [cocotb docs]() gives us many clues on how to do it and you don't need that much feature after all do write some nice and robust testbenches.

Let's review what makes a good testbench :

- Assertion
- Possibility of using a clock
- Maybe some randomness ?

So yeah you don't need that much ! If you are here, there is a good chance you know how to make a testbench and maybe you know your way around cocotb.

Nevertheless, here is an exmaple of how to make a basic testbench, with a clock signl, some randomness and a reset co-routine you can call whenever you need to reset the design.

> Note that "```dut```" stands for "_**D**esign **U**nder **T**est"_ and is an handle you can use to access... well... all the dut's signals to do whatever you want with it ! 

```python
# test_logic1.py

import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, Timer
import random

@cocotb.coroutine
async def reset(dut):
    await RisingEdge(dut.clk)
    dut.reset_n.value = 0
    await RisingEdge(dut.clk)
    dut.rst_n.value = 1
    await Timer(1, units="ns")

    print("reset done !")

    assert dut.data_out.value == 0b0

@cocotb.test()
async def initial_read_test(dut):
    cocotb.start_soon(Clock(dut.clk, 1, units="ns").start())
    await RisingEdge(dut.clk)
    
    # call the reset co-routine
    await reset(dut)

    # do some random tests
    N_TESTS = 1000
    for _ in range (N_TESTS):
      test_value = random.randint(0, 1)
      dut.data_in.value = test_value
      await RisingEdge(dut.clk)
      assert dut.data_out == test_value
```

And now, with our test set up, all that remains to do is to run the ```make``` command in our tb sub-dir and we're all set !

In the backend, Verilator will run and create some build_dir with all of the compiled files that we don't care about because cocotb handles this for us.

Given that we wrote our makefile the right way, we now have a ```dump.vcd``` file that contains all the waveforms. You can open it using ```gtkwave dump.vcd``` or any other wave viewer you want :

[Waveform example](https://image.noelshack.com/fichiers/2024/47/4/1732198043-capture.jpg)

This waveform is totally unrelated to the design we just made, It's just an illustration.

### A tip for have clean projects

> Running the test can create a lot of build folders and files. I suggeest you use a makefile at the root of your project whith the only purpose of clening the project.
{: .prompt-tip }

Here is an exmaple of such a Makefile form the HOLY CORE course I'm developing right now :

```makefile
.PHONY: clean

clean:
	@find ./tb -type d -name "__pycache__" -exec rm -rf {} +
	@find ./tb -type d -name "sim_build" -exec rm -rf {} +
	@find ./tb -type f -name "results.xml" -exec rm -f {} +
	@find ./tb -type f -name "*.None" -exec rm -f {} +
	@find ./tb -type d -name ".pytest_cache" -exec rm -rf {} +
	@find ./tb -type f -name "dump.vcd" -exec rm -f {} +
```

## Another tip for when the project gets bigger

Imagine the following project :

```txt
.
├── src              
│   └── logic1.sv
├── tb              
│   └── logic1
│       ├── Makefile
|       └── test_logic1.py
```