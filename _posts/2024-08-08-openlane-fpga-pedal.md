---
layout:     post
title:      "Verilog for Guitarists"
subtitle: "Creating FPGA Guitar Effects"
author:     "strojacek"
header-style: text
catalog: true
tags:
  - Verilog
  - Audio Programming
  - FPGA
  - OpenLane
---

> Learning Verilog by making an FPGA Guitar Effects Pedal


Introduction
---

This series of posts is based upon the following [repository](https://github.com/jagumiel/Guitar-Pedal-Effects-on-FPGA), however using Verilog instead of VHDL.
The purpose of this series is to teach hardware design to guitarists, and to help explain the different signal processing changes that occur at the hardware level, to a guitar's signal.


A brief introduction to Verilog
---






Audio In
---
```verilog
module au_in (
    input wire clk,
    input wire reset,
    input wire adclrc,
    input wire bclk,
    input wire adcdat,
    output reg [15:0] sample,
    output reg ready
);

    // State encoding
    typedef enum reg [2:0] {
        e0 = 3'b000,
        e1 = 3'b001,
        e2 = 3'b010,
        e3 = 3'b011,
        e4 = 3'b100,
        e5 = 3'b101
    } estado;

    // State registers
    reg [2:0] ep, es;
    
    // Shift register and control signals
    reg [15:0] srdato;
    reg desplaza;
    reg incbits;
    reg ultimo;

    // Synchronized inputs
    reg adclrcs, bclks, adcdats;

    // ==================================================
    // Control Unit
    // Next state calculation -------------------------
    always @* begin
        case (ep)
            e0: es = (adclrcs == 1'b0) ? e1 : e0;
            e1: es = (adclrcs == 1'b1 && bclks == 1'b0) ? e2 : e1;
            e2: es = (bclks == 1'b1) ? e3 : e2;
            e3: es = (ultimo == 1'b0) ? e4 : e5;
            e4: es = (bclks == 1'b0) ? e2 : e4;
            e5: es = e0;
        endcase
    end

    // State storage -------------------------------
    always @(posedge clk or posedge reset) begin
        if (reset) 
            ep <= e0;
        else
            ep <= es;
    end

    // Control signals -----------------------------
    always @* begin
        desplaza = (ep == e3) ? 1'b1 : 1'b0;
        incbits = (ep == e3) ? 1'b1 : 1'b0;
        ready = (ep == e5) ? 1'b1 : 1'b0;
    end

    // ==================================================
    // Shift Register Unit
    // Shift register -------------------------------
    always @(posedge clk or posedge reset) begin
        if (reset)
            srdato <= 16'b0;
        else if (desplaza)
            srdato <= {srdato[14:0], adcdats};
    end

    // Assign output sample
    assign sample = srdato;

    // Bit counter ------------------------------
    reg [4:0] cbits; // Counter range 0 to 15
    always @(posedge clk or posedge reset) begin
        if (reset)
            cbits <= 5'b0;
        else if (incbits)
            cbits <= cbits + 1;
    end

    // Last bit signal -------------------------
    always @* begin
        ultimo = (cbits == 5'd15) ? 1'b1 : 1'b0;
    end

    // Input synchronization ------------------
    always @(posedge clk or posedge reset) begin
        if (reset) begin
            adclrcs <= 1'b0;
            bclks <= 1'b0;
            adcdats <= 1'b0;
        end else begin
            adclrcs <= adclrc;
            bclks <= bclk;
            adcdats <= adcdat;
        end
    end

endmodule
```
