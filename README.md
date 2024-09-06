### GSoC Final Report

**Intro**

The goal for ZynqParrot this summer was to fit two BlackParrot cores onto the PYNQ Z2 educational FPGA development board. This is constrained by resources; the programmable logic (PL) of the ZYNQ-7000 SoC in the PYNQ Z2 only contains a finite number of hardware primitives such as LUTs, flip-flops, BRAMs, and DSP slices. Fitting two BlackParrot cores requires carefully re-mapping hardware primitives such that the design can fit onto the board without going over the maximum number of primitives. ZynqParrot is constrained by LUTs in this context. At the beginning of this summer, we set out to divert the usage of LUTs to another under-utilized resource on the Z2: DSP48E1 slices. Using Lakeroad, we can apply program synthesis to more efficiently map the DSP48E1 slices being used on the FPGA.

---

***What is Lakeroad?***

[Lakeroad](https://github.com/uwsampl/lakeroad) is a Yosys plugin that uses sketch-guided program synthesis to compile Verilog/SystemVerilog designs to FPGA primitives. Using Rosette, Lakeroad is able to transform a digital circuit design into a network of the specific logic blocks available in the target FPGA device. It is particularly adept at performing technology mapping on DSPs.

Lakeroad allows FPGA designers to use fewer resources to synthesize a design. On certain representative microbenchmarks, Lakeroad is able to produce 2-3.5x the number of optimal mappings when compared to proprietary SoTA tools such as Xilinx Vivado. Lakeroad is able to produce 6-44x the number of optimal mappings when compared to open-source tools such as Yosys. In addition, Lakeroad is able to provide formal correctness guarantees that tools such as Vivado and Yosys cannot.

---

***Why would we use Lakeroad for ZynqParrot?***

For certain types of designs, such as multipliers, Lakeroad is able to synthesize optimal mappings when compared to both commercial and open-source tools. It does this using sketch templates, FPGA architecture descriptions, and source code semantics extraction. 

Since we know Lakeroad is very efficient at synthesizing multiplier designs, we can apply Lakeroad to segments of the ZynqParrot codebase that perform multiplication. The goal here is to construct an optimized mapping for certain modules, such that fewer primitives are used in the implemented design on an FPGA.

The segment of the ZynqParrot codebase that we are interested in applying Lakeroad to is `bsg_mul_add_unsigned.sv` , which comes from the BaseJump STL library that ZynqParrot uses.

---

***How do we use Lakeroad for ZynqParrot?***

BaseJump STL has two versions of `bsg_mul_add_unsigned.sv` . One is hardened, and one is not. We can select [the hardened version](https://github.com/bespoke-silicon-group/basejump_stl/blob/0b8ca3b19d3ae27c5e6f6d4335ca6d792bf1f45e/hard/ultrascale_plus/bsg_misc/bsg_mul_add_unsigned.sv#L12) because we will be mapping `bsg_mul_add_unsigned.sv` to the Zynq 7020 FPGA. The next step we will take is to specify the bit widths. In our case, let’s do a 13x13 multiplication. We will have to remove all segments of the file that are not synthesizable by Lakeroad, which means the lines beginning with `//` will be removed, along with expressions like the `initial assert...` statement. However, new lines beginning with `//` will be added to the top of the file because this allows us to use the Lit testing framework to run this file as an integration test. 

I will walk through the modifications required because I hope it will be useful in case you decide to use Lakeroad on your own.

We will begin with our top-level module: `bsg_mul_add_unsigned.sv`. I have strike-throughed all segments of the code that should be removed, and included nearby the lines that should either be modified or added to the file in order to successfully synthesize with Lakeroad.

`bsg_mul_add_unsigned.sv` will look like the following:

```verilog
~~// 21 Jul 2021
// A special module for use with HardFloat's mulAddRecFN for
// easy absorption of pipeline stages by the DSPs that get
// synthesised in some FPGAs. This helps in retiming the
// paths in FPGA implementations as only immediate registers
// are absorbed, and global retiming does not seem to do this.
//
// For Zynq 7020, pipeline_p = 3~~
// RUN: outfile=$(mktemp)
// RUN: yosys -m "$LAKEROAD_DIR/yosys-plugin/lakeroad.so" -p " \
// RUN:  read_verilog %s; \
// RUN:  hierarchy -top in_module; \
// RUN:  lakeroad in_module; \
// RUN:  rename in_module out_module; \
// RUN:  write_verilog $outfile"
// RUN: FileCheck %s < $outfile

`include "bsg_defines.sv"

(* template = "dsp" *) 
(* architecture = "xilinx-ultrascale-plus" *) 
(* pipeline_depth = 3 *)
module bsg_mul_add_unsigned #(
		parameter width_a_p = 13
		,parameter width_b_p = 13
		,parameter width_c_p = 26
		,parameter width_o_p = 27
    ~~parameter  `BSG_INV_PARAM(width_a_p)
    ,parameter `BSG_INV_PARAM(width_b_p)~~
    ~~,parameter width_c_p = width_a_p + width_b_p
    ,parameter width_o_p = `BSG_SAFE_CLOG2( ((1 << width_a_p) - 1) * ((1 << width_b_p) - 1) +
                                                    ((1 << width_c_p)-1) + 1 )~~
    ,parameter pipeline_p = 3
  ) (
    input clk_i
    ,input [width_a_p-1 : 0] a_i
    ,input [width_b_p-1 : 0] b_i
    ,input [width_c_p-1 : 0] c_i
    ,output [width_o_p-1 : 0] o
    );

  ~~initial assert (pipeline_p > 2) else $warning ("%m: pipeline_p is set quite low; most likely frequency will be impacted");~~
    localparam pre_pipeline_lp = pipeline_p > 2 ? 1 : 0;
    localparam post_pipeline_lp = pipeline_p > 2 ? pipeline_p -1 : pipeline_p; ~~//for excess~~

    wire [width_a_p-1:0] a_r;
    wire [width_b_p-1:0] b_r;
    wire [width_c_p-1:0] c_r;

    bsg_dff_chain #(width_a_p + width_b_p + width_c_p, pre_pipeline_lp)
        pre_mul_add (
            .clk_i(clk_i)
            ,.data_i({a_i, b_i, c_i})
            ,.data_o({a_r, b_r, c_r})
        );

    wire [width_o_p-1:0] o_r = a_r * b_r + c_r;

    bsg_dff_chain #(width_o_p, post_pipeline_lp)
        post_mul_add (
            .clk_i(clk_i)
            ,.data_i(o_r)
            ,.data_o(o)
        );
endmodule

~~`BSG_ABSTRACT_MODULE(bsg_mul_add_unsigned)~~
// CHECK: module out_module(a_i, b_i, c_i, clk_i, o);
// CHECK:   DSP48E2 #(
// CHECK: endmodule
```

We’ll also need to do something about the different included modules. As it is, this file won’t synthesize. In fact, it will likely either result in a segmentation fault or fail. Upon inspection, we notice that this file has three dependencies:

- `bsg_defines.sv`
- `bsg_dff.sv`
- `bsg_dff_chain.sv`

Each of these three dependencies will need to be modified in order to avoid a segfault or synthesis timeout. 

[`bsg_dff.sv`](https://github.com/bespoke-silicon-group/basejump_stl/blob/master/bsg_misc/bsg_dff.sv)

```verilog
`include "bsg_defines.sv"

(* template = "dsp" *) 
(* architecture = "xilinx-ultrascale-plus" *) 
(* pipeline_depth = 0 *)
module bsg_dff #(~~parameter `BSG_INV_PARAM(width_p)~~
	    parameter width_p = 27
		 ,harden_p=0
		 ,strength_p=1   // set drive strength
		 )
   (input   clk_i
    ,input  [width_p-1:0] data_i
    ,output [width_p-1:0] data_o
    );

   reg [width_p-1:0] data_r;

   assign data_o = data_r;

   always @(posedge clk_i)
     data_r <= data_i;

endmodule

~~`BSG_ABSTRACT_MODULE(bsg_dff)~~
```

[`bsg_dff_chain`](https://github.com/bespoke-silicon-group/basejump_stl/blob/master/bsg_misc/bsg_dff_chain.sv)

```
~~//====================================================================
// bsg_dff_chain.sv
// 04/01/2018, shawnless.xie@gmail.com
//====================================================================
//
// Pass the input singal to a chainded  DFF registers~~

`include "bsg_dff.sv"
`include "bsg_defines.sv"

(* template = "dsp" *) 
(* architecture = "xilinx-ultrascale-plus" *) 
(* pipeline_depth = 1 *)
module bsg_dff_chain #(
                 ~~//the width of the input signal~~
                 ~~parameter `BSG_INV_PARAM(      width_p         )~~
                 parameter width_p = 27;

                 ~~//the stages of the chained DFF register
                 //can be 0~~
                ,parameter       num_stages_p    =       1
        )
        (
                 (* data *)
                 input                           clk_i
                 (* data *)
                ,input [width_p-1:0]             data_i
                 (* out *)
                ,output[width_p-1:0]             data_o
        );

        if( num_stages_p == 0) begin:pass_through
                wire unused = clk_i;
                assign data_o   = data_i;
        end:pass_through

        else begin:chained
                ~~// data_i -- delayed[0]
                //
                // data_o -- delayed[num_stages_p]~~
                logic [num_stages_p:0][width_p-1:0] data_delayed;

                assign data_delayed[0]  = data_i                        ;
                assign data_o           = data_delayed[num_stages_p]    ;

                genvar i;
                for(i=1; i<= num_stages_p; i++) begin
                        bsg_dff #( .width_p ( width_p ) )
                                ch_reg (
                                        .clk_i        ( clk_i                 )
                                       ,.data_i         ( data_delayed[ i-1 ]   )
                                       ,.data_o         ( data_delayed[ i   ]   )
                                );
                end

        end:chained

endmodule

~~`BSG_ABSTRACT_MODULE(bsg_dff_chain)~~
```

I will leave out [`bsg_defines.sv`](https://github.com/bespoke-silicon-group/basejump_stl/blob/master/bsg_misc/bsg_defines.sv) as that particular file does not need to be modified. All three of these files can be then placed into the directory `lakeroad/integration_tests/lakeroad` . Navigate to this directory, and then run the command `lit -a "bsg_mul_add_unsigned.sv` . If the test was successful, you should see something like: `PASS: Lakeroad tests :: bsg_mul_add_unsigned.sv (1 of 1)` in your terminal output. You should also see the compiled version of your file in the `$outfile` location.

Now that we’ve gone over a brief introduction to how HDL code can be modified to support Lakeroad, I’ll discuss what the benefits of Lakeroad look like when applied to designs.

---

**The Constraints of the PYNQ Z2 - ZYNQ 7020 FPGA**

In the excerpt below from `vivado.log` [1] (produced by synthesizing `black-parrot-example` of `zynq-parrot` using Vivado), we can see that the “naive” ZynqParrot utilizes many LUTs and few DSPs.

```
Report Cell Usage: 
+------+-------------+------+
|      |Cell         |Count |
+------+-------------+------+
|1     |BIBUF        |   130|
|2     |BUFG         |     4|
|3     |BUFGMUX_CTRL |     3|
|4     |CARRY4       |  1089|
**|5     |DSP48E1      |    11|**
**|7     |LUT1         |   533|
|8     |LUT2         |  2815|
|9     |LUT3         |  6618|
|10    |LUT4         |  5627|
|11    |LUT5         |  8237|
|12    |LUT6         | 20330|**
|13    |MMCME2_ADV   |     1|
|14    |MUXF7        |   395|
|15    |MUXF8        |     4|
|16    |PS7          |     1|
|17    |RAM16X1D     |     5|
|18    |RAM32M       |   619|
|19    |RAM32X1D     |    11|
|20    |RAM64M       |     5|
|21    |RAM64X1S     |     7|
|22    |RAMB18E1     |     3|
|24    |RAMB36E1     |    48|
|28    |SRL16        |     4|
|29    |SRL16E       |   105|
|30    |SRLC32E      |   136|
|31    |FDCE         |   126|
|32    |FDR          |    16|
|33    |FDRE         | 16056|
|34    |FDSE         |   798|
|35    |LD           |   240|
|36    |IBUF         |     1|
+------+-------------+------+
```

Here is the same data visualized into a bar chart.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/a28cecc8-6cbe-4cb2-bc3b-71829693742d/b804568a-ec4b-4a25-86e5-2b8e7ef1e39d/image.png)

We see that only 11 `DSP48E1` slices are being used when synthesizing ZynqParrot. We can see in the table below that there are 220 `DSP48E1` slices on the PYNQ Z2’s XCZ7020.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/a28cecc8-6cbe-4cb2-bc3b-71829693742d/5108ad25-ace1-4f43-8e6f-5b1dfe683dc5/image.png)

