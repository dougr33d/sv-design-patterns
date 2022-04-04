
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
assign out = sel0 ? in0 : in1;
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
// AOMUX 2c, GOOD: this code is more explicit and less likely to be broken  
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



# Module Instances #

SystemVerilog provides four syntaxes for specifying port connections when instantiating a module: positional port connections, named port connections, and `.name` implicit port connections, and `.*` implicit port connections.

## Module instances with positional ports

The simplest way to specify ports on a module instance is through positional ports:  
~~~verilog
module top();

fred fred_inst (clk, reset, d, c, b, a);
endmodule
~~~

It might surprise you that this is also, for many module instances, a *terrible* way to specify ports -- it's very prone to errors, very difficult to maintain, and completely non-obvious.  We can surmise the meaning of the first two ports -- `clk` and `reset` -- from the signal names.  But what of the others?  Are they all inputs?  All outputs?  Some mixture of the two?  It's impossible to know, and if the module signature changes by adding, removing, or reorganizing ports, this module instance might break without you finding out immediately.

The one time it's okay to use positional ports is for instantiating library elements that are used pervasively through the design.  If you have a simple library element, used pervasively, defined like this:

~~~verilog
module dff #(parameter WIDTH=1) (
	output logic[WIDTH-1:0] Q,
	input  logic[WIDTH-1:0] D,
	input  logic            clk
);
~~~

it's probably fine to use positional ports in your instances:

~~~verilog
dff dffAout ( Aout, Ain, clk )
/* ... */
dff #(12) dffApg ( Apgstg, Apg, clk );
/* ... */
dff #(46) dffpa (phys_addr, phys_addr_nxt, clk );
~~~

After typing the `dff` instantiation a few thousand times, the port order will be obvious at first glance and any additional information will be superfluous.

## Module instances with named ports
Named ports have been supported since before SystemVerilog, and they continue to be one of the cleanest ways to instantiate modules.  Returning to the first example in the above section:

~~~verilog
module top();

fred fred_inst (
	.clk ( clk    ) , 
	.rst ( reset  ) , 
	.out ( d      ) , 
	.in0 ( c      ) , 
	.in1 ( b      ) , 
	.in2 ( a      )
);
endmodule
~~~
It's not concise, exactly, but it gets the job done!  The explicit port names communicate important information about the underlying module instance: which ports are defined, and how they are connected.  You should be able to easily imagine reversing the `d`, `c`, `b`, and `a` ports on an instance with positional ports, which could be a gnarly bug.  With named ports, so long as you know the semantics of the submodule, you'll never make that mistake.

## Module instances with `.name`
It will often be the case that a module instance has ports you would like to connect to nets of the same name.  For example, in the `fred_inst` instance above, `clk` is connected to `clk`.  Rather than typing the name twice, SystemVerilog provides a `.name` syntax for cleaning up port lists:

~~~verilog
module top();

logic clk;
logic reset;

fred fred_inst (
	.clk, // <-- only specified once
	.rst ( reset  ) , 
	.out ( d      ) , 
	.in0 ( c      ) , 
	.in1 ( b      ) , 
	.in2 ( a      )
);
endmodule
~~~

If we further rename our `reset` net in `top()`, we can similarly shorten the second port connection, while shaving off one line of code:

~~~verilog
module top();

logic clk;
logic rst; // <-- was: reset

fred fred_inst (
	.clk, .rst, // <-- so fresh and so clean!
	.out ( d      ) , 
	.in0 ( c      ) , 
	.in1 ( b      ) , 
	.in2 ( a      )
);
endmodule
~~~

We often don't really care about the `clk` and `rst` signals; they're just part of the landscape most of the time.  It's important to know that they're sunk by `fred_inst`, but typing them twice is just overkill.

## Module instances with `.*`
SystemVerilog provides a second way to infer port connections, via `.*`: a directive that tells the compiler that *all unspecified ports should be connected to nets of the same name*.  For starters: **you should almost never use this feature**.

Let's look at `.*` in action:
~~~verilog
module top();

logic clk;
logic rst; // <-- was: reset

fred fred_inst (.*, // <-- clk, reset auto-connected
	.out ( d      ) , 
	.in0 ( c      ) , 
	.in1 ( b      ) , 
	.in2 ( a      )
);
endmodule
~~~
This is okay sometimes, but it leads to a problem: as your code becomes more complex, it will become very difficult to find, at a glance, which instances underneath a module are sinking which nets.  If our `top()` module instantiated ten different instances of ten different modules with `.*`, finding the sinks of `clk` would require us to look through all ten of those submodules, which is clearly not worth the trouble.  

So, in summary:
- you should always prefer named to positional ports, except for pervasively used library elements; and,
- you should prefer implicit `.name` ports to explicit connections if it makes your code cleaner; and,
- you should *almost never* use `.*`, unless you're writing a book about all the bad ways to design hardware.

# Deduplicating I/O with Structs #  
  
A common design pattern is to have many instances -- say, 16 -- of a submodule instantiated underneath a wrapper module, the latter of which might contain control logic to coordinate between the submodules. Logically, the submodules might form a buffer or queue, like the ROB or one of the structures within the MOB. Without loss of generality, the examples here will refer to a Load Queue, a structure that tracks outstanding load transactions.  
  
