### ZynqParrot24 Final Report

**ZynqParrot Intro**

ZynqParrot is a version of [BlackParrot](https://github.com/black-parrot/black-parrot) (an open-source, Linux-capable, cache-coherent, RV64GC multicore) that’s designed to be able to fit on the [PYNQ Z2](http://www.pynq.io/) educational FPGA development board released by Xilinx. One of the challenges with ZynqParrot is dealing with the constrained hardware resources of the XCZ-7020 FPGA located on the PYNQ Z2. 

The goal for ZynqParrot this summer was to fit two BlackParrot cores onto the PYNQ Z2 educational FPGA development board. This is constrained by resources; the programmable logic (PL) of the ZYNQ-7000 SoC in the PYNQ Z2 only contains a finite number of hardware primitives such as LUTs, flip-flops, BRAMs, and DSP slices. Fitting two BlackParrot cores requires carefully re-mapping hardware primitives such that the design can fit onto the board without going over the maximum number of primitives. ZynqParrot is constrained by LUTs in this context. At the beginning of this summer, we set out to divert the usage of LUTs to another under-utilized resource on the Z2: DSP48E1 slices. Using Lakeroad, we can apply program synthesis to more efficiently map the DSP48E1 slices being used on the FPGA.

---

**What are DSP48E1 slices?**

DSP48E1 slices are hardened programmable logic elements that are contained with certain Xilinx FPGAs. [You can see Xilinx’s Unisim library component for the DSP48E1 here.](https://github.com/Xilinx/XilinxUnisimLibrary/blob/master/verilog/src/unisims/DSP48E1.v) It’s about 2000 lines of Verilog, but provides a little more context of what’s going on under the hood. Here is a quick diagram of what kinds of inputs and outputs the DSP slice provides.

![DSP48E1 Slice Functionality](https://github.com/cknizek/ZynqParrot24/blob/main/assets/basic-dsp48-slice-functionality.png?raw=true)

Configuring the DSP48E1 slices manually is a bit complex. [The documentation from Xilinx itself is 58 pages.](https://docs.amd.com/v/u/en-US/ug479_7Series_DSP48E1) There are a wide array of inputs that can be configured, and they can be used to efficiently perform multiplication and multiply-accumulate operations common to many DSP applications. We’re interested in the DSP48E1 slices because it provides a way to off-load some of the hardware resources of ZynqParrot that are used by the FMA pipeline. In particular, we want to replace specific multiply-add segments of the design such that they use DSP slices, and not LUTs.

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
|5     |DSP48E1      |    11|
|7     |LUT1         |   533|
|8     |LUT2         |  2815|
|9     |LUT3         |  6618|
|10    |LUT4         |  5627|
|11    |LUT5         |  8237|
|12    |LUT6         | 20330|
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

![FPGA Primitive Visualization](https://github.com/cknizek/ZynqParrot24/blob/main/assets/fpga_primitive_visualization.png?raw=true)

We see that only 11 `DSP48E1` slices are being used when synthesizing ZynqParrot. We can see in the table below that there are 220 `DSP48E1` slices on the PYNQ Z2’s XCZ7020.

![XCZ-7020 Datasheet](https://github.com/cknizek/ZynqParrot24/blob/main/assets/xcz-7020-datasheet.png?raw=true)

This implies that there is significant opportunity in terms of utilization. And using Lakeroad, we can exploit this opportunity.

Lakeroad takes a design, an architecture description, and a model of an FPGA primitive (such as a DSP48E1). Then, Lakeroad uses program synthesis to compile the design to Verilog after generating a sketch and semantically analyzing the file. This compiled Verilog file can be the optimal mapping for the design. 

![Lakeroad components](https://github.com/cknizek/ZynqParrot24/blob/main/assets/lakeroad_components.png?raw=true)

I’ll briefly note that what’s “optimal” can vary between design contexts. In the case of ZynqParrot, we are explicitly trying to reduce consumption of LUT primitives and increase consumption of DSP48E1 primitives. But this is not always the case. It could be the reverse, or it could even be using Lakeroad to compile a more efficient mapping of BRAM18/36 slices from a design. 

One of the really interesting things about Lakeroad is that it can be extended to contexts beyond DSPs. For example, I spent a fair amount of time this summer exploring the usage of Lakeroad on BRAM slices. Interestingly, it may even be possible to extend Lakeroad beyond the context of FPGAs. My mentor Dan spoke about how we could possibly use Lakeroad on SKY130. This summer we used Lakeroad to more efficiently synthesize unsigned multiplication/addition from the BaseJump STL library onto the PYNQ Z2 FPGA. 

---

***How do we optimize using DSP slice mappings?***

Enter Lakeroad. [Lakeroad](https://github.com/uwsampl/lakeroad) is a Yosys plugin that uses sketch-guided program synthesis to compile Verilog/SystemVerilog designs to FPGA primitives. Using Rosette, Lakeroad is able to transform a digital circuit design into a network of the specific logic blocks available in the target FPGA device. It is particularly adept at performing technology mapping on DSPs.

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

We will begin with our top-level module: `bsg_mul_add_unsigned.sv`. I have striked-through all segments of the code that can be removed, and included nearby the lines that can either be modified or added to the file in order to successfully synthesize with Lakeroad.

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

***The Lakeroad/ZynqParrot Flow***

We have a file, `bsg_mul_add_unsigned.sv` we would like to integrate into ZynqParrot’s FMA pipeline after the file has been optimized with Lakeroad.

One thing you may notice is that the bit-widths for the inputs are fairly small. This is because `bsg_dff.sv` will consolidate the widths of `width_a_p`, `width_b_p`, and `width_c_p` in the `pre_mul_add` section. Remember that for the DSP48E1, the input at port A can be a maximum of 18-bits wide, and the input at port B can be a maximum of 25-bits wide. In the toy example below, we will perform a 4x4 multiplication with 8-bit addition operation. In the BaseJump STL source file, `width_o_p` is calculated from:

```verilog
parameter width_o_p = `BSG_SAFE_CLOG2( ((1 << width_a_p) - 1) * ((1 << width_b_p) - 1) +
((1 << width_c_p)-1) + 1 )
```

… but I’ve hardcoded it as `9` for now.

```verilog
`include "bsg_dff.sv"

(* template = "dsp" *) 
(* architecture = "xilinx-ultrascale-plus" *) 
(* pipeline_depth = 3 *)
module in_module #(
    parameter  width_a_p = 4
    ,parameter width_b_p = 4
    ,parameter width_c_p = width_a_p + width_b_p
    ,parameter width_o_p = 9
    ,parameter pipeline_p = 3
  ) (
    (* data *)
    input [width_a_p-1 : 0] a_i,
    (* data *)
    input [width_b_p-1 : 0] b_i,
    (* data *)
    input [width_c_p-1 : 0] c_i,
    (* clk *)
    input clk_i,
    (* out *)
    output [width_o_p-1 : 0] o);

    localparam pre_pipeline_lp = pipeline_p > 2 ? 1 : 0;
    localparam post_pipeline_lp = pipeline_p > 2 ? pipeline_p -1 : pipeline_p;

    wire [width_a_p-1:0] a_r;
    wire [width_b_p-1:0] b_r;
    wire [width_c_p-1:0] c_r;

    wire [1:0] post_data_delayed [width_o_p-1:0];

    // pre_mul_add
    localparam pre_width_p = width_a_p + width_b_p + width_c_p; 
    wire [pre_width_p-1:0] pre_data_o;
    wire [pre_width_p-1:0] pre_data_i;
    wire [1:0] pre_data_delayed [pre_width_p-1:0];
    assign pre_data_i = {a_i, b_i, c_i};

    if(pre_pipeline_lp == 0) begin:pass_through
            wire unused = clk_i;
            assign pre_data_o   = pre_data_i;
    end:pass_through

    else begin:chained
            assign pre_data_delayed[0]  = pre_data_i                        ;
            assign pre_data_o           = pre_data_delayed[pre_pipeline_lp]    ;
            genvar i;
            for(i=1; i<= 1; i++) begin
                reg [pre_width_p-1:0] data_r_pre;
                assign pre_data_delayed[i] = data_r_pre; 
                always @(posedge clk_i) 
                    data_r_pre <= pre_data_delayed[i-1];
            end        
    end:chained

    wire [width_o_p-1:0] o_r = a_r * b_r + c_r;

    // post_mul_add
    if( post_pipeline_lp == 0) begin:pass_through_post
            wire unused = clk_i;
            assign o   = o_r;
    end:pass_through_post

    else begin:chained_post
            assign post_data_delayed[0]  = o_r                        ;
            assign o           = post_data_delayed[post_pipeline_lp]    ;

            genvar i;
            for(i=1; i<= 1; i++) begin
                reg [width_o_p-1:0] data_r_post;
                assign post_data_delayed[i] = data_r_post;
                always @(posedge clk_i) 
                    data_r_post <= post_data_delayed[i-1];
            end
    end:chained_post
endmodule
```

We’ll launch the Lakeroad plugin for Yosys by running the command `yosys -m [lakeroad.so](http://lakeroad.so)` from the `lakeroad/yosys-plugin` directory. 

Then, we run the command `read_verilog /path/to/bsg_mul_add_unsigned.sv` once the Yosys plugin has been launched. The top-level module is `in_module` , so we’ll set that in the hierarchy with the command `hierarchy -top in_module` . Next, we’ll run the command `lakeroad in_module` . Then, we’ll run `rename in_module out_module`. Finally, we’ll run `write_verilog /path/to/bsg_mul_add_unsigned_compiled.v` . The last command will write the compiled output of Lakeroad to the filepath of your choice. 

As an example of what the compiled output look like, the file above is synthesized by Lakeroad as the following:

```verilog
/* Generated by Yosys 0.42 (git sha1 9b6afcf3f, clang++ 15.0.0 -fPIC -Os) */

(* cells_not_processed =  1  *)
(* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:3.1-112.10" *)
module out_module(a_i, b_i, c_i, clk_i, o);
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:4.15-4.18" *)
  wire [47:0] P_0;
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:5.15-5.18" *)
  input [3:0] a_i;
  wire [3:0] a_i;
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:7.15-7.18" *)
  input [3:0] b_i;
  wire [3:0] b_i;
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:9.15-9.18" *)
  input [7:0] c_i;
  wire [7:0] c_i;
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:11.9-11.14" *)
  input clk_i;
  wire clk_i;
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:13.16-13.17" *)
  output [8:0] o;
  wire [8:0] o;
  (* module_not_derived = 32'd1 *)
  (* src = "/var/folders/rg/jq5symwd1cnc731n43t_4dd00000gn/T/4ddc-3074-cec0-9675.v:62.5-102.4" *)
  DSP48E2 #(
    .ACASCREG(32'd0),
    .ADREG(32'd0),
    .ALUMODEREG(32'd0),
    .AMULTSEL("AD"),
    .AREG(32'd0),
    .AUTORESET_PATDET("NO_RESET"),
    .AUTORESET_PRIORITY("RESET"),
    .A_INPUT("DIRECT"),
    .BCASCREG(32'd0),
    .BMULTSEL("AD"),
    .BREG(32'd0),
    .B_INPUT("DIRECT"),
    .CARRYINREG(32'd0),
    .CARRYINSELREG(32'd0),
    .CREG(32'd0),
    .DREG(32'd0),
    .INMODEREG(32'd0),
    .IS_ALUMODE_INVERTED(4'h0),
    .IS_CARRYIN_INVERTED(1'h0),
    .IS_CLK_INVERTED(1'h0),
    .IS_INMODE_INVERTED(5'h00),
    .IS_OPMODE_INVERTED(9'h000),
    .IS_RSTALLCARRYIN_INVERTED(1'h0),
    .IS_RSTALUMODE_INVERTED(1'h0),
    .IS_RSTA_INVERTED(1'h0),
    .IS_RSTB_INVERTED(1'h0),
    .IS_RSTCTRL_INVERTED(1'h0),
    .IS_RSTC_INVERTED(1'h0),
    .IS_RSTD_INVERTED(1'h0),
    .IS_RSTINMODE_INVERTED(1'h0),
    .IS_RSTM_INVERTED(1'h0),
    .IS_RSTP_INVERTED(1'h0),
    .MASK(48'h000000000000),
    .MREG(32'd0),
    .OPMODEREG(32'd0),
    .PATTERN(48'h000000000000),
    .PREADDINSEL("A"),
    .PREG(32'd0),
    .RND(48'h000000000000),
    .SEL_MASK("MASK"),
    .SEL_PATTERN("PATTERN"),
    .USE_MULT("MULTIPLY"),
    .USE_PATTERN_DETECT("NO_PATDET"),
    .USE_SIMD("ONE48"),
    .USE_WIDEXOR("FALSE"),
    .XORSIMD("XOR12")
  ) DSP48E2_0 (
    .A({ a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i[3], a_i }),
    .ACIN(30'h00000000),
    .ALUMODE(4'h0),
    .B({ b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i[3], b_i }),
    .BCIN(18'h00000),
    .C({ c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i }),
    .CARRYCASCIN(1'h0),
    .CARRYIN(1'h0),
    .CARRYINSEL(3'h0),
    .CEA1(1'h1),
    .CEA2(1'h1),
    .CEAD(1'h1),
    .CEALUMODE(1'h1),
    .CEB1(1'h1),
    .CEB2(1'h1),
    .CEC(1'h1),
    .CECARRYIN(1'h1),
    .CECTRL(1'h1),
    .CED(1'h1),
    .CEINMODE(1'h1),
    .CEM(1'h1),
    .CEP(1'h1),
    .CLK(clk_i),
    .D({ c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i[7], c_i }),
    .INMODE(5'h00),
    .MULTSIGNIN(1'h0),
    .OPMODE(9'h000),
    .P({ P_0[47:9], o }),
    .PCIN(48'h000000000000),
    .RSTA(1'h0),
    .RSTALLCARRYIN(1'h0),
    .RSTALUMODE(1'h0),
    .RSTB(1'h0),
    .RSTC(1'h0),
    .RSTCTRL(1'h0),
    .RSTD(1'h0),
    .RSTINMODE(1'h0),
    .RSTM(1'h0),
    .RSTP(1'h0)
  );
  assign P_0[8] = o[8];
  assign P_0[7] = o[7];
  assign P_0[6] = o[6];
  assign P_0[5] = o[5];
  assign P_0[4] = o[4];
  assign P_0[3] = o[3];
  assign P_0[2] = o[2];
  assign P_0[1] = o[1];
  assign P_0[0] = o[0];
endmodule

```

Notice that there was a single DSP that was instantiated. There were no LUTs. We can compare this to the standard output from `yosys` 

```verilog
/* Generated by Yosys 0.42 (git sha1 9b6afcf3f, clang++ 15.0.0 -fPIC -Os) */

(* pipeline_depth = 32'd3 *)
(* architecture = "xilinx-ultrascale-plus" *)
(* template = "dsp" *)
(* dynports =  1  *)
(* top =  1  *)
(* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:17.1-92.10" *)
module out_module(a_i, b_i, c_i, clk_i, o);
  reg \$auto$verilog_backend.cc:2352:dump_module$7  = 0;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:67.17-68.57" *)
  reg [15:0] _0_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:88.17-89.59" *)
  reg [8:0] _1_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:0.0-0.0" *)
  reg [1:0] _2_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:0.0-0.0" *)
  reg [1:0] _3_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:0.0-0.0" *)
  reg [1:0] _4_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:0.0-0.0" *)
  reg [1:0] _5_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:72.32-72.47" *)
  wire [8:0] _6_;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:72.32-72.41" *)
  wire [8:0] _7_;
  (* data = 32'd1 *)
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:25.29-25.32" *)
  input [3:0] a_i;
  wire [3:0] a_i;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:42.26-42.29" *)
  wire [3:0] a_r;
  (* data = 32'd1 *)
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:28.29-28.32" *)
  input [3:0] b_i;
  wire [3:0] b_i;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:43.26-43.29" *)
  wire [3:0] b_r;
  (* data = 32'd1 *)
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:31.29-31.32" *)
  input [7:0] c_i;
  wire [7:0] c_i;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:44.26-44.29" *)
  wire [7:0] c_r;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:65.39-65.49" *)
  reg [15:0] \chained.genblk1[1].data_r_pre ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:86.37-86.48" *)
  reg [8:0] \chained_post.genblk1[1].data_r_post ;
  (* clk = 32'd1 *)
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:34.11-34.16" *)
  input clk_i;
  wire clk_i;
  (* out = 32'd1 *)
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:37.30-37.31" *)
  output [8:0] o;
  wire [8:0] o;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:72.26-72.29" *)
  wire [8:0] o_r;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  reg [1:0] \post_data_delayed[0] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  reg [1:0] \post_data_delayed[1] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[2] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[3] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[4] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[5] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[6] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[7] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:46.16-46.33" *)
  wire [1:0] \post_data_delayed[8] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  reg [1:0] \pre_data_delayed[0] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[10] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[11] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[12] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[13] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[14] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[15] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  reg [1:0] \pre_data_delayed[1] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[2] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[3] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[4] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[5] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[6] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[7] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[8] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:52.16-52.32" *)
  wire [1:0] \pre_data_delayed[9] ;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:51.28-51.38" *)
  wire [15:0] pre_data_i;
  (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:50.28-50.38" *)
  wire [15:0] pre_data_o;
  assign _6_ = _7_ + (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:72.32-72.47" *) c_r;
  assign _7_ = a_r * (* src = "/Users/colinknizek/dev/lakeroad_bitbit/lakeroad/integration_tests/lakeroad/bsg_mul_add_unsigned/bsg_mul_add_unsigned_synth.sv:72.32-72.41" *) b_r;
  always @* begin
    if (\$auto$verilog_backend.cc:2352:dump_module$7 ) begin end
    _4_ = pre_data_i[1:0];
    _5_ = \chained.genblk1[1].data_r_pre [1:0];
    _2_ = o_r[1:0];
    _3_ = \chained_post.genblk1[1].data_r_post [1:0];
  end
  always @* begin
      \post_data_delayed[0]  <= _2_;
      \post_data_delayed[1]  <= _3_;
      \pre_data_delayed[0]  <= _4_;
      \pre_data_delayed[1]  <= _5_;
  end
  always @* begin
    if (\$auto$verilog_backend.cc:2352:dump_module$7 ) begin end
    _0_ = { 14'h0000, \pre_data_delayed[0]  };
  end
  always @(posedge clk_i) begin
      \chained.genblk1[1].data_r_pre  <= _0_;
  end
  always @* begin
    if (\$auto$verilog_backend.cc:2352:dump_module$7 ) begin end
    _1_ = { 7'h00, \post_data_delayed[0]  };
  end
  always @(posedge clk_i) begin
      \chained_post.genblk1[1].data_r_post  <= _1_;
  end
  assign pre_data_i = { a_i, b_i, c_i };
  assign o_r = _6_;
  assign pre_data_o = { 14'h0000, \pre_data_delayed[1]  };
  assign o = { 7'h00, \post_data_delayed[2]  };
endmodule

```

This compiled file can replace a file that would otherwise be compiled with Vivado or Yosys. 

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

![Vivado synthesis flags](https://github.com/cknizek/ZynqParrot24/blob/main/assets/vivado_synthesis_flags.png?raw=true)

When using the `synth_design` command, these flags can be included with the command in order to change the behavior during synthesis. In the case of our Churchroad evaluation, we discovered that certain flags (such as setting `cascade_dsp force` ) would drastically reduce the number of LUT and carry primitives being consumed, and instead use DSP slices instead. In some cases, the overall number of DSPs used was unchanged, and all other primitives were eliminated from use.

The ability to encode features is a feature I thought was important to add because it enables for new visualization opportunities when presenting Churchroad benchmarks. 

*A screenshot of a Pandas DataFrame of benchmark output data*

![Pandas DataFrame](https://github.com/cknizek/ZynqParrot24/blob/main/assets/benchmark_pandas_df.png?raw=true)

---

***How does Churchroad relate to ZynqParrot?***

The Xilinx DSP48E1 slices are limited to 18x25-width inputs. This means that, if either input A is larger than 18-bits, or input B is larger than 25-bits, the operation will involve more than 1 DSP. Lakeroad cannot handle more than 1 DSP at a time for operations like multiplication. This is where Churchroad comes in. Churchroad is an extension of Lakeroad that uses equality saturation to break up larger multiplication operations into ones that can fit onto a single DSP, by building up an e-graph and performing rewrites. Churchroad is relevant to ZynqParrot because the FMA pipeline uses multiplication involving higher bit-widths than 18/25-bits. And so, Churchroad is required in order to optimize the pipeline for the DSP48E1 slices. 

---

***What’s next for ZynqParrot?***

The next step for optimizing the DSP slice mappings for ZynqParrot is using Churchroad on `bsg_mul_add_unsigned.sv` and other potentially relevant BaseJump STL files, and placing the compiled output into ZynqParrot. This would require Churchroad because currently the bit-width limitations of the DSP48E1 slices are such that they cannot accept inputs larger than 18x25-bits, which is smaller than the bit-width used by the FMA pipe for ZynqParrot. 

Another task that needs to be done is figuring out how to get Lakeroad to succesfully execute on files that have multiple instantiated modules. If you look at the example above, in order to get `bsg_mul_add_unsigned.sv` to actually synthesize I needed to re-write the file to directly implement the logic that's otherwise done in `bsg_dff.sv` and `bsg_dff_chain.sv`. I honestly am not too sure why Lakeroad fails at the moment on this type of file, but I have some guesses.

---

**Pull Requests**

*Lakeroad*

[bitwuzla_commit_hash_update](https://github.com/uwsampl/lakeroad/pull/461)

[verilator_testbench_specification](https://github.com/uwsampl/lakeroad/pull/459)

[bsg_mul_add_unsigned_dev](https://github.com/uwsampl/lakeroad/pull/454)

[Enable Rosette SMT Output](https://github.com/uwsampl/lakeroad/pull/460)

[Added unique log file for each yosys command](https://github.com/uwsampl/lakeroad/pull/446)

*Churchroad*

[New benchmarks + synthesis flags + encoded features for dataviz](https://github.com/uwsampl/churchroad-evaluation/pull/2)

---

**Issues**

*Bitwuzla* 

[AigCnfEncoder::_encode Timeout](https://github.com/bitwuzla/bitwuzla/issues/122)

[Timing out on SMT2 File](https://github.com/bitwuzla/bitwuzla/issues/121)

*Churchroad*

[2RW RAM Vivado Benchmark](https://github.com/uwsampl/churchroad/issues/97)

[Bitmask RAM Vivado Benchmark](https://github.com/uwsampl/churchroad/issues/96)

[Vivado Benchmarks](https://github.com/uwsampl/churchroad/issues/90)

*Lakeroad*

[yosys-plugin segfaults on multi-module designs](https://github.com/uwsampl/lakeroad/issues/466)

[Including bsg_dff_chain.sv into a Lakeroad integration test causes test failure](https://github.com/uwsampl/lakeroad/issues/458)

---

**Conclusion**

I’m really grateful for the opportunity this summer to have been a part of ZynqParrot. The single most valuable thing I learned was how to contribute to open-source projects. Before this summer, I had no idea how contributing to open-source even worked. Getting started with open-source seemed daunting. The search space of potential open-source projects to contribute is vast. But GSoC allowed me to approach a project with training wheels on. I was fortunate to be able to receive mentorship from Dan Petrisko and Gus Henry Smith; both whom dedicated large amounts of (unpaid) time in helping introduce me to open-source and hardware synthesis. 

I am excited to continue contributing to Lakeroad, Churchroad, and ZynqParrot!

---

**Sources**

[1] [ZYNQ-7000 Datasheet](https://www.mouser.com/datasheet/2/903/ds190_Zynq_7000_Overview-1595492.pdf)

[2] [Lakeroad paper](https://arxiv.org/abs/2401.16526)

[3] [UG901 Vivado Design Suite User Guide: Synthesis](https://docs.amd.com/v/u/2021.1-English/ug901-vivado-synthesis)

[4] [7 Series DSP48E1 Slice User Guide (UG479)](https://docs.amd.com/v/u/en-US/ug479_7Series_DSP48E1)