This implies that there is significant opportunity in terms of utilization. And using Lakeroad, we can exploit this opportunity.

Lakeroad takes a design, an architecture description, and a model of an FPGA primitive (such as a DSP48E1). Then, Lakeroad uses program synthesis to compile the design to Verilog after generating a sketch and semantically analyzing the file. This compiled Verilog file can be the optimal mapping for the design. 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/a28cecc8-6cbe-4cb2-bc3b-71829693742d/39d585a6-8374-495f-9714-c1897bb256e3/image.png)

I’ll briefly note that what’s “optimal” can vary between design contexts. In the case of ZynqParrot, we are explicitly trying to reduce consumption of LUT primitives and increase consumption of DSP48E1 primitives. But this is not always the case. It could the reverse, or it could even be using Lakeroad to compile a more efficient mapping of BRAM18/36 slices from a design. 

One of the really interesting things about Lakeroad is that it can be extended to contexts beyond DSPs. For example, I spent a fair amount of time this summer exploring the usage of Lakeroad on BRAM slices. Interestingly, it may even be possible to extend Lakeroad beyond the context of FPGAs. My mentor Dan spoke about how we could possibly use Lakeroad on SKY130. This summer we used Lakeroad to more efficiently synthesize unsigned multiplication/addition from the BaseJump STL library onto the PYNQ Z2 FPGA. 

