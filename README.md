# Common Operations

## Simpler is Better

SystemVerilog has many different ways to express the same logic.  Consider how you might write a simple AND-OR equation:

~~~verilog
// option 1: one way
assign z1 = a & (b | c);

// option 2: another way
always_comb begin
	z2 = a & (b | c);
end

// option 3: yet another way (don't do this!)
always_comb begin
	z2 = 1'b0;
	if (a) begin
		if (b | c) begin
			z2 = 1'b1;
		end
	end
end
~~~

Options 1, 2, and 3 all implement the same logic, but they're ordered in increasing complexity and decreasing desirability.  In SystemVerilog, as with all programming languages, *always prefer the simplest option*.  Since option 1 (the continuous assignment) is the most concise, you should always prefer this option to the procedural blocks.  The second procedural example, option 3, makes my head spin -- so much needless complexity.  

What's worse is that the procedural blocks are bound to grow over time, separating the `a` term from `b|c` across multiple, perhaps several, lines of code.  Why bother?  Use the continuous assignment.

## Muxes
There are fundamentally two types of muxes:
1.  priority muxes, where an encoded select signal chooses between multiple inputs; and,
2.  one-hot muxes, where a decoded, one-hot signal chooses between multiple inputs.
 
Both types of mux are easily implemented in SystemVerilog, but you should take care to use whitespace to communicate the structure of the mux. If your encoded mux only has one select (i.e. a 2-to-1 mux), putting it all on one line is fine:

~~~verilog
// OK if only two inputs  
assign out = sel0 ? in0 :in1;
~~~
 On the other hand, a simple priority mux three or more inputs might be formatted in any number of logically correct ways, but one of them is the clear winner:

~~~verilog
// Option 1, BAD: the mux structure is not obvious from the code.  
assign out = sel0 ? in0 : sel1 & other_quals ? in1 : in2 ;  
  
// Option 2, GOOD: the mux structure is obvious from the code  
assign out = sel0               ? in0 :  
             sel1 & other_quals ? in1 :  
                                  in2 ;
~~~
 
Note that in option 2, the parallelism between `sel0` and `sel1 & other_quals`  is clearly implied by the structure of the code. Likewise with the parallelism between the inputs. That’s a great mux!

What if you have a one-hot mux? These can be easily coded in an AND-OR fashion, but we should again be careful with whitespace to imply the structure of the hardware we’re describing. Also, we should use the ternary operator to implement the AND part, in case our inputs are multi-bit.

~~~verilog
// AOMUX 1, BAD: the mux structure is non-obvious and we're specifying widths when we shouldn't have to.  
assign out[15:0] = {16{sel0 }} & in0[15:0]  
                 | {16{sel1 & qual1}} & in1[15:0]  
                 | {16{sel1 & ~qual1}} & in2[15:0];  
  
// AOMUX 2, GOOD: the mux structure is obvious AND we don't have error-prone constants cluttering the code  
assign out[15:0] = ( sel0          ? in0[15:0] : '0 )  
                 | ( sel1 &  qual1 ? in1[15:0] : '0 )  
                 | ( sel1 & ~qual1 ? in2[15:0] : '0 );
~~~

In an AND-OR mux, it’s important both to have selects that cover all valid cases (i.e. all conditions under which the output is used), and to ensure that the selects are mutually exclusive under valid conditions. The above AND-OR mux design pattern doesn’t help you much with the first case; if you don’t cover all valid conditions, the mux output is going to be all zeros, which is not ideal. The pattern does help with the second case, though, since the parallelism between the selects makes it easier to see whether the truth table is fleshed out.

~~~verilog
// AOMUX 2b, PERNICIOUS: there is a good chance this code is broken  
assign out[15:0] = ( sel0                  ? in0[15:0] : '0 )  
                 | ( sel1 &  qual1         ? in1[15:0] : '0 )  
                 | ( sel1 & ~qual1 & qual2 ? in2[15:0] : '0 );
~~~
 
Do you see the problem? It’s impossible to know whether this is an actual bug without more context, but it is likely that we should have an input to select in the case of `sel1 & ~qual1 & ~qual2`. What if we actually wanted to output zeros in that case? The code works, but it certainly isn't clear! This would be much better:

~~~verilog
// AOMUX 2c, GOOD: there is a good chance this code is broken  
assign in3[15:0] = '0;  
assign out[15:0] = ( sel0                   ? in0[15:0] : '0 )  
                 | ( sel1 &  qual1          ? in1[15:0] : '0 )  
                 | ( sel1 & ~qual1 &  qual2 ? in2[15:0] : '0 )  
                 | ( sel1 & ~qual1 & ~qual2 ? in3[15:0] : '0 );
~~~
  
In AOMUX 2c above, the inclusion of the & ~qual2 term makes the code much more intentional. As with many such changes, our synthesis tool will almost certainly infer that the final leg of the AOMUX is essentially a Don’tCare and optimize it away, so there is really no reason to omit it.

## Parameterized muxes

The mux patterns we saw in the previous section are nice and concise, but they don’t lend well to parameterization. One way to make a parameterized mux would be to use a preprocessing tool like Mako to generate the code. Though I concede that you can make an AOMUX in a manner that is concise and readable with Mako, I generally have a preference for sticking with pure SystemVerilog implementations for cases where such an implementation is comparable. Though this AOMUX is a few more lines of code than a Mako solution might be, I believe it is easy to understand and doesn’t clutter the code with intermixed preprocessing directives.

~~~verilog
localparam WIDTH=16;  
  
// PARAMETERIZED AOMUX 1, GOOD: the always_comb’s innermost statement mirrors our previous AOMUX example  
always_comb  begin  
	out = '0;  
	for (int i=0; i<WIDTH; i+=1) begin  
		out |= selects[i] ? ins[i] : '0;  
	end  
end
~~~

And what of a parameterized priority mux?

~~~verilog
localparam WIDTH=16;  
  
// PARAMETERIZED PRIORITY MUX 1, OKAY: this is admittedly less satisfying than the AOMUX.  
always_comb  begin  
	out = '0;  
	automatic  int found = 0;  
	for (int i=0; i<WIDTH; i+=1) begin  
		out |= (pri_sels[i] & ~found) ? ins[i] : '0;  
		found &= ~pri_sels[i];  
	end  
end
~~~

This is okay, but the inclusion of `found` feels a little clumsy and obscures what the hardware is trying to do. Another way to parameterize a priority mux would be to compute the highest priority input and then do an AND-OR mux:

~~~verilog
localparam WIDTH=16;  
logic[WIDTH-1:0] highest_pri_sel;  
  
// calculate highest priority selector  
assign highest_pri_sel = pri_sels & ~(pri_sels & (pri_sels - 1));  
  
// PARAMETERIZED PRIORITY MUX 2: here we turn the priority select into a one-hot select, and feed it to an AND-OR mux  
always_comb  begin  
	out = '0;  
	for (int i=0; i<WIDTH; i+=1) begin  
		out |= (highest_pri_sel[i] & ~found) ? ins[i] : '0;  
	end  
end
~~~

It is often the case that the surrounding logic already needs to know the highest priority selector. If that is the case, you should probably use the latter implementation.
