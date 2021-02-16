# stop_watch
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 05.01.2021 15:31:28
// Design Name: 
// Module Name: stopwatch
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////

module displayMux

	( 
	  input wire clk,                             // clock and reset input lines
	  input wire [7:0]hex7, hex6, hex5, hex4, hex3, hex2, hex1, hex0,    // hex digits
	  input wire [7:0] dp_in,                     // 8 dec pts
	  output reg [7:0] an,                        // enable for 7 displays
	  output reg [7:0] sseg                       // led segments
	);

	// constant for refresh rate: 50 Mhz / 2^16 = 763 Hz
	// we will use 3 of the  MSB states to multiplex 8 displays
	localparam N = 20;
	
	// internal signals
	reg [N-1:0] q_reg;
	wire [N-1:0] q_next;
	reg [6:0] hex_in;
	reg dp;
	
	// counter register 
	always @(posedge clk)
              q_reg <= q_next; 
	
	// next state logic 
	assign q_next = q_reg + 1;
	
	// 3 MSBs control 8-to-1 multiplexing (active low)
	always @*
	   case (q_reg[N-1:N-3])
   
	3'b000:
	           begin 
		   an = 8'b11111110;
                   hex_in = hex0;
		   dp = dp_in[0];
                   end
				
	3'b001:
                   begin 
		   an = 8'b11111101;
                   hex_in = hex1;
		   dp = dp_in[1];
                   end
				
	3'b010:
	           begin 
		   an =8'b11111011;
                   hex_in = hex2;
		   dp = dp_in[2];
                   end	

     3'b011:				
                   begin 
		   an = 8'b11110111;
                   hex_in = hex3;
		   dp = dp_in[3];
                   end	
      3'b100:				
                   begin 
             an =8'b11101111;
                 hex_in = hex4;
              dp = dp_in[4];
                    end
      3'b101:			
                    begin 
              an = 8'b11011111;
                 hex_in = hex5;
                dp = dp_in[5];
                     end
       3'b110:				
                  begin     
              an = 8'b10111111;
                hex_in = hex6;
               dp = dp_in[6];
                   end
       3'b111:
                 begin 
               an = 8'b01111111;
                 hex_in = hex7;
                  dp = dp_in[7];
                    end        	
                default:
                   begin 
                   an = 8'b00000000;
		   dp = 1'b1;
                   end
           endcase 			
	
	// hex to seven-segment circuit 
	always @*
        begin 
         case(hex_in)
	       4'h0: sseg[6:0] = 7'b0000001;
	       4'h1: sseg[6:0] = 7'b1001111;
	       4'h2: sseg[6:0] = 7'b0010010;
	       4'h3: sseg[6:0] = 7'b0000110;
	       4'h4: sseg[6:0] = 7'b1001100;
	       4'h5: sseg[6:0] = 7'b0100100;
	       4'h6: sseg[6:0] = 7'b0100000;
	      4'h7: sseg[6:0] = 7'b0001111;
	       4'h8: sseg[6:0] = 7'b0000000;
	       4'h9: sseg[6:0] = 7'b0000100;
	       4'ha: sseg[6:0] = 7'b0001000;
	       4'hb: sseg[6:0] = 7'b1100000;
	       4'hc: sseg[6:0] = 7'b0110001;
	       4'hd: sseg[6:0] = 7'b1000010;
	       4'he: sseg[6:0] = 7'b0110000;
	       default: sseg[6:0] = 7'b0111000; // 4'hf
	  endcase 
	  sseg[7] = dp;
         end 
