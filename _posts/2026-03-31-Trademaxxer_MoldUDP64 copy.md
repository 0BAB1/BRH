---
title: "Trademaxxing part1 : Parsing MoldUDP64"
date: 2026-03-31 15:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction

I'd say I'm a normal guy, but I've always liked very stupid projects.

You may not know it but I have a decent interrest for finance. So today we are exploring how FPGAs are used t process market data in order to make the decisions as fast as possible.

This will in turn allows us to implement shitty financial strats as HDL without any overhead, aloowing us to dilapidate our money at lightning speeds and become poor ASAP.

## WHY FPGAs ?

CPUs are stupid and full of BLOAT.

Unless you are using a baremetal program on the HOLY CORE (the best CPU ever), you delays are aweful and undertermenistic, making C++ developpers and their facy 1ms execution times
cry as they see literral logic gates crush their useless and bloated perfs, which could have been made better by claude.ai anyway. GG for them.

> Us hardware devs still have a couple of years to go before AI replace us, this leaves us some time to mock softwares devs befores we get replaced too, so let me enjy it while it lasts

Anyway, when you wanna capture a spread and/or execute arbitrage strategies, the first to pass orders is the one who wins. These strategies are often dumb simple and to get the bags,
it doesn't come to whom has the best quant but rather whom can buy/sell the fastest as the opportunities are only present for milliseconds.

This is why FGPAS are appreciated. the time it takes to RX an ethernet packet and TX an order can be *very* small, like **under 100ns** but that thre whole point, you have to be first and
no matter if its 1-2 nanosecond before the guy next door, if you are first it's GG and you get the bags.

## MoldUPD64

Okay so In this post, we'll try to figure out how to parse incomming ehternet frames. We won't process any data yet nor keep track of the market state. No. The objective here is simply to
understand how one can get market data, what it is and how to parse the incomming data to get to the PAYLOAD i.e. market data.

### MoldUDP64 & ITCH.. What are thoooose ?

ITCH is the Nasdaq protocol where they'll broacast market events. It's recieved in FPGAs via ethernet and the frames follow the MoldUDP Standard that looks like this:

```
Ethernet Frame -> IPv4 headers -> UDP Headers -> MoldUDP64 -> ITCH Messages
```

Or even better with an image:

![MoldUDP64 Packet view](/assets/img/moldUDP.png)

> I won't get into the specifics of IPv4 and UDP. 

MoldUDP64 is simple and only contain a few fileds:

| Field Name  | Length | Value | Notes |
|---|---|---|---|
| Session | 0 | 10  | Session ID, don't care much |
| Sequence Number | 10 | 8 | First messages sequence number. 1 per message. Can use that and compare with last seq message and check if we missed something. Each message have one so if we get seq = 10 and message count = 5 then the next frame should have nuber 16 as seq number.|
| Message Count | 18 | 2 | The number of messages in the package |

Our goal here will be to filter incomming packet, we only want:

- IPv4 UDP packets
- That contains MoldUPD64

So it's mostly just a big forwarding funnel / pipeline (whatever you wanna call it).

The only thing "out of the ordinary" this pepeline can and will do is detect missing packets using the `Sequence Number` field, which contains the first message's number.

E.g. if you miss a frame and seq number does not match the previous message you handled... it means you missed some orders.

Now Nasdaq being very smart gives you another server you can request missing paquest from. Which we won't do in the context of our demo, instead, when a missing paquet is deteced, we raise a sequential `packet_gap_error` we can later use as a reset signal or an interrupt if I decide to add a monitoring CPU to the system.

Anyway, that's the idea of what we **need to do** to *access* the ITCH messages, again, we don't wanna process them yet.

## FPGA design

![MoldUDP64 Parsing FPGA design](/assets/img/moldudp_fpga.png)

> AXI stream is used a a standard connection between parsers.

So the deisgn is pretty straight forward. ecause this project is more of a demo, te ethernet speed is limited to 1Gps as I plan on using a KC705 board:

![KC705 image](/assets/img/kc705.png)

Each parser revolve around a simple FSM and a byte counter, then look at the upcomming data, parser the respective fields by transitionning states.