---

**Lakeroad Pull Requests**

I became involved with Lakeroad as a result of the relationship between ZynqParrot and Lakeroad. The primary focus of my involvement with Lakeroad was to implement `bsg_mul_add_unsigned.sv` into Lakeroad, so that we could use it for ZynqParrot. 

In this section, I’ll go over some of the pull requests I made to Lakeroad. In each description, I’ll detail the change and why it was made.

I’ll start with my most recent PR to Lakeroad.

`bitwuzla_commit_hash_update`: https://github.com/uwsampl/lakeroad/pull/461 

The motivation behind this PR was to update the version of Bitwuzla (an SMT2 solver used for Lakeroad’s counter-example guided synthesis, and kind of our favorite solver) to the most recent version. This is because the version of Bitwuzla previously used was over a year old, and the Bitwuzla team made some changes that fixed some other issues (related to stuff like integrating `bsg_mul_add_unsigned.sv` with Lakeroad segfaulting or hanging indefinitely. This was complicated by the fact that the most recent version of Bitwuzla breaks Lakeroad. That is, using the most recent version will result in Lakeroad failing to synthesize anything at all. It turns out that the Bitwuzla team made a fix, and we’re integrating it into Lakeroad. You can learn more about the issue here: `AigCnfEncoder::_encode Timeout:` https://github.com/bitwuzla/bitwuzla/issues/122

The next PR focused on adding support for Rosette’s SMT output feature. This feature allows for the specification of a directory to output all SMT2 files produced by Lakeroad’s program synthesis. This is especially useful for debugging solvers (checking sat vs. unsat, whether a solver is timing out a certain file, or whether it’s segfaulting). `Enable Rosette SMT Output:` https://github.com/uwsampl/lakeroad/pull/460

Going back in time, I made another PR to Lakeroad was meant to solve a similar issue as `bitwuzla_commit_hash_update`: Lakeroad would time out when trying to synthesize `bsg_mul_add_unsigned.sv` 

`verilator_testbench_specification`: https://github.com/uwsampl/lakeroad/pull/459

In this PR, I added some code to Lakeroad’s integration with Verilator in `bin/simulate_with_verilator.py` such that the `testbench_module_name` and `top_module` for a file can be specified. This is useful because we noticed that certain files would time out less after this fix.

The only pending PR I have made to Lakeroad, as of the time of writing, is `bsg_mul_add_unsigned_dev`. This PR adds `bsg_mul_add_unsigned.sv` and the file’s dependencies to Lakeroad’s integration tests. This PR is still pending and hasn’t been merged yet, due to some remaining issues with synthesizing the designs using Lakeroad’s `yosys-plugin` and with Lakeroad in general.

`bsg_mul_add_unsigned_dev`: https://github.com/uwsampl/lakeroad/pull/454

The first PR I made to Lakeroad was: `Added unique log file for each yosys command:` https://github.com/uwsampl/lakeroad/pull/446. This PR added supported for specifying the log filepath for Yosys, which was useful because it allows for debugging Yosys synthesis errors when synthesizing integration tests and files.

---

**Churchroad**

This summer also included development related to Churchroad - the “next-gen” variant of Lakeroad. [Churchroad](https://github.com/uwsampl/churchroad) uses egglog and equality saturation. It generates an egraph of the design. The egraph is iterated upon with successive rewrites until an optimal mapping is achieved. 

My contributions towards the Churchroad evaluation repo consisted of expanding the number of benchmarks available, adding the capability of custom Vivado synthesis flags, and adding the ability to encode features such as input/output bit-width to individual benchmarks. I added 4 MAC benchmarks, 25 multiply benchmarks, and 6 mul-add benchmarks. The benchmarks were permutated through pipeline stages, input bit-widths, and output bit-widths. This is important for the purposes of Churchroad’s benchmarking because it allows us to measure the primitive utilization (particularly of DSP slices) across various multiply patterns that pop up in different situations. You can view all of the new benchmarks in the pull request here: `New benchmarks + synthesis flags + encoded features for dataviz`: https://github.com/uwsampl/churchroad-evaluation/pull/2

The addition of capability for custom Vivado synthesis flags was because Vivado allows for a significant amount of customization in terms of synthesis, based on specific flags included in the `.tcl` file. For example, some Vivado flags are listed in the screenshot below [2]:

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/a28cecc8-6cbe-4cb2-bc3b-71829693742d/d5a6f3db-2f5d-49a4-a9d5-0909daef435b/image.png)

When using the `synth_design` command, these flags can be included with the command in order to change the behavior during synthesis. In the case of our Churchroad evaluation, we discovered that certain flags (such as setting `cascade_dsp force` ) would drastically reduce the number of LUT and carry primitives being consumed, and instead use DSP slices instead. In some cases, the overall number of DSPs used was unchanged, and all other primitives were eliminated from use.

The ability to encode features is a feature I thought was important to add because it enables for new visualization opportunities when presenting Churchroad benchmarks. 

*A screenshot of a Pandas DataFrame of benchmark output data*

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/a28cecc8-6cbe-4cb2-bc3b-71829693742d/3844a037-05e5-48c8-8140-44c06360a70b/image.png)