endmodule 
module stop_watch
	
	(
	  input wire clk,
	  input wire go, stop, clr, 
	  output wire [7:0] d7, d6,d5, d4, d3, d2, d1, d0
	);
	
	// declarations for FSM circuit
	reg state_reg, state_next;           // register for current state and next state     
	
	localparam off = 1'b0,               // states 
	           on = 1'b1;
	        // state register
   always @(posedge clk, posedge clr)
       if(clr)
	 state_reg <= 0;
       else 
         state_reg <= state_next;

   // FSM next state logic
   always @*
       case(state_reg)
         off:begin
	     if(go)
                state_next = on;
	     else 
		state_next = off;
	     end
			
         on: begin
	     if(stop)
               state_next = off;
	     else 
	       state_next = on;
             end
        endcase		
        
        // declarations for counter circuit
     localparam divisor = 50000000;                  // number of clock cycles in 1 s, for mod-50M counter
		//localparam divisor = 40000;
	reg [26:0] sec_reg;                             // register for second counter
	wire [26:0] sec_next;                           // next state connection for second counter
	reg [5:0] d5_reg,d4_reg, d3_reg, d2_reg, d1_reg, d0_reg;       // registers for decimal values displayed on 4 digit displays
	wire [5:0] d5_next, d4_next, d3_next, d2_next, d1_next, d0_next;  // next state wiring for 4 digit displays
	wire d5_en, d4_en, d3_en, d2_en, d1_en, d0_en;                // enable signals for display multiplexing 
	wire sec_tick, d0_tick, d1_tick, d2_tick, d3_tick, d4_tick;       // signals to enable next stage of counting

        // counter register 
	always @(posedge clk)
	    begin
	       sec_reg <= sec_next;
	       d0_reg  <= d0_next;
	       d1_reg  <= d1_next;
	       d2_reg  <= d2_next; 
	       d3_reg  <= d3_next;
	       d4_reg  <= d4_next;
	       d5_reg  <= d5_next;
            end
	
	// next state logic 
	// 1 second tick generator : mod-50M
	assign sec_next = (clr || sec_reg == divisor && (state_reg == on)) ? 6'b0 : 
	                  (state_reg == on) ? sec_reg + 1 : sec_reg;
	
	assign sec_tick = (sec_reg == divisor) ? 1'b1 : 1'b0;
	
	// second ones counter 
	assign d0_en   = sec_tick; 
	
	assign d0_next = (clr || (d0_en && d0_reg == 9)) ? 6'b0 : 
	                  d0_en ? d0_reg + 1 : d0_reg; 
	
	assign d0_tick = (d0_reg == 9) ? 1'b1 : 1'b0;
							
	// second tenths counter 
	assign d1_en = sec_tick & d0_tick; 
	
	assign d1_next = (clr || (d1_en && d1_reg == 5)) ? 6'b0 : 
	                  d1_en ? d1_reg + 1 : d1_reg; 	
							
	assign d1_tick = (d1_reg == 5) ? 1'b1 : 1'b0;
	
	// minute ones counter 
	assign d2_en = sec_tick & d0_tick & d1_tick; 
	
	assign d2_next = (clr || (d2_en && d2_reg == 9)) ?6'b0 : 
	                  d2_en ? d2_reg + 1 : d2_reg;
	
        assign d2_tick = (d2_reg == 9) ? 1'b1 : 1'b0;	
	
	// minute tenths counter 
	assign d3_en = sec_tick & d0_tick & d1_tick & d2_tick; 
	
	assign d3_next = (clr || (d3_en && d3_reg == 5)) ? 6'b0 : 
	                  d3_en ? d3_reg + 1 : d3_reg;
    assign d3_tick = (d3_reg == 5) ? 1'b1 : 1'b0;
        // hour sec counter
    assign d4_en = sec_tick & d0_tick & d1_tick & d2_tick & d3_tick; 
            
    assign d4_next = (clr || (d4_en && d4_reg == 9)) ? 6'b0 : 
                        d4_en ? d4_reg + 1 : d4_reg;
    assign d4_tick = (d4_reg == 9 ) ? 1'b1 : 1'b0;
        // hour min counter       
    assign d5_en = sec_tick & d0_tick & d1_tick & d2_tick & d3_tick & d4_tick; 
                  
     assign d5_next = (clr || (d5_en && d5_reg == 5)) ? 6'b0 : 
                        d5_en ? d5_reg + 1 : d5_reg;
                  
       // route digit registers to outputs      
       
       assign d0 = d0_reg;
       assign d1 = d1_reg; 
       assign d2 = d2_reg;
       assign d3 = d3_reg;
       assign d4 = d4_reg;
       assign d5 = d5_reg; 
  endmodule 

module stopwatch_unit

	( 
	  input wire clk, go, stop, clr,      // clock and reset input lines
	  output wire [7:0] an,               // enable for 8 displays
	  output wire [7:0] sseg              // led segments
	);
	
	wire [7:0] d7, d6, d5, d4, d3, d2, d1, d0;
	
	stop_watch timer1 (.clk(clk), .go(go), .stop(stop), .clr(clr),.d7(d7),.d6(d6), .d5(d5), .d4(d4), .d3(d3), .d2(d2), .d1(d1), .d0(d0));
	
	displayMux display_unit (.clk(clk),.hex7(d7),.hex6(d6), .hex5(d5), .hex4(d4), .hex3(d3), .hex2(d2), .hex1(d1), .hex0(d0), .dp_in(8'b11111111), .an(an), .sseg(sseg));
	
endmodule 