If a frame is not valid (has to be litered out), it simply goes back to IDLE and wiat for the data streamto end by monitoring the AXIS' `tlast`.

If a frame passes the couple of of checks, the parser transitions into `FORWARDING` state and will open the gate to the next parser, but it only passes its **payload** meaning the "bloat" headers are dropped at each steps.

Here is an exmaple, the simplest (UDP parsing) where we literally have **no filter** implemented:

```verilog
// udp_parser.sv

/* UDP Parser
*
* This parser is pretty much just a forwarder that cuts out the UDP headers
* As the UDP port, just like some IPV4 params, depend on the day to day instructions from Nasdaq,
* We'll just forward shit for the mist part, very simple design.
*
* BRH 03/2026
*/

module udp_parser (
    input logic clk,
    input logic rst_n,
    AXI_STREAM_BUS.Rx axis_in,
    AXI_STREAM_BUS.Tx axis_out
);

// these states represent 
typedef enum logic [3:0] {
    IDLE,
    SRC_PORT,
    DST_PORT,
    LENGTH,
    HEADER_CHECKSUM,
    FORWARDING
} state_t;
    
state_t state, next_state;
// we track ongoing frame
logic ongoing_frame;
// Global bytes counter
logic [7:0] bytes_counter;

always_ff @(posedge clk) begin
    if(~rst_n) begin
        state <= IDLE;
        ongoing_frame <= 0;
        bytes_counter <= 0;
    end else begin
        state <= next_state;

        // track ongoing frame
        ongoing_frame <= ongoing_frame;
        if(axis_in.tlast) begin
            ongoing_frame <= 0;
        end else if(state == IDLE && axis_in.tvalid) begin
            ongoing_frame <= 1;
        end

        // update byte counter
        if(axis_in.tvalid && (state != IDLE)) begin
            bytes_counter <= axis_in.tlast ? 0 : bytes_counter + 1;
        end else begin
            bytes_counter <= bytes_counter;
        end
    end
end

always_comb begin
    // defaul assigments
    next_state = state;
    axis_out.tvalid = 0;
    axis_out.tdata = 0;
    axis_out.tlast = 0;
    axis_in.tready = 1;

    case (state)
        IDLE : begin
            if(axis_in.tvalid && ~ongoing_frame) next_state = SRC_PORT;
            axis_in.tready = 0;
        end

        SRC_PORT : begin
            if(bytes_counter == 1 && axis_in.tvalid) next_state = DST_PORT;
        end

        DST_PORT : begin
            if(bytes_counter == 3 && axis_in.tvalid) next_state = LENGTH;
        end

        LENGTH : begin
            if(bytes_counter == 5 && axis_in.tvalid) next_state = HEADER_CHECKSUM;
        end

        HEADER_CHECKSUM : begin
            if(bytes_counter == 7 && axis_in.tvalid) next_state = FORWARDING;
        end

        FORWARDING : begin
            // as long as TLAT does not hit, marking the end of the ethernet payload
            // (RX PArser only gives us the raw payload, nothing else)
            // we keep on forwarding the selected data to next parser
            axis_out.tvalid = axis_in.tvalid;
            axis_out.tdata = axis_in.tdata;
            axis_in.tready = axis_out.tready;
            axis_out.tlast = axis_in.tlast;
        end

        default: ;
    endcase

    if(axis_in.tlast && axis_in.tvalid) next_state = IDLE;
end
    
endmodule
```

> In fact, the check are kept minmal for the demo and wha tmost parsers do is just count bytes until thay can start forwarding, effectively just stripping the bloaty protcol header.

## Verification

In summary, the design is **very simple** and it's just a matter of stripping of headers from a stream, nothing fancy.

You can see this simplicty really translate in simulation where we je see stream getting provesively stripped of their headers.

![KC705 image](/assets/img/mold_sim.png)

## Upcomming Work

Now we have to deisng a ITCH bookkeeper, that will decode the nature of the incomming messages and act uppon an order list + "price ladder". The goal will be to only cover a single stock like `APPL` to keep things simple as that is just a demo.

But that... is for another post !

Thank you for reading to this point. You can write a comment below if you have any question.

*Godspeed*

-BRH