---

**Pull Requests**

*Lakeroad*

`bitwuzla_commit_hash_update`: https://github.com/uwsampl/lakeroad/pull/461 

`verilator_testbench_specification`: https://github.com/uwsampl/lakeroad/pull/459

`bsg_mul_add_unsigned_dev`: https://github.com/uwsampl/lakeroad/pull/454

`Enable Rosette SMT Output:` https://github.com/uwsampl/lakeroad/pull/460

`Added unique log file for each yosys command:` https://github.com/uwsampl/lakeroad/pull/446

*Churchroad*

`New benchmarks + synthesis flags + encoded features for dataviz`: https://github.com/uwsampl/churchroad-evaluation/pull/2

---

**Issues**

*Bitwuzla* 

`AigCnfEncoder::_encode Timeout:` https://github.com/bitwuzla/bitwuzla/issues/122

`Timing out on SMT2 File:` https://github.com/bitwuzla/bitwuzla/issues/121

*Churchroad*

`2RW RAM Vivado Benchmark:` https://github.com/uwsampl/churchroad/issues/96

`Bitmask RAM Vivado Benchmark:` https://github.com/uwsampl/churchroad/issues/96

`Vivado Benchmarks:` https://github.com/uwsampl/churchroad/issues/90

*Lakeroad*