An initial implementation of a Load Queue might look something like this:  

~~~verilog  
module loadq ( /*...*/ );  
localparam DEPTH=16;  
  
logic   [DEPTH-1:0] leValid;  
addr_t  [DEPTH-1:0] leAddr;  
trait_t [DEPTH-1:0] leTrait;  
logic   [DEPTH-1:0] lePrefetch;  
logic   [DEPTH-1:0] leHighPriority;  
  
logic   [DEPTH-1:0] lePush;  
addr_t  [DEPTH-1:0] lePushAddr;  
trait_t [DEPTH-1:0] lePushTrait;  
logic   [DEPTH-1:0] lePushPrefetch;  
logic   [DEPTH-1:0] lePushHighPriority;  
  
  
loadq_entry entry[15:0] (  
	.clk,  
	.reset,  
	.lePush,  
	.lePushAddr,  
	.lePushTrait,  
	.lePushPrefetch  
	.lePushHighPriority  
	.leValid,  
	.leAddr,  
	.leTrait  
	.lePrefetch,  
	.leHighPriority,  
	/*...*/  
);  
  
endmodule  
  
module loadq_entry (  
	input  logic   clk,  
	input  logic   reset,  
	input  logic   lePush,  
	input  addr_t  lePushAddr,  
	input  trait_t lePushTrait,  
	input  logic   lePushPrefetch,  
	input  logic   lePushHiPriority,  
	  
	output logic   leValid,  
	output addr_t  leAddr,  
	output trait_t leTrait,  
	output logic   lePrefetch,  
	output logic   leHighPriority,  
  
/*...*/  
);  
  
dff                      rgvalid ( leValid,        lePush | (leValid & ~leDone & ~reset), clk );  

dff_en #($bits(addr_t )) rgaddr  ( leAddr,         lePushAddr,         clk, lePush );  
dff_en #($bits(trait_t)) rgtrait ( leTrait,        lePushTrait,        clk, lePush );  
dff_en                   rgtrait ( lePrefetch,     lePushPrefetch,     clk, lePush );  
dff_en                   rghipri ( leHighPriority, lePushHighPriority, clk, lePush );  

endmodule  
~~~

Well, we could certainly do a bit worse, but look at all the redundant code! Look at how many times we typed `HighPriority` -- 7 times by my count, and that’s with the `.name` instance shortcut saving us two additional instances of `HighPriority`. Ditto for `Addr`, `Trait`, and `Prefetch`. Any time you find yourself typing the same code over and over, look for ways to trim the fat. There is usually a small change you can make to the implementation that will make your life easier.  
  
So, here, what specifically can we do? Notice that all of these signals are used the exact same way: they’re sunk in the submodules, registered at `lePush` time, and then driven back out. We should just define a data structure that encapsulates all of the static state of each entry as follows:  
  
~~~verilog
// define the “static” state of each entry instance  
typedef struct packed {  
	addr_t  Addr;  
	trait_t Trait;  
	logic   Prefetch;  
	logic   HighPriority;  
} loadq_entry_static_t;  
  
module loadq ( /*...*/ );  
localparam DEPTH=16;  
  
loadq_entry_static_t[DEPTH-1:0] lePushStatic;  
loadq_entry_static_t[DEPTH-1:0] leStatic;  
loadq_entry entry[15:0] (  
	.clk,  
	.reset,  
	.lePush,  
	.lePushStatic,  
	.leValid,  
	.leStatic,  
	/*...*/  
);  
  
endmodule  
  
module loadq_entry (  
	input  logic clk,  
	input  logic reset,  
	input  logic lePush,  
	input  loadq_entry_static_t lePushStatic,  
	  
	output logic leValid,  
	input  loadq_entry_static_t leStatic,  
  
  
/*...*/  
);  
  
dff #(1) rgvalid ( leValid, lePush | (leValid & ~leDone & ~reset), clk );  
dff_en #($bits(loadq_entry_static_t)) rgstatic ( leStatic, lePushStatic, clk, lePush );  
  
endmodule  
~~~
  
We still have some duplicated code, but now it’s only the “Static” signals that are duplicated. Adding a signal to each entry is as simple as just changing the struct.  You might have noticed that I left `leValid` out of the `loadq_entry_static_t` struct.  Why?  A few reasons.  

First, note that the `loadq_entry_static_t` bits are registered in `rgstatic`, which has a clock gate to only allow the elements to change when `lePush` asserts.  This allows the clock gating to be very aggressive.  Unfortunately, `leValid` has to change when `leDone` asserts as well.  If we put `leValid` in the struct with the rest of the `leStatic` signals, we would need to use a less aggressive clock gate on the rest of the (potentially very many) bits in the struct.

Second, inside the `loadq` module, it is a very common design pattern to need to operate on `leValid[15:0]` as a single bus with reduction operators; for example, you may want to compute something like:
~~~verilog
assign lcAnyValid   = |leValid[15:0];
assign lcAllValid   = &leValid[15:0];
assign lcQueueEmpty = ~|leValid[15:0];
~~~  
SystemVerilog doesn't allow reduction operators across fields of a vector of structs, so if we had somehow cajoled leValid into the struct, none of these operations would work.