`yosys-plugin segfaults on multi-module designs:` https://github.com/uwsampl/lakeroad/issues/466

`Including bsg_dff_chain.sv into a Lakeroad integration test causes test failure:` https://github.com/uwsampl/lakeroad/issues/458

---

**Conclusion**

I’m really grateful for the opportunity this summer to have been a part of ZynqParrot. The single most valuable thing I learned was how to contribute to open-source projects. Before this summer, I had no idea how contributing to open-source even worked. Getting started with open-source seemed daunting. The search space of potential open-source projects to contribute is vast. But GSoC allowed me to approach a project with training wheels on. I was fortunate to be able to receive mentorship from Dan Petrisko and Gus Henry Smith; both whom dedicated large amounts of (unpaid) time in helping introduce me to open-source and hardware synthesis. 

I am excited to continue contributing to Lakeroad, Churchroad, and ZynqParrot!

---

**Sources**

[x] [ZYNQ-7000 Datasheet](https://www.mouser.com/datasheet/2/903/ds190_Zynq_7000_Overview-1595492.pdf)

[x] [Lakeroad paper](https://arxiv.org/abs/2401.16526)

[x] [UG901 Vivado Design Suite User Guide: Synthesis](https://docs.amd.com/v/u/2021.1-English/ug901-vivado-synthesis)
