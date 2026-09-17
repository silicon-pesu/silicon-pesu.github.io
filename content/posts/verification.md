---
title: "Verification Is The Way To Go"
date: "2026-09-17"
description: "Verification basics using SystemVerilog and elective subject notes by Keval"
layout: "post"
authors:
    - name: "Keval Pattani"
      url: "/members/keval/"
tags:
    - "Verification"
    - "SystemVerilog"
    - "Digital"   
---

Verification is a course in my uni for which I am making notes and writing it down in the discord.
`UE24EC343AB1 : Verification of Digital Systems`
The book which I am following to read is SystemVerilog of Verification by Chris Spear

_Layered Testbench_: It a structured verification environment to split the verification tasks into distinct hierarchical abstraction layers to test digital hardware designs efficiently. Instead of connecting signals to directly a single test file layered testbench have different modular modules to support.

Typical Layers of Testbench:

1. _Signal Layer (Interface)_: This is the absolute bottom of any testbench, it consists of physical wires, clocking blocks and raw signals directly connecting to the DUT.

2. _Command / Physical Layer (Driver & Monitor)_: This layer acts as a translator. The Driver acts as a translator. It takes abstract, high-level software data (like a packet containing a message) and converts it into real, low-level electrical signals (ones and zeros) on physical wires at the exact right times. The Monitor passively watches the physical pins, translates those pin wiggles back into high-level packets, and broadcasts them to the rest of the testbench.

3. _Protocol Layer (Agent)_: This layer encapsulates the Driver, Monitor, and a Sequencer into a single reusable block. If you are verifying an AXI bus, you will have an "AXI Agent." This is the core of Verification IP (VIP).

4. _Scenario / Transaction Layer (Sequences)_: generates high-level, protocol-independent data objects to define exactly what operations to execute, isolating your testing intent from the physical clock cycles required to actually transmit those signals across the bus. Basically just passing the data without thinking much about the clock or the drivers

5. _Functional Layer (Scoreboard & Coverage)_: This layer determines if the test passed. The Scoreboard receives transactions from the Monitors, predicts what the DUT should have done, and compares it against what the DUT actually did. The Coverage collector ensures that your randomized stimuli are actually hitting all the required corner cases.

6. _Test Layer_: The highest level of abstraction. The Test layer configures the entire environment and decides which specific sequences to run to achieve a particular verification goal

For every designer U need two verification guys,
every line that you put into overly complicated testbench can eliminate a line in every single test,
A constrained-random test that tries thousands of different protocol variations is worth more than the handful of directed tests that could have been created in the same amount of time.

Block Level testing -> Block-level testing isolates a single IP or module from the rest of the design and tests it.
> Pros: You can easily manipulate specific inputs and monitor internal signals without traversing a complex hierarchy. and smaller design meaning faster simulations times
> --------
> Cons: It cannot verify if the block communicates correctly with other components (e.g., protocol mismatches on an AXI bus). and it requires writing custom testbenches, modules, drivers and all. also A block might pass all tests but fail instantly when connected to the rest of the system due to misunderstood specifications.

System-Level Testing -> System-level testing instantiates the entire architecture and tests it in one go.
> Pros: Ensures all blocks, buses, and interfaces talk to each other correctly. Allows running actual kernel or firmware and check the software boundary.
> --------
> Cons: Its very slow, finding a root cause is like finding a needle in haystack (e.g., like error might have happened a thousand cycles ago but its reflecting afterwards maybe cause of a sub-module issue), Its difficult to write a testbench on system level that reliably forces a specific deep-pipeline stall or minor module error.

Directed Testing -> Directed testing relies on manual, human-generated manual stimulus.
> Pros: Straightforward to implement. You write a specific stimulus and check for a specific expected output. Highly effective for verifying reset sequences. If a directed test fails, you know exactly what the test was trying to do, making root-cause analysis fast. 
> --------
> Cons: Poor scaling as design complexity and the design grows, t only tests what the verification engineer explicitly thought to test, meaning it rarely finds unexpected or obscure bugs. If the design specification changes, many directed tests usually have to be manually rewritten.

Constrained Random Testing (CRT) -> uses a solver to automatically generate random stimulus bounded by specific rules (constraints) you define.
> Pros: It finds very obscure deeply hidden and hard to find bugs which normal testbench can't find. A single testbench can generate thousands of different tests simply by changing the randomization seed. Automatically explores the design's state space, hitting multiple coverage points faster than writing manual tests.
> --------
> Cons: Requires complex verification architectures (like UVM in SystemVerilog) and object-oriented programming to set up. When a test fails, you must first reverse-engineer the random sequence of events the solver generated before you can fix the RTL. Because you don't know exactly what is being tested, you must write extensive "functional coverage" to prove the randomizer actually hit the important scenarios.

_Logic_: is a data type used in system verilog which cannot be driven by multiple structural drivers. A logic signal can be used anywhere a net is used.

SystemVerilog introduces several 2-state data types to improve simulator performance and reduce memory usage, compared to variables declared as 4-state types.

```
2 state Data types:
bit b; // single-bit
bit [31:0] b32; // 32-bit unsigned integer
int unsigned ui; // 32-bit unsigned integer
int i; // 32-bit singed integer
byte b8; // 8-bit signed integer
shortint s; // 16-bit signed integer
longint l; // 64-bit singed integer
real r; // double precision floating point

4 state Data types:
integer i4; // 32-bit signed integer
time t; // 64-bit unsigned integer
logic l: // 1 bit and unsigned by default could go more

-> There are 4 (four) singed 2-state datatypes.
```

```
1. Verilog Built-In Datatypes
4 - States Datatypes
  → Values - 0,1,X,Z
  → reg, integer, time (default value: X)
  → wire, wand, wore, uwire (default value: Z) 
2 - State Datatypes
  → Values - 0,1
  → real, realtime (default value: 0.0)
```

2-state data types usually ignore X or Z, which may cause a lot of trouble if the driver outputs that. So to be safe always check for propagation of unknown values. Use the $isunknown() operator that returns 1 if any bit of the expression is X or Z.
Like look at this example:
```
if ($isunknown(iport) == 1)
    $display("@%0t: 4-state value detected on iport %b", $time, iport);
```

The format `%0t` and the argument `$time` print the current simulation time, formatted as specified with the `$timeformat()` routine.

So how do you test the default value? 

Example: 1
```
module test;
  <your_wish_of_the_datatype> a; // could any 4-state datatype
  initial 
  begin
    $display(“The value of a = %b”,a);
  end
endmodule
```

Example: 2
```
module test;
  time a;
  initial
  begin
    #20 = a;
    #30 = a;
    $display (“the value of a = %0t”, a);
  end
endmodule
```

Example: 3
```
module test;
  real a;
  initial
  begin
    a = 5.656;
    $display (“The value of a = ”,a);
  end
endmodule
```

Verilog (`2001 verilog standards`) Datatypes were strictly divided into two different categories;
1. `Nets` → `wire`
2. `Variables` → `Reg`

What is the difference between X and Z?
> 1. X - Unknown (Indeterminate) State
>     Node is driven by something, the simulator knows that a signal exists. But it doesn’t have any idea if the voltage level is 0 or 1.
>     Causes of X are → Uninitialized storage, Multiple Drivers Fighting, Setup/Hold time violations, Out-Of-Bounds arrays.

> 2. Z - High Impedance (Floating) State
>     Node is not driven by anything at all, it is both disconnected from the VDD and GND.
>     Causes of Z are → Unconnected wires, Tri-State Buffers

Example of Logic that how it can be used: 

```
module test;
logic q1,q2,q3,q4,inp;
  initial q1 = 0; // Procedural assignment
  assign q2 = inp; // Continuous assignment
  not not_inst(q3,inp); // q3 is driven by primitive 
  my_diff d1(q4,inp,clk,rst_l); // q4 is driven by module
endmodule
```

Example of freedom of reg:
```
# This one is not allowed in verilog 
reg a;
assign a = sel ? inp1 : inp2; // ERROR
```
```
# This one is allowed in System Verilog
reg a;
assign a = sel ? inp1 : inp2; // OK
```

Now we will see a clog function which will calculate the ceiling of log base 2 and can be used in assigning the number of bits needed.

```
parameter int MEM_SIZE = 256;
parameter int ADDR_WIDTH = $clog2(MEM_SIZE); // $clog2(256) = 8
bit [15:0] mem[MEM_SIZE];
bit [ADDR_WIDTH-1:0] addr;                   // [7:0]
```

Now we will look at what is the verbose format and what is the compact format. 
```
# both are the same
int array2 [0:7][0:3]; // Verbose declaration 
int array3 [8][4]; // compact declaration
```

Logic has a default type for out of bounds and any other errors that u through -> `logic (default: X)`

**Packed and Unpacked Arrays:**

1. Packed
```
bit [7:0] a;
int [3:0][2:0] a;
reg [1:0] a;
```
So basically if you have the bit-select after the variable, like this  _`“[ <and here your address> ]”`_ then it's unpacked otherwise packed.

Packed array s bit-select has the bigger value MSB following LSB smaller limit. Like this `[5:0]` or `[5:3]`.

2. Unpacked
```
bit a [0:7];
int a [0:2][0:3];
reg a [0:1];
```
_Unpacked_ has its own way where first is _LSB on the left_ and following that _MSB on the right_.

_Unpacked will always be after the variable_, **if there is any bit-select after the variable then it will be unpacked.**

`bit [7:0] b_unpack[3];`

Now our point of perspective is from the point of view of a 32 bit CPU while decoding what this syntax means.

`bit [7:0]` → This part just serially dedicates 8 bits in a row of the memory as a single element/storage space.
But but but the unpacked bit-selector is also creating an issue

`bit [7:0] b_unpack [3];` → here as I said we are imagining that we are putting it in a 32-bit RAM, and here it looks like

Here the unused spaces are 3 bytes per address
![Pasted image 20260917094838.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-17%2009-48-37.png)

Now we will see how you can initialize arrays
```
initial begin
    static int ascend[4] = '{0,1,2,3}; // Initialize 4 elements
    int descend[5];

    descend = '{4,3,2,1,0};             // Set 5 elements
    descend[0:2] = '{7,6,5};            // Set just first 3 elements
    ascend = '{4{8}};                   // Four values of 8
    ascend = '{default:42};             // All elements are set to 42
end
```
The syntax `‘{blah… blah… blah…}` is meant for saving the data in an array

Example:
```
initial 
begin
    bit [31:0] src[5], dst[5];
    for (int i=0; i<$size(src); i++)
      src[i] = i;
    foreach (dst[i])
      dst[i] = src[i] * 2;
end
```
`$size` is there for finding how many elements are present in the given array and here its 5, 32-bits long elements. Also the way we declared it makes 32-bits 5 destination elements.

In the `foreach loop`, you specify the array name and an index in square brackets, and SystemVerilog automatically steps through all the elements of the array.

The syntax `‘{4{8}}` means `‘{8,8,8,8}`.

Example:
```
int md[2][3] = '{'{0,1,2}, '{3,4,5}};
initial begin
  $display("Initial value:");
  foreach (md[i,j])
    $display("md[%0d][%0d] = %0d", i, j, md[i][j]);

  $display("New value:");

    // Replicate last 3 values of 5

  md = '{'{9, 8, 7}, '{3{5}}};
  foreach (md[i,j])
    $display("md[%0d][%0d] = %0d", i, j, md[i][j]);
end
```

Example:
```
bit [3:0] [7:0] bytes;
bytes = 32h’CAFE_DADA;
$displayh(bytes,,           // Show all 32-bits
          bytes[3],,       // Most significant byte “CA”
          bytes[3][7]);   // Most significant bit “1” of “CA”
```

![Pasted image 20260917094934.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-17%2009-49-33.png)

Now one more
Example:
```
bit [3:0] [7:0] barray [5];    // 5 elements: packed 4-bytes
bit [31:0] lw = 32'h0123_4567; // Word
bit [7:0] [3:0] nibbles;       // Packed array of nibbles
barray[0] = lw;
barray[0][3] = 8'h01;
barray[0][1][6] = 1'b1;
nibbles = barray[2];           // Copy packed values
```

![Pasted image 20260917095005.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-17%2009-50-04.png)

***barray*** **[< `individual address vertically` >]** **[< `8-bit or a byte element` >]** **[< `individual element in the byte` >]**

`individual address vertically` -> `[5]` in _compact form_ or `[0:4]` unpacked
`8-bit or a byte element` -> `[3:0]` packed
`individual element in the byte` -> `[7:0]` packed

**Deciding between packed and unpacked arrays** based on how you intend to access and monitor the data.

Packed arrays guarantee that the simulator stores the elements contiguously in memory. This allows you to slice and dice the data, treating it as individual elements (like bytes) or as one single scalar value (like a word).

```
// Packed array: 4 contiguous bytes (32 bits total)
bit [3:0][7:0] packed_word;

initial begin
  packed_word = 32'hDEADBEEF; // Assign to the whole scalar at once
  $display("%h", packed_word[0]); // Access a single byte (EF)
end
```

Because packed arrays are treated as a single contiguous vector, you can block on them using the `@` event control operator. If you try to monitor an entire unpacked array, the compiler will throw an error unless you explicitly list every single index.

```
// Unpacked array: 4 separate bytes scattered in memory
bit [7:0] unpacked_word [4]; 

// LEGAL: The simulator can monitor the single contiguous block
always @(packed_word) begin 
  $display("The packed array changed!");
end

// ILLEGAL: Cannot monitor an entire unpacked array at once
always @(unpacked_word) begin 
  // Compiler error!
end

// LEGAL but tedious: Expanding the unpacked array
always @(unpacked_word[0] or unpacked_word[1] /* ... */) begin
  $display("An element in the unpacked array changed!");
end
```

A dynamic array is declared with empty word subscripts `[]`. This means that you do not specify the array size at compile time; instead, give it at run time.

The array is initially empty, so you must call the `new[]` constructor to allocate space, passing in the number of entries in the square brackets.

```
int dyn[], d2[];               // Declare dynamic arrays

initial begin
  dyn = new[5];                   // A: Allocate 5 elements
  foreach (dyn[j]) dyn[j] = j; // B: Initialize the elements
  d2 = dyn;                          // C: Copy a dynamic array
  d2[0] = 5;                          // D: Modify the copy
  $display(dyn[0],d2[0]);      // E: See both values (0 & 5)
  dyn = new[20](dyn);          // F: Allocate 20 ints & copy
  dyn = new[100];                // G: Allocate 100 new ints
                               //    Old values are lost
  dyn.delete();                // H: Delete all elements
end
```

benefits of multidimensional array is -> u can adjust the element size per index
like,
```
module tb;
  int d[][];

  initial begin
    // 1. Construct the first dimension
    d = new[4];

    // 2. Construct the sub-arrays
    foreach(d[i])
      d[i] = new[i+1];

    // 3. Initialize the elements
    foreach(d[i,j])
      d[i][j] = i*10 + j;

    // 4. Query dimensions and print the results
    $display("Outer array size: %0d", d.size());
    
    foreach(d[i]) begin
      $display("Row %0d size: %0d | Contents: %p", i, d[i].size(), d[i]);
    end
  end
endmodule
```

and the output will be:

```
Outer array size: 4
Row 0 size: 1 | Contents: '{0}
Row 1 size: 2 | Contents: '{10, 11}
Row 2 size: 3 | Contents: '{20, 21, 22}
Row 3 size: 4 | Contents: '{30, 31, 32, 33}
```

Here is the correct mapping of the `i` and `j` indices evaluated by the `foreach` loop:

 **`i = 0`** (Size is 1) → `j` is **0**
 **`i = 1`** (Size is 2) → `j` is **0**, and **1**
 **`i = 2`** (Size is 3) → `j` is **0**, **1**, and **2**
 **`i = 3`** (Size is 4) → `j` is **0**, **1**, **2**, and **3**

Because the indices start at `0`, the assignment `d[i][j] = i*10 + j;` works out to:

 `i=0, j=0` → `0*10 + 0` = **0**
 `i=1, j=0` → `1*10 + 0` = **10**
 `i=1, j=1` → `1*10 + 1` = **11**
 `i=2, j=0` → `2*10 + 0` = **20**
 `i=2, j=1` → `2*10 + 1` = **21**
and so on...........

Queues vs Dynamic Arrays:
-> Queues automatically grow and shrink as you add or remove elements. You never need to use the `new[]` constructor.
-> They come with native methods optimized for adding and removing data, such as `.push_back()`, `.push_front()`, `.pop_back()`, `.pop_front()`, `.insert()`, and `.delete()`.

-> You must explicitly write `new[]` to allocate memory.
-> To resize a dynamic array, the simulator must allocate a brand new block of memory and copy the old elements into it (e.g., `d = new[10](d);`). Doing this frequently slows down simulation.

Queue methods:

```
int j = 1, q2[$] = {3,4}, q[$] = {0,2,3}; // Queue literals do not use this ` tick sign 

initial begin
  q.insert(1,j);      // {0,1,2,3} Insert j before ele #1
  q.delete(1);        // {0,2,3} Delete ele #1

// These operations are quite fast on the simulation time
  q.push_front(6);    // {6,0,2,3} Insert at front
  j = q.pop_back;     // {6,0,2} j = 3
  q.push_back(8);     // {6,0,2,8} Insert at back
  j = q.pop_front;    // {0,2,8} j = 6
  foreach (q[i])      
    $display(q[i]);   // printing the queue
  q.delete();         // deleting the queue
end
```

The LRM does not allow inserting a queue in another queue using the above methods, though some simulators permit this.
You can use word subscripts and concatenation instead of methods. As a shortcut, if you put a `$` on the left side of a range, such as `[$:2]`, the `$` stands for the minimum value, `[0:2]`. A `$` on the right side, as in `[1:$]`, stands for the maximum value, `[1:2]`

```
int j = 1, q2[$] = {3,4}, q[$] = {0,2,5}; // Again remember queue literals don't use ` tick sign

initial begin
  q = {q[0], j, q[1:$]};    // {0,1,2,5} Insert 1 before 2
  q = {q[0:2], q2, q[3:$]}; // {0,1,2,3,4,5} Insert queue in q
  q = {q[0], q[2:$]};       // {0,2,3,4,5} Delete ele #1

  // These operations are fast
  q = {6, q};               // {6,0,2,3,4,5} Insert at front
  
  j = q[$];                 // {6,0,2,3,4,5} j = 5 without damaging the queue
  q = q[0:$-1];             // {6,0,2,3,4} pop_back by slicing the queue
  
  q = {q, 8};               // {6,0,2,3,4,8} Insert at back
  
  j = q[0];                 // {6,0,2,3,4,8} j = 6 just storing the value in j
  q = q[1:$];               // {0,2,3,4,8} pop_front by slicing the queue
  
  q = {}                    // {} deleting the queue contents
end
```

The queue elements are stored in contiguous locations, so it is efficient to push and pop elements from the front and back. This takes a fixed amount of time no matter how large the queue. Adding and deleting elements in the middle of a queue requires shifting the existing data to make room. The time to do this grows linearly with the size of the queue.

Associative arrays : They are abstract data structures that store data as ***key-value*** pairs, where each unique key maps to a specific value. While a sparse matrix can be implemented using an associative array (by using the row and column coordinates as the key), the associative array itself is a general-purpose structure typically powered by underlying mechanisms like hash tables or search trees to allow fast data retrieval using any data type (such as strings) as a key.

```
byte assoc[byte], idx = 1;
initial begin
  // Initialize widely scattered values
  do begin
    assoc[idx] = idx;
    idx = idx << 1;
  end while (idx != 0);

/*
by doing what is described above we are shifting a 8 bit number 0000_0001 to the left and saving that as the key and the value at the same time, if the idx number sets to zero (0) then we are terminating the while loop and moving ahead.
*/

  // Step through all index values with foreach
  foreach (assoc[i])
    $display("assoc[%h] = %h", i, assoc[i]);

/*The syntax above describes the ability of foreach loop statement to step through all index values of the associative array, where the i is just a number/integer to iterate through the exact keys where the value is stored*/

  // Step through all index values with functions
  if (assoc.first(idx))   // Get first index
    do
      $display("assoc[%h]=%h", idx, assoc[idx]);
    while (assoc.next(idx));  // Get next index

/*This code written above is equivalent to the one above this one, the statement assoc.first(idx) saves the first key of the associative matrix in the idx variable which is supposed to be and is the same data type as the associative array, following that there is a do while loop to iterate through the associative loop and increment the key by assoc.next(idx) which updates the idx by storing the next key storing a value*/

  // Find and delete the first element
  void'(assoc.first(idx));
  void'(assoc.delete(idx));

/*Above code uses void' statement which ignores the value of the return value by the statement which is being executed, the assoc.first(idx) has a return value 1 for success and 0 for the failure of the operation. Since we just need the statement to execute and make specific change and not care about the return value we prefix the call with void'() syntax*/

  $display("The array now has %0d elements", assoc.num());
end
```

You can find the number of elements in an associative array with the num or size functions

```
/* Input file contains:
   42   mix_address
   1492 max_address
*/

int switch[string], min_address, max_address, i, file;
initial begin
  string s;
  file = $fopen("switch.txt",r); // 
  while (! $feof(file)) begin
    $fscanf(file, "%d %s", i, s); // getting the number and the string like 42 min_address
    switch[s] = i;
  end
  $fclose(file);  

  min_address = switch["min_address"];

  if (switch.exists("max_address"))
    max_address = switch["max_address"];
  else
    max_address = 1000;

  foreach (switch[s])
    $display("switch['%s']=%0d", s, switch[s]);
end
```
self explanatory code

u can also do,
```
int power_of_2[int] = '{0:1, 1:2, 2:4};
initial begin
  for(int i = 3; i < 5; i++)
    power_of_2[i] = 1 << i;
  $display("%p", power_of_2); // '{0:1, 1:2, 2:4, 3:8, 4:16}
end
```

Array Methods:

Array reduction operations:
```
byte b[$] = {2, 3, 4, 5};
int w;
w = b.sum();      // 14        = 2 + 3 + 4 + 5
w = b.product();  // 120       = 2 * 3 * 4 * 5
w = b.and();      // 0000_0000 = 2 & 3 & 4 & 5
```

SystemVerilog does not have a method specifically for choosing a random element from an array, so use the index `$urandom_range(array.size()−1)` for queues and dynamic arrays, and `$urandom_range($size(array)−1)` for fixed arrays, queues, dynamic and associative arrays.
   
```
   int f[6] = '{1,6,2,6,8,6};        // Fixed-size array
   int d[] = '{2,4,6,8,10};          // Dynamic array
   int q[$] = '{1,3,5,7}, tq[$];     // Queue
   
   tq = q.min();                     // {1}
   tq = q.max();                     // {10}
   tq = f.unique();                  // {1,6,2,8}
```


```
int d[] = '{9,1,8,3,4,4}, tq[$];

// Find all elements greater than 3
tq = d.find with (items > 3);
tq.delete();

// Equivalent Code
foreach(d[i])
	if (d[i] > 3)
		tq.push_back(d[i]);

tq = d.find_index with (item > 3);      // {0,2,4,5}
tq = d.find_first with (item > 99);     // {} - non found
tq = d.find_first_index with (item==8); // {2} cause d[2] = 8
tq = d.find_last with (item==4);        // {4}
tq = d.find_last_index with (item==4);  // {5} cause d[5] = 4

-------------------------- If the Tq is a dynamic array

int d[] = '{9,1,8,3,4,4}, tq[];

// Find all elements greater than 3
tq = d.find with (items > 3);
tq.delete();

// Equivalent Code
foreach(d[i])
	if (d[i] > 3)
		tq = new[tq.size() + 1](tq);
		tq[tq.size() - 1] = d[i];

tq = d.find_index with (item > 3);      // {0,2,4,5}
tq = d.find_first with (item > 99);     // {} - non found
tq = d.find_first_index with (item==8); // {2} cause d[2] = 8
tq = d.find_last with (item==4);        // {4}
tq = d.find_last_index with (item==4);  // {5} cause d[5] = 4
```

In a _with_ clause, the name _item_ is called the _iterator argument_ and represents a single element of the array.

```
tq = d.find_first with (item==4);       // These
tq = d.find_first() with (item==4);     // are
tq = d.find_first(item) with (item==4); // all
tq = d.find_first(x) with (x==4);       // equivalent
```

The code array locator methods

```
int count, total, d[] = '{9,1,8,3,4,4};

count = d.sum(x) with (x > 7);          // 2=sum{1,0,1,0,0}
total = d.sum(x) with ((x > 7) * x);    // 17=sum{9,0,8,0,0,0}
count = d.sum(x) with (x < 8);          // 4=sum{0,1,0,1,1,1}
total = d.sum(x) with (x < 8 ? x : 0);  // 12=sum{0,1,0,3,4,4}
count = d.sum(x) with (x == 4);         // 2=sum{0,0,0,0,1,1}
```

array sorting and operations:

```
int d = '{9,1,8,3,4,4};
d.reverse(); // '{4,4,3,8,1,9}
d.sort();    // '{1,3,4,4,8,9}
d.rsort();   // '{9,8,4,4,3,1}
d.shuffle(); // '{9,4,3,8,1,4}
```
The _reverse_ and _shuffle_ methods have no with-clause, so they work on the
entire array.

Use a fixed-size or dynamic array if it is accessed with consecutive positive integer indices: 0, 1, 2, 3… Choose a fixed-size array if the array size is known at compile time, or choose a dynamic array if the size is not known until run time.

Use associative arrays instead of fixed-arrays cause what if the memory size u want to simulate is humongous. 

Queues have almost the same access time as a fixed-size or dynamic array for reads and writes. The first and last elements can be pushed and popped with almost no overhead. Inserting or removing elements in the middle requires many elements to be shifted up or down to make room. If you need to insert new elements into a large queue, your testbench may slow down, so consider changing how you store new elements.

When reading and writing associative arrays, the simulator must search for the element in memory. The LRM does not specify how this is done, but popular ways are hash tables and trees. These require more computation than other arrays, and therefore associative arrays are the slowest.

You can create new types using the typedef statement.

User-defined type-macro in Verilog

```
`define OPSIZE 8
`define OPREG reg [`OPSIZE-1:0]

`OPREG op_a, op_b;
```

User-defined type in SV

```
parameter OPSIZE = 8;
typedef logic [OPSIZE-1:0] opreg_t;

opreg_t op_a, op_b;
```

uint is unsigned integer.
and the definition for it is,
```
typedef bit [31:0] uint;   // 32-bit unsigned
typedef int unsigned uint; // equivalent of that
```

User-defined array type

```
typedef int fixed_array5_t[5];
fixed_array5_t f5;

initial begin
	foreach (f5[i])
		f5[i] = i;
end
```

User-defined associative array index

```
typedef bit[63:0] bit64_t;
bit64_t assoc[bit64_t], idx = 1;
```

a user-defined composite data type that allows you to **group multiple variables of different data types** under a single name

```
typedef struct {
	int id;
	string name;
	bit active
} employee_t;

employee_t emp1, emp2;

--------------------------

struct {bit [7:0] r, g, b;} pixel;

--------------------------

typedef struct {bit [7:0] r, g, b;} pixel_s;
pixel_s my_pixel;
```

initializing a structure

```
initial begin
	typedef struct {int a;
					byte b;
					shortint c;
					int d;} my_struct_s;
	my_struct_s st = '{32'haaaa_aaa,
					   8'hbb,
					   16'hcccc,
					   32'hdddd_dddd};
	$display("str = %x %x %x %x", st.a, st.b, st.c, st.d);
end
```

packed sturcture are more synthsizable then unpacked, also use packed more cause unpacked requires to occupy unnessesory memory.

When you are trying to choose between packed and unpacked structures, consider how the structure is most commonly used and the alignment of the elements. If you plan on making aggregate operations on the structure, such as copying the entire structure, a packed structure is more efficient. However, if your code accesses the individual members more than the entire structure, use an unpacked structure. The difference in performance is greater if the elements are not aligned on byte boundaries, have sizes that don’t match the typical byte, or have word instructions used by processors. Reading and writing elements with odd sizes in a packed structure requires expensive shift and mask operations.

Package declaration:

```
// The ABC Bus Package
package abc_pkg;
  parameter WIDTH = 32;   // The "John" of the ABC family
  typedef logic [7:0] data_t;
endpackage

// The XYZ Bus Package
package xyz_pkg;
  parameter WIDTH = 64;   // The "John" of the XYZ family
  typedef logic [15:0] data_t;
endpackage
```

Package use:

```
module processor;
  // We specify exactly which WIDTH we want using the double-colon
  logic [abc_pkg::WIDTH-1 : 0] abc_data; 
  logic [xyz_pkg::WIDTH-1 : 0] xyz_data;
endmodule
```

one more example

```
package ABC;
  parameter int abc_data_width = 32;
  typedef logic [abc_data_width-1:0] abc_data_t;
  parameter time timeout = 100ns;
  string message = "ABC done";
endpackage // ABC
```
This is for the package definition is up here

now use
```
module test;
  import ABC::*;                 // Search ABC for symbols

  abc_data_t data;               // From package ABC
  string message = "Test timed out"; // Hides message in ABC

  initial begin
    #(timeout);                  // From package ABC
    $display("Timeout - %s", message);
    $finish;
  end
endmodule
```
The module implementation demonstrating the wildcard import is up here

Down here we are importing selected symbols from a package
```
module test;
  import ABC::*;                 // Search ABC for symbols
  import XYZ::timeout;           // Just import timeout
  string message = "Test timed out"; // Hides message in ABC

  initial begin
    #(timeout);                  // From package XYZ
    $display("Timeout - %s"\, message);
    $finish;
  end
endmodule
```

Packages can only see symbols defined inside themselves, or packages that they import. You can not have hierarchical references to symbols such as signals, routines, or modules from outside the package. Think of a package as being completely standalone, able to plug in where needed, with no outside dependencies.

the _streaming operators_ `<<` and `>>` take an expression, structure, or array, and packs it into a stream of bits. The `>>` operator streams data from left to right while `<<` streams from right to left.

```
initial begin
	int h;
	bit [7:0] b, g[4], j[4] = '{8'ha, 8'hb, 8'hc, 8'hd};
	bit [7:0] q, r, s, t;
	
	h = { >> {j}};              // 0a0b0c0d pack array into int
	h = { << {j}};              // b030d050 reverse bits
	h = { << byte {j}};         // 0d0c0b0a reverse bytes
	{>>{g}} = { << byte {j}};   // 0d,0c,0b,0a unpack into array
	b = { << {8'b0011_0101}};   // 1010_1100 reverse bits
	b = { << 4 {8'b0011_0101}}; // 0101_0011 reverse nibble
	{ >> {q, r, s, t}} = j;     // Scatter j into bytes
	h = { >> {t, s, r, q}};     // Gather bytes into h
end
```

Converting between queues with streaming operator

```
initial begin
	bit [15:0] wq[$] = {16'h1234, 16'h5678};
	bit [7:0] bq[$];
	
	// convert word array to byte
	bq = { >> {wq}}; // 12 34 56 78
	
	// convert byte array to words
	bq = {8'h98, 8'h76, 8'h54, 8'h32};
	wq = { >> {bq}}; // 9876 5432
end
```

Is `logic my_array[0:255]` same as `logic my_array[255:0]`?
-> Nope, but if we compare it on a silicon level where everything is synthesized then yes it is, but if we compare on the compiler level then no.
-> To get more clarity lemme tell u this,
-> **`logic my_array[255:0]` (Descending):** The compiler considers index `255` to be the leftmost element, and `0` to be the rightmost.
-> **`logic my_array[0:255]` (Ascending):** The compiler considers index `0` to be the leftmost element, and `255` to be the rightmost.
-> If u attempt to do my_array_ascending = my_array_descending; then compiler would throw an error a type-mismatch cause their directional shapes are different.
-> but if u use streaming operator and copy then like this, 
my_array_ascending = { >> {my_array_descendin}}; it will be reversed for the my_array_ascending. 
-> so in this case u would have to do my_array_ascending = { << {my_array_descendin}};

Let's talk about packed and unpacked dimention trap
-> The source `bit [7:0] src[255:0]` (unpacked): This means u have 256 distinct items each 8 bits.
-> The mistake would be `bit [7:0] [255:0] dst` (packed) : This immitates 8 distinct items each 256 bits wide.
-> the correct way is `bit [255:0] [7:0] dst`.

```
initial begin
  typedef struct {int a;
                  byte b;
                  shortint c;
                  int d;} my_struct_s;
  my_struct_s st = '{32'haaaa_aaaa,
                     8'hbb,
                     16'hcccc,
                     32'hdddd_dddd};
  byte b[];

  // Covert from struct to byte array
  b = { >> {st}};         // {aa aa aa aa bb cc cc dd dd dd dd}

  // Convert from byte array to a struct
  b = '{8'h11, 8'h22, 8'h33, 8'h44, 8'h55, 8'h66, 8'h77,
        8'h88, 8'h99, 8'haa, 8'hbb};
  st = { >> {b}};         // st = 11223344, 55, 6677, 8899aabb
end
```

An enumerated type allows you to create a set of related but unique constants such as states in a state machine or opcodes. In classic Verilog, you had to use text macros. Their global scope is too broad, and their value might not be visible in the debugger. An enumeration creates a strongly typed variable that is limited to a set of specified names. For example, the names `ADD`, `MOVE`, or `ROTW` make your code easier to write and maintain than if you had used literals such as `8'h01` or _macros_. A weaker alternative for defining constants is a parameter. These are fine for individual values, but an enumerated type automatically gives a unique value to every name in the list.

```
// Create data type for values 0, 1, 2
typedef enum {INIT, DECODE, IDLE} fsmstate_e;
fsmstate_e pstate, nstate;    // declare typed variables

initial begin
  case (pstate)
    IDLE:    nstate = INIT;    // data assignment
    INIT:    nstate = DECODE;
    default: nstate = IDLE;
  endcase
  $display("Next state is %s",
           nstate.name());     // Display symbolic state name
end
```

U can choose your own enumerated values,
`typedef enum {INIT, DECODE=2, IDLE} fsmtype_e;`
Here the INIT is 0, Decode is 2 and following that IDLE is 3

An enumerated type is stored as int unless you specify otherwise. Also please be careful while assigning the initial value which is the first value it shouldn't be other than 0 if so then can make an issue.

Incorrectly specifying enumerated values
```
typedef enum {FIRST=1, SECOND, THIRD} ordinal_e;
ordinal_e position;
```

Correctly specifying enumerated values
```
typedef enum {BAD_O=0, FIRST, SECOND, THIRD} ordinal_e;
ordinal_e position;
```

Routines for Enumerated Types
- `first ()` returns the first member of the enumeration.
- `last ()` returns the last member of the enumeration.
- `next ()` returns the next element of the enumeration.
- `next (N)` returns the N<sup>th</sup> next element.
- `prev ()` returns the previous element of the enumeration.
- `prev (N)` returns the N<sup>th</sup> previous element.

Note that there is no clean way to write a for loop that steps through all members of an enumerated type if you use an enumerated loop variable. You get the starting member with first function and the next member with next. A for loop ends when the loop variable is outside the defined bounds, but the next function always returns a value inside the enumeration. If you use the test `current!= current.last()`, the loop ends before using the last value. If you use current<=current.last(), you get an infinite loop, as next never gives you a value that is greater than the final value. This is similar to trying to make a for loop that steps through the values `0..3` with an index declared as `bit [1:0]`. The loop never exits! You can get around this limitation by either using an integer variable in the loop, or incrementing the enumerated variable, but both of these solutions can give illegal values if your enumerated values are not contigious, such as 1, 2, 3, 5, 8. You can use a do…while loop to step through all the values, checking when the value wraps around, as shown down here.

```
typedef enum {RED, BLUE, GREEN} color_e;
color_e color;
color = color.first;
do
	begin
	$display("Color = %0d/%s", color, color.name());
	color = color.next;
	end
while (color != color.first);
```

If you have ever tried to use a Verilog reg variable to hold a string of characters, your suffering is over. The SystemVerilog string type holds variable-length strings. An individual character is of type byte. The elements of a string of length N are numbered 0 to N-1. Note that, unlike C, there is no null character at the end of a string, and any attempt to use the character “\0” is ignored. Memory for strings is dynamically allocated, so you do not have to worry about running out of space to store the string.

String methods:
- `getc(N)` returns the byte at location N
- `toupper` returns an upper-case copy of the string
- `tolower` returns a lowercase copy
- `putc(M, C)` writes a byte `C` into a string at location `M`
- `substr(start,end)` function extracts characters from location _start_ to _end_

```
string s;

initial begin
  s = "IEEE ";
  $display(s.getc(0));        // Display: 73, ASCII value of 'I'
  $display(s.tolower());      // Display: 'ieee '

  s.putc(s.len()-1, "-");     // change ' '-> '-'
  s = {s, "1800"};            // "IEEE-1800"

  $display(s.substr(2, 5));   // Display: EE-1

  // Create temporary string, note format
  my_log($sformatf("%s %5d", s, 42));
end

function void my_log(string message);
  // Print a message to a log
  $display("@%0t: %s", $time, message);
endfunction
```

There are two ways to compare strings, but they behave differently. The equality operator, `s1==s2`, returns _1_ if the strings are identical, and _0_ if they are not. The string comparison function, `s1.compare(s2)`, returns _1_ if _s1_ is greater than _s2_, _0_ if they are equal, and _−1_ if _s1_ is less than _s2_.

```
bit [7:0] b8;
bit one = 1'b1;                // Single bit
$displayb(one + one);          // A: 1 + 1 = 0

b8 = one + one;                // B: 1 + 1 = 2
$displayb(b8);

$displayb(one + one + 2'b0);   // C: 1 + 1 = 2 with constant

$displayb(2'(one) + one);      // D: 1 + 1 = 2 with cast
```

U can put a label on _begin_ or _fork_ statement also u can put the same label on the _end_ or _join_ statement of the same block.

```
initial
begin : example_label

|
........logic here.........
|

end : example_label 
```

if you are in a loop, but want to skip over the rest of the statements and do the next iteration, use `continue`. If you want to leave the loop immediately, use `break`.

```
initial begin
  bit [127:0] cmd;
  int file, c;

  file = $fopen("commands.txt", "r");
  while (!$feof(file)) begin
    c = $fscanf(file, "%s", cmd);
    case (cmd)
      "":       continue;       // Blank line - skip to loop end
      "done":   break;          // Done - leave loop
      ...                       // Process other commands here
    endcase // case(cmd)
  end
  $fclose(file);
end
```

-> The `continue` Statement (Skip): The `case` block checks the value of `cmd`. If it reads an empty string (`""`), it executes `continue`. This command immediately halts the current iteration of the loop. It skips any code below it and jumps straight back to the top of the `while` loop to evaluate the condition and read the next line.

-> **The `break` Statement (Exit):** If the command read is `"done"`, the code executes `break`. This completely terminates the `while` loop immediately. The program stops reading the file and jumps directly to the `$fclose(file)` statement at the bottom to clean up.

Verilog makes a real clear differentiation between tasks and functions. The biggest different is that a task can consume time whereas a function cannot. A function cannot have a delay (like `#10`) or it cannot call a task or blocking statement as `@(posedge clock)` or `wait (ready)` .

SystemVerilog relaxes this rule a little in that a function can call a task, but only in a thread spawned with the _fork… join_none_ statement.

If u have a system verilog task that does not consume time. you should make it a `void function`, which is a function that does not return a value and it can be called anywhere from any task or function. for maximum flexibility, any debug routine should be a void function rather than a task so that it can be called from any task or function.

```
function void print_state();
	$display("%0t: state = %s", $time, cur_state.name());
endfunction
```

In SystemVerilog, if you want to call a function and ignore its return value, cast the result to void.

Ignoring a function's value
```
void' ($fscanf(file, "%d", i));
```

SystemVerilog has more of C style less verbose routine arguments. like this,

```
task mytask2 (output logic [31:0] x,
			  input logic y);
...
endtask
```

Verbose verilog argument

```
task t3;
	input a, b;
	logic a, b;
	output [15:0] u, v;
	bit [15:0] u, v;
	...
endtask
```

Routine arguments with sticky types

```
	task t3(a, b, output bit [15:0] u, v); // Lazy declaration 
	...
	endtask
```



Passing an array using `ref` and `const`

```
function automatic void print_csm11 (const ref bit [31:0] a []);
	bit [31:0] checksum = 0;
	for (int i = 0; i<a.size(); i++)
		checksum ^= a[i];
	$display("The array checksum is %h", checksum);
endfunction
```

The const modifier will prevents the array to be changed.

Always use _ref_ when passing arrays to a routine for best performance. If you don’t want the routine to change the array values, use the _const ref_ type, which causes the compiler to check that your routine does not modify the array.

```
module test;

  // 1. THE BLUEPRINT (Your code)
  // 'a' is just an empty slot waiting for an array to be passed in.
  function automatic void print_csm11 (const ref bit [31:0] a []);
      bit [31:0] checksum = 0;
      for (int i = 0; i<a.size(); i++)
          checksum ^= a[i];
      $display("The array checksum is %h", checksum);
  endfunction

  // 2. THE EXECUTION
  initial begin
      // We create a real, physical array in the testbench
      bit [31:0] my_real_array [] = new[3]; 
      
      // We put actual data into it
      my_real_array[0] = 32'h11111111;
      my_real_array[1] = 32'h22222222;
      my_real_array[2] = 32'h33333333;

      // We CALL the function. 
      // The simulator passes the memory address of 'my_real_array' into 'a'.
      print_csm11(my_real_array); 
  end

endmodule
```

othrwise u change before only, 

```
function automatic void print_csm11 (ref bit [31:0] a []);
    bit [31:0] checksum = 0;
    a[0] = 32'hFFFFFFFF;             // OVERWRITE HAPPENS HERE
    for (int i = 0; i<a.size(); i++)
        checksum ^= a[i];            // Reads the NEW FFFFFFFF value
    $display("...", checksum);
endfunction
```

If you specify the _automatic_ attribute for programs and module, all the routines inside are automatic.

```
function automatic void print_csm (const ref bit [31:0] a[],
								   input bit [31:0] low = 0,
								   input int high = -1);
	bit [31:0] checksum = 0;
	
	if (high == -1 || high >= a.size())
		high = a.size()-1;
	
	for (int i = low; i<=high; i++)
		checksum ^= a[i];
	$display("The array checksum is %h", checksum);
endfunction
```

This code above uses values of input array `a`  and the low and high are endianness where low is LSB and high is MSB. This code take multidimentional array `a` which is `[31:0] a []` 
and then calculate the checksum using XOR operation, firstly they put checksum as 0.

```
module test;
  // 1. Declare a dynamic array (empty at first)
  bit [31:0] my_dyn_array [];

  initial begin
    // 2. Allocate exactly 4 elements. 
    // Valid indices are now 0, 1, 2, 3.
    my_dyn_array = new[4]; 
    
    my_dyn_array[0] = 32'h1111;
    my_dyn_array[1] = 32'h2222;
    my_dyn_array[2] = 32'h3333;
    my_dyn_array[3] = 32'h4444;

    // SCENARIO A: Using defaults
    // You call: print_csm(my_dyn_array)
    // The function receives: low = 0, high = -1
    // a.size() evaluates to 4.
    // The if-statement triggers because (high == -1).
    // high becomes: 4 - 1 = 3.
    // The loop safely runs from index 0 to 3.

    // SCENARIO B: Accidental out-of-bounds
    // You call: print_csm(my_dyn_array, 0, 99)
    // The function receives: low = 0, high = 99
    // a.size() evaluates to 4.
    // The if-statement triggers because (99 >= 4).
    // high becomes: 4 - 1 = 3.
    // The loop safely runs from index 0 to 3 instead of crashing at index 4.
  end
endmodule
```

Now we will see what happens inside the function

```
Initial State:
checksum = 32'h00000000    

Iteration 0 (i = 0):

The code calculates: checksum = checksum ^ a[0]   
Hexadecimal math: 32'h00000000 ^ 32'h00001111
New checksum = 32'h00001111

Iteration 1 (i = 1):

The code calculates: checksum = checksum ^ a[1]
Hexadecimal math: 32'h00001111 ^ 32'h00002222
New checksum = 32'h00003333

Iteration 2 (i = 2):

The code calculates: checksum = checksum ^ a[2]
Hexadecimal math: 32'h00003333 ^ 32'h00003333
 
Note: XORing any number by itself perfectly cancels out to zero.

New checksum = 32'h00000000

Iteration 3 (i = 3):

The code calculates: checksum = checksum ^ a[3]
Hexadecimal math: 32'h00000000 ^ 32'h00004444
New checksum = 32'h00004444

```

When the loop finishes, the `$display` statement will print the final accumulated result of **00004444** to the console.

Ok so can u do this without ref? Absolutely yes, u can directly put dynamic arrays and do the task. Without `ref`, the simulator is forced to allocate a new, temporary block of memory on the stack every time the function is called. It must then copy every single element from your original array into this new temporary array before it can start calculating the checksum. so if u are verifying a large mem block, a video frame, or a dense data payload containing tens of thousands of elements, copying that entire structure into the function just to read it creates severe simulation overhead. It wastes host machine RAM and significantly slows down the simulation.

The fundamental purpose of an XOR checksum is to verify **data integrity** and catch accidental corruption when moving data from one place to another.

Gotta be real careful with Sticky argument type, why? lemme explain

Let's say that u are writing a task header with sticky type,
`task sticky(int a, b);`

The two argument are input integers. As you are writing the task, you realize that you need access to an array, so you add a new array argument, and use the `ref` type so it does not have to be copied.

and then u write it, 

```
task automatic sticky(ref int array[50],
					  int a, b); // What direction are there?
```

So whats the issue here? What argument types are a and b? They take the direction of the previous argument that is a `ref`. Using ref for a simple variable such as an int is not usually needed, but you would not get even a warning from the compiler, and thus would not realize that you were using the wrong direction.

Task header with additional array argument
```
task automatic sticky(ref int array[50],
					  input int a, b); // Be explicit
```

In SystemVerilog, a **routine** is simply a generic umbrella term used to refer to either a `function` or a `task`. It is a reusable block of code that you write once and can execute multiple times.

SystemVerilog adds the return statement to make it easier for you to control the flow in your routines.

Return in a task
```
task automatic load_array(input int len, ref int array[]);
	if (len <= 0) begin
		$display("Bad len");
		return;
	end
	
	// Code for the rest of the task
	...
endtask
```

Return in a function
```
function bit transmit(input bit [31:0] data);
	// Send transaction
	...
	return status; // Return status: 0=error
endfunction
```

The primary difference between these two codes written above lies in **what** is being returned and **why** the execution is stopping. In a `task`, `return` acts solely as an emergency exit. In a `function`, `return` is the delivery mechanism for the final result.

`return` in a Task (Early Exit)
-> What it does: It immediately aborts the execution of the task and jumps back to the calling block. Notice there is no value attached to it (`return;`).
-> Use it for "guard clauses" at the very beginning of a task to check for invalid inputs, hardware timeouts, or conditions where the rest of the task should not run.
-> It prevents deep, messy `if/else` nesting. Instead of wrapping the entire task inside a giant `if (len > 0) begin ... end` block, you simply check the bad condition, print an error, and bail out immediately.

`return` in a Function (Value Delivery)
-> It stops execution and simultaneously passes a specific data value (like `status`) back to the block that called it.
-> Use it when the calling code expects a concrete result, such as a calculated math value, a generated packet, or a success/failure flag. Because your function is declared as `function bit transmit`, it is contractually obligated to hand back a 1-bit value before it finishes.
-> While tasks are generally used to drive hardware signals over time, functions are mathematically modeled to evaluate data and return a single answer. The `return <value>;` statement fulfills that requirement.

```
typedef int fixed_array5_t[5]; // 5-element integer array type
fixed_array5_t f5;

function fixed_array5_t init(input int start);
	foreach (init[i])
		init[i] = i + start;
endfunction

initial begin
	f5 = init(5);
	foreach (f5[i])
		$display("f5[%0d] = %0d", i, f5[i]);
end
```

The output will be,
```
f5[0] = 5
f5[1] = 6
f5[2] = 7
f5[3] = 8
f5[4] = 9
```

One problem here with the preceding code is that the function init creates an array, which is copied into the array f5. If the array was large, this could be a large performance problem.

The alternative is to pass the routine by reference. The easiest way is to pass the array into the function as a `ref` argument.

Passing an array to a function as a ref argument

```
function automatic void init(ref int f[5], input int start);
	foreach (f[i])
	  f[i] = i + start;
endfunction

int fa[5];
initial begin
	init(fa, 5);
	foreach (fa[i])
		$display("fa[%0d] = %0d", i, fa[i]);
end
```

In SystemVerilog, routines still use static storage by default, for both modules and program blocks. You should always make program blocks (and their routines) use automatic storage by putting the automatic keyword in the program statement. 

In Verilog-1995, if you tried to call a task from multiple places in your testbench, the local variables shared common, static storage, and so the different threads stepped on each other’s values. In Verilog-2001 you can specify that tasks, functions, and modules use automatic storage, which causes the simulator to use the stack for local variables.

Specifying automatic storage in program blocks
```
program automatic test();
	task wait_for_bus(input logic [31:0] addr, expect_data,
	                  output logic success);
		while (bus_addr !== addr)
			@(bus_addr);
		success = (bus_data == expect_data);
	endtask
endprogram	
```

This code must be written as a `task` rather than a `function` (which must execute in zero simulation time). The variables `bus_addr` and `bus_data` are implicitly accessed from a higher-level scope, such as a virtual interface or the surrounding module.

The `@(bus_addr)` statement is an **event control trigger** that pauses the execution of the task until the value of the `bus_addr` signal physically changes.

When the loop executes and hits `@(bus_addr)`, this specific task goes to sleep. Simulation time continues to move forward for the rest of your testbench and hardware, but this line of code waits.

**Zero-Delay Prevention:** This is the most critical function of the statement. If you wrote `while (bus_addr !== addr)` without an event trigger inside it, the simulator would check the condition millions of times in the exact same picosecond. The simulation time would never advance, and your software would instantly freeze (a "zero-delay infinite loop"). The `@(bus_addr)` forces the loop to yield time back to the simulator.

`@(bus_addr)` is exactly what allows simulation time to advance. It acts as a yield command to the simulation engine.

Chronological sequence of how `@(bus_addr)` interacts with the `while` loop and simulation time

- **Condition Check:** The `while` loop evaluates `bus_addr !== addr`. If the address on the bus is currently not the target address, the execution steps inside the loop.
    
- **Suspension (Yielding Time):** The code hits `@(bus_addr)`. The task immediately goes to sleep. By stopping execution here, the task gives the simulator permission to move time forward.
    
- **Time Advancement:** With the task safely asleep, the simulator engine focuses on running the rest of your system (like clock generators, external memory, or state machines) and pushes global simulation time forward (e.g., from 10ns to 20ns to 30ns).
    
- **The Trigger:** Eventually, another piece of hardware in your testbench drives a new value onto the `bus_addr` wire.
    
- **Wake Up and Re-evaluate:** The exact picosecond `bus_addr` changes, the simulator wakes this task back up. The code steps past `@(bus_addr)`, reaches the bottom of the loop, and jumps back to the top to re-evaluate `while (bus_addr !== addr)` against the newly updated address.

You can call this task multiple times concurrently, as the addr and expect_data arguments are stored separately for each call. Without the automatic modifier, if you called wait_for_bus a second time while the first was still waiting, the second call would overwrite the two arguments.

Keval wdym by that? -> In SystemVerilog, the `automatic` keyword dictates how the simulator allocates memory for a task's internal variables and arguments, which is critical when managing parallel threads in a testbench.

- **Static Memory (Without `automatic`):** By default, traditional Verilog allocates a single, shared block of physical memory for a task at compile time. If Thread 1 calls `wait_for_bus(Target_A)` and goes to sleep at the `@(bus_addr)` statement, the shared memory holds "Target A". If Thread 2 concurrently calls `wait_for_bus(Target_B)`, it physically overwrites that shared memory with "Target B". When Thread 1 eventually wakes up, it will mistakenly read the overwritten memory and look for Target B, corrupting your verification logic.

- **Dynamic Memory (With `automatic`):** The `automatic` keyword forces the simulator to allocate a brand-new, independent memory stack every single time the task is invoked.

When you define the task as `automatic`, parallel calls cannot interfere with each other. Thread 1 gets its own isolated memory space to store "Target A", and Thread 2 gets a completely separate memory space to store "Target B". Both instances can sleep and wake up independently, evaluating their own protected arguments without cross-contamination.

So what is Thread? -> A thread is an independent, parallel path of execution within your simulation.

In traditional software code execution runs sequentially—one line at a time, from top to bottom. Hardware is different; dozens of components (like buses, clocks, and state machines) operate simultaneously. To accurately model this parallel behavior, the SystemVerilog simulator uses multiple threads to process different blocks of code at the exact same time.

Here is how threads appear in a testbench:

- **Implicit Threads:** Every `initial` block and `always` block automatically runs as its own parallel thread. If you write three `always` blocks in a module, they execute concurrently as three separate threads.

- **Explicit Threads:** You can manually split a single sequential path into multiple parallel threads using a `fork...join` block.

```
initial begin
  // Main thread starts executing

  fork
    // --- THREAD 1 ---
    begin
      wait_for_bus(32'hA000, 32'hFFFF); // Waits for Address A
      $display("Transaction A complete");
    end

    // --- THREAD 2 ---
    begin
      wait_for_bus(32'hB000, 32'hEEEE); // Waits for Address B
      $display("Transaction B complete");
    end
  join

  // Main thread resumes only after both Thread 1 and Thread 2 finish
end
```

Also u may ask so if program is called twice then automatic will look for `@(bus_addr)` ka different mem allocation?

You don't actually "call" a `program` twice. A `program` is instantiated in your testbench exactly once at compile time, just like a physical hardware module.

The concurrency happens when the **task** (`wait_for_bus`) is called multiple times concurrently from different threads _inside_ that single program block.

By declaring `program automatic test();` at the very top, you are applying the `automatic` memory behavior globally to every single task and function defined inside that program.

Here is exactly what happens if you use a `fork...join` block to launch two threads that call the task simultaneously:

- **Thread 1** calls `wait_for_bus` looking for Address A. It hits the `@(bus_addr)` trigger and goes to sleep.

- **Thread 2** concurrently calls `wait_for_bus` looking for Address B. It also hits the `@(bus_addr)` trigger and goes to sleep.

- Because the program is `automatic`, the simulator allocates two separate memory vaults. Thread 1's vault safely holds Address A, and Thread 2's vault safely holds Address B while they both wait.

- If the program lacked the `automatic` keyword, it would only possess one static memory slot for the task's arguments. When Thread 2 made its call, it would instantly overwrite the shared memory to Address B. When Thread 1 eventually woke up on a bus change, it would read the corrupted memory and verify the wrong address.

problem occurs when you try to initialize a local variable in a declaration, as it is actually initialized before the start of simulation.

Static initialization bug
```
program initialization; // Buggy version
	task check_bus();
		repeat (5) @(posedge clock);
		if (bus_cmd == READ) begin
			// When is local_addr initialized?
			logic [7:0] local_addr = addr<<2; // Bug
			$display("Local Addr = %h", local_addr);
		end
	endtask
endprogram
```

The bug is that the variable local_addr is statically allocated, so it is actually initialized at the start of simulation, not when the begin…end block is entered. Once again, the solution is to declare the program as `automatic`.

Static initialization fix: use `automatic`
```
program automatic initialization; // Bug solved
...
endprogram
```

Additionally, you can avoid this by never initializing a variable in the declaration.

A "declaration" is the specific line of code where you introduce a variable to the compiler by assigning it a data type and a name.

"Initializing in the declaration" means setting its starting value on that exact same line.

so basically a fix is,
```
task count_things();
  int count = 0;       // Declaration AND Initialization on one line
  
  // ... task logic
endtask

// This one to avoid
// -----------------------------
// This one is recommended

task count_things();
  int count;           // 1. Declaration only
  
  begin
    count = 0;         // 2. Procedural Assignment (happens during execution)
    // ... task logic
  end
endtask
```

The system task $time returns an integer scaled to the time unit of the current module, but missing any fractional units, while $realtime returns a real number with the complete time value, including fractions.

Deep copy -> question solve it

Static variables exists in the memory we can access the variables alawys, no object is neccessary. 

Arbiter model using ports
```
module arb_with_port (output logic [1:0] grant,
					  input logic [1:0] request,
					  input bit rst, clk);
	always @(posedge clk or posedge rst)
		if (rst)
			grant <= 2'b00;
		else if (request[0])    // High priority
			grant <= 2'b01;
		else if (request[1])    // Low priority
			grant <= 2'b10;
		else
			grant <= '0;
	end
endmodule
```

Testbench module using ports
```
model test_with_port (input logic [1:0] grant,
					  output logic [1:0] request,
					  output bit rst,
					  output bit clk);
	initial begin
		@(posedge clk)
		request <= 2'b01;
		$display("%0t: Drove req=01", $time);
		repeat (2) @(posedge clk);
		if (grant == 2'b01)
			$display("@%0t: Success: grant == 2'b01", $time);
		else
			$display("@%0t: Error: grant != 2'b01", $time);
		$finish;
	end
endmodule
```

Top-level module with ports
```
module top;
	logic [1:0] grant, request;
	bit clk, rst;
	always #50 clk = ~clk;
	
	arb_with_port a1 (grant, request, rst, clk); // Arbiter model using ports
	test_with_port t1 (grant, request, rst, clk); // Testbench module using ports
endmodule
```

![Pasted image 20260911223701.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-11%2022-36-59.png)

Designs have become so complex that even the communication between blocks may need to be separated out into separate entities. To model this, SystemVerilog uses the interface construct that you can think of as an intelligent bundle of wires. It contains the connectivity, synchronization, and optionally, the functionality of the communication between two or more blocks and, optionally, error checking. They connect design blocks and/or testbenches.

![Pasted image 20260911223935.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-11%2022-39-34.png)

Simple interface for arbiter
```
interface arb_if(input bit clk);
	logic [1:0] grant, request;
	bit rst;
endinterface
```

Arbiter using a simple interface
```
module arb_with_ifc (arb_if arbif);
	always @(posedge arbif.clk or posedge arbif.rst)
	begin
		if (arbif.rst)
			arbif.grant <= '0;
		else if (arbif.request[0])    // High Priority
			arbif.grant <= 2'b01;
		else if (arbif.request[1])    // Low Priority 
			arbif.grant <= 2'b10;
		else
			arbif.grant <= '0;
	end
endmodule
```

Testbench using a simple arbiter interface
```
module test_with_ifc(arb_if arbif);
	initial begin
		@(posedge arbif.clk);
		arbif.request <= 2'b01;
		$display("%0t: Drove req=01", $time);
		repeat (2) @(posedge arbif.clk);
		if (arbif.grant != 2'b01)
			$display("@%0t: Error: grant != 2'b01", $time);
		$finish;
	end
endmodule
```

Top module with a simple arbiter interface
```
module top;
	bit clk;
	always #50 clk = ~clk;
	
	arb_if arbif(clk);         // Simple interface for arbiter
	arb_with_ifc a1 (arbif);   // Arbiter using a simple interface
	test_with_ifc t1 (arbif);  // Testbench using a simple arbiter interface
endmodule : top
```

You can see an immediate benefit, even on this small device: the connections become cleaner and less prone to mistakes. If you wanted to put a new signal in an interface, you would just have to add it to the interface definition and the modules that actually used it. You would not have to change any module such as top that just passes the interface through. This language feature greatly reduces the chance for wiring errors.

Connecting an interface to a module that uses ports
```
module top;
	bit clk;
	always #50 clk = ~clk;
	
	arb_if arbif(clk);
	arb_wtih_port a1 (.grant (arbif.grant), // .port (ifc.signal)
					  .request (arbif.request),
					  .rst (arbif.rst),
					  .clk (arbif.clk));
	
	test_with_ifc t1(arbif);
endmodule : top
```

The original modules using ports had this information that the compiler uses to check for wiring mistakes. The modport construct in an interface lets you group signals and specify directions.

```
interface arb_if(input bit clk);
	logic [1:0] grant, request;
	bit rst;
	
	modport TEST (output request, rst,
				  input grant, clk);
	
	modport DUT (input request, rst, clk,
				 output grant);
				 
	modport MONITOR (input request, grant, rst, clk);
	
endinterface
```

The MONITOR modport in the code above allows you to connect a monitor module to the interface.

Arbiter model with interface using modports
```
module arb_with_mp (arb_if.DUT arbif);
	...
endmodule
```

Testbench with interface using modports
```
module test_with_mp (arb_if.TEST arbif);
	...
endmodule
```

Even though the code didn’t change much (except that the interface grew larger), this interface more accurately represents the real design, especially the signal direction.

so use this modports quite good it is.

Top level module with modports
```
module top;
	logic [1:0] grant, request;
	bit clk;
	always #50 clk ~= clk;
	
	arb_if arbif(clk);
	arb_with_mp a1 (arbif.DUT);
	test_with_mp t1 (arbif.TEST);
endmodule
```

With this style, you have the flexibility to instantiate a module more than once, with each instance connected to a different modport, that is, a different subset of interface signals. For example, a byte-wide RAM model could connect to one of four slots on a 32-bit bus. In this case, you would need to specify the modport when you instantiate the module, not in the module itself.

The name `arb_if.TEST.grant` is illegal!

**The Restrictive Way (Hardcoded in the module):** If you define the RAM module like this, you have locked it to `LANE0`. You cannot use this same code for LANE1.

Code snippet
```
// The modport is hardcoded in the module definition
module byte_ram (bus_if.LANE0 bus); 
```

**The Flexible Way (Specified at instantiation):** Instead, you leave the module definition generic. You just tell it to expect a `bus_if`.

Code snippet
```
// Generic definition
module byte_ram (bus_if bus); 
```

Then, in your top-level environment, you specify the exact modport when you instantiate the modules. This allows you to reuse the exact same `byte_ram` code four times, connecting each instance to a different physical slice of the interface:

Code snippet
```
module top;
    bus_if my_bus(clk);

    // Specifying the modport during instantiation
    byte_ram ram0 (my_bus.LANE0);
    byte_ram ram1 (my_bus.LANE1);
    byte_ram ram2 (my_bus.LANE2);
    byte_ram ram3 (my_bus.LANE3);
endmodule
```

A modport is not a physical folder or a nested object; it is simply an access filter (like a pair of tinted glasses) that enforces directional rules (input vs output). Because it is just a filter, it does not add a layer of hierarchy to the signal's physical name.

**Incorrect:**
Code snippet
```
module test_with_mp (arb_if.TEST arbif);
    initial begin
        // ILLEGAL: You cannot use the modport name in the signal path.
        arbif.TEST.request = 1; 
    end
endmodule
```

**Correct:**
Code snippet
```
module test_with_mp (arb_if.TEST arbif);
    initial begin
        // CORRECT: The compiler already knows you are using the TEST filter.
        // Just call the signal directly.
        arbif.request = 1; 
    end
endmodule
```

You can create a bus monitor using the MONITOR modport.
Arbiter monitor with interface using modports

```
module monitor (arb_if.MONITOR arbif);

	always @(posedge arbif.request[0]) begin
		$display("@%0t: request[0] asserted", $time);
		@(posedge arbif.grant[0])
		$display("@%0t: grant[0] asserted", $time);
	end

	always @(posedge arbif.request[1]) begin
		$display("@%0t: request[1] asserted", $time);
		@(posedge arbif.grant[1])
		$display("@%0t: grant[1] asserted", $time);
	end

endmodule
```

The core issue is that SystemVerilog is a **strictly typed** language. To the compiler, an `arb_if` and a `bus_if` (both are interfaces) are completely different data types, just like an integer and a string. Even if they share wires with the exact same names (like `req` and `grant`), the compiler refuses to let them touch directly.

```
// 1. The Small Interface
interface arb_if(input bit clk);
    logic req;
    logic grant;
endinterface

// 2. The Large Interface
interface bus_if(input bit clk);
    logic req;
    logic grant;
    logic [31:0] addr;
    logic [31:0] data;
endinterface

// 3. The Target Module (Strictly expects the small interface)
module arbiter(arb_if a_if);
    always_ff @(posedge a_if.clk) begin
        a_if.grant <= a_if.req; // Dummy logic
    end
endmodule

// 4. The Top-Level Integration
module top;
    bit clk;
    
    // The main system bus used by the rest of your design
    bus_if main_bus(clk);

    // ---------------------------------------------------------
    // THE PROBLEM: 
    // You cannot do this. It will immediately throw a compiler error
    // because 'main_bus' is not of type 'arb_if'.
    // 
    // arbiter u_arb (main_bus); 
    // ---------------------------------------------------------

    // THE SOLUTION: The "Break Out" Adapter Method
    // 1. Instantiate a dedicated small interface just for the arbiter
    arb_if adapter_if(clk);

    // 2. Manually map the overlapping signals wire-by-wire
    assign adapter_if.req = main_bus.req;     // Route request INTO the adapter
    assign main_bus.grant = adapter_if.grant; // Route grant OUT OF the adapter

    // 3. Connect the target module to the adapter
    arbiter u_arb (adapter_if);

endmodule
```

If your testbench drives an asynchronous signal in an interface with a procedural assignment, the signal must be a logic type. A wire can only be driven with a continuous assignment statement. Signals in a clocking block are always synchronous and can be declared as logic or wire.

Driving logic and wires in an interface
```
interface asynch_if();
	logic l;
	wire w;
endinterface

module test(asynch_if ifc);
	logic local_wire;
	assign ifc.w = local_wire;
	
	initial begin
		ifc.l <= 0;       // Drive asych logic directly ....
		local_wire <= 1;  // but drive wire through assign
		...
	end
endmodule
```

In SystemVerilog, data types are broadly categorized into singular (non-composite) and aggregate (composite) types based on whether they contain single independent values or collections of smaller values.

1. What is a Non-Composite (Singular) Type?
-> A **non-composite data type** (officially called a **singular type** in SystemVerilog) represents a **single, individual value**. It has no internal, named sub-structures or sub-elements. Even if it consists of multiple bits (like a wide vector), it is treated mathematically as a single numerical value.
 - **Basic scalar types**: `bit`, `logic`, `reg`, `wire`
 - **Multi-bit vectors**: `logic [31:0]` (treated as one 32-bit number)
 - **Numeric types**: `int`, `integer`, `byte`, `shortint`, `real`
 - **Handles & Enums**: Handles to objects (classes) and enumerated types (`enum`)

2. What is a Composite Type?
->A **composite data type** (officially called an **aggregate type** in SystemVerilog) is a data type made by **combining multiple smaller types** together under a single identifier.
- It acts as a container holding multiple distinct values simultaneously.
- It allows you to perform operations on the container as a whole, or dive inside to read or write specific elements.
There are exactly two kinds of composite types in SystemVerilog:
 - **Unpacked Structures (`struct`)**: Group elements of **different** data types using named members.
 -  **Unpacked Arrays**: Group elements of the **same** data type using numerical indexes (e.g., `int memory [0:255]`).

Clocking blocks are not synthesizable.

An interface should contain a clocking block to specify the timing of synchronous signals relative to the clocks. Clocking blocks are mainly used by testbenches but also allow you to create abstract synchronous models. Signals in a clocking block are driven or sampled synchronously, ensuring that your testbench interacts with the signals at the right time. Synthesis tools do not support clocking blocks, so your RTL code can not take advantage of them. The chief benefit of clocking blocks is that you can put all the detailed timing information in here, and not clutter your testbench.

Interface with a clocking block
```
interface arb_if(input bit clk);
	logic [1:0] grant, request;
	bit rst;
	
	clocking cb @(posedge clk);        // Declaration of cb 
		output request;
		input grant;
	endclocking
	
	modport TEST (clocking cb,         // Use of cb
				  output rst);
				  
	modport DUT (input request, rst, clk,
				 output grant);
				 
endinterface

module test_with_cb(arb_if.TEST arbif);
	initial begin
	  @arbif.cb;
	  arbif.cb.request <= 2'b01;
		  // Dynamically wait for the grant instead of hardcoding repeated lines
	  while (arbif.cb.grant == 2'b00) begin
	    @arbif.cb; // Advance time by one clock cycle
      end
	  $display("@%0t: Grant received = %b", $time, arbif.cb.grant);
	  $finish;
	end
endmodule
```

A clocking block perfectly synchronizes the testbench and DUT by exploiting SystemVerilog's simulation event regions to eliminate zero-time race conditions.
Basically its just made for eliminating race conditions.


**The Physical Hardware Goal** In the real world, a physical chip tester drives voltages into a chip exactly on the clock edge. The chip's internal flip-flops latch that data, process it through logic gates (which takes time), and output the result. The tester waits as long as possible—right before the _next_ clock edge—to sample the output, ensuring the logic has completely settled. A testbench must mimic this "drive on the edge, sample right before the next edge" behavior.

**The Simulation Race Condition** If both your Design Under Test (DUT) and your testbench are written as standard Verilog modules triggering on `@(posedge clk)`, they wake up in the exact same picosecond of simulation time. Because software executes sequentially, the simulator has to pick one to process first.

- If it processes the testbench first, the testbench injects a new value. The DUT then wakes up a microsecond later, reads that new value, and processes it a full clock cycle too early.
- If it processes the DUT first, the DUT reads the old value (which is correct), and then the testbench updates the wire. This unpredictability is a race condition.

The `#0` and `#1` hacks just fails.
-> `#0` fails cause of simulator has to send a lot of stuff at the end so it generally gets overwhelmed and picks an arbitrary order, and the unpredictable bugs return.
-> `#1` if the different verilog modules have different clock then will create a massive problem

***USE CLOCKING BLOCK WHICH PREVENTS ALL THESE ERRORS***

Now let's look at a Race condition between the testbench and design
```
module memory(
    input wire start, write,
    input wire [7:0] addr,
    inout wire [7:0] data
);

logic [7:0] mem[256];

always @(posedge start) begin
    if (write)
        mem[addr] <= data;
    ...
end

endmodule


module test(
    output logic start, write,
    output logic [7:0] addr, data
);

initial begin
    start = 0;              // Initialize signals
    write = 0;
    #10;                    // Short delay
    addr = 8'h42;           // Start first command
    data = 8'h5a;
    start = 1;
    write = 1;
    ...
end

endmodule
```

Here Race condition scenarios are 
- **Scenario A (The Write Fails):** The simulator pauses the testbench and immediately jumps to evaluate the `memory` module. The memory checks `if (write)`. Because the testbench was paused before it could execute the `write = 1;` line, the `write` signal is still `0`. The memory ignores the data.
- **Scenario B (The Write Succeeds):** The simulator decides to finish evaluating the testbench's sequential block first. It executes `write = 1;` and then switches over to the `memory` module. The memory checks `if (write)`, sees the `1`, and successfully saves the data.

Guideline to follow to remove the SystemVerilog Race Conditions:
1. Sequential Logic - use nonblocking assignments
2. Latches - use nonblocking assignments
3. Combinational logic in an always block - use blocking assignments
4. Mixed sequential and combinational logic in the same always block - use nonblocking assignments
5. Do not mix blocking and nonblocking assignments in the same always block
6. Do not make assignments to the same variable from more than one always block - Enforced by _always_comb_, _always_latch_ & _always_ff_.
7. Use `$strobe` to display values that have been assigned using nonblocking assignments.
8. Do not make `#0` procedural assignments.

This is the Time event queue or time step diagram
![Pasted image 20260913133427.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-13%2013-34-26.png)

Steps are:

Active: Simulation of design code in modules
Observed: Evaluation of SystemVerilog Assertions
Reactive: Execution of testbench code in programs
Postponed: Sampling design signals for testbench input

![Pasted image 20260913133717.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-13%2013-37-16.png)

`#1step` -> forces the simulator to read the signal in the **Postponed region** of the exact microsecond _before_ the clock edge hits, ***Used for Sampling***

`#0` -> When the testbench drives a signal into the DUT, it does so with a `#0` skew. But because the testbench (specifically if written in a `program` block, as the text notes) executes in the **Reactive region**, the simulation's Active hardware phase has already finished. ***Used for Driving***

Signal synchronization using `@` and `wait` constructs to synchronize the signals in a testbench
```
program automatic test(bus_if.TB bus);
	initial begin
		@bus.cb;                      // Continue on active edge
		                              // in clocking block
		repeat (3) @bus.cb;           // Wait for 3 active edges
		@bus.cb.grant;                // Continue on any edge
		@(posedge bus.cb.grant);      // Continue on posedge
		@(negedge bus.cb.grant);      // Continue on negedge
		wait (bus.cb.grant==1);       // Wait for expression
		                              // No delay if already true
		@(posedge bus.cb.grant or     
		  negedge bus.rst);           // Wait for several signals
	end
endprogram
```

Synchronous interface sample and drive from module
```
program automatic test(arb_if.TEST arbif);
	initial begin
		$monitor("@%0t: grant=%h", $time, arbif.cb.grant);
		#500ns $display("End of test");
	end
endprogram

module arb_dummy(arb_if.DUT arbif);
	intital
		fork
			#70ns arbif.grant = 1;
			#170ns arbif.grant = 2;
			#250ns arbif.grant = 3;
		join
endmodule
```

The output waveform:
![Pasted image 20260913140559.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-13%2016-05-06.png)

Testbench using interface with clocking block
```
program automatic test_with_cb (arb_if.TEST arbif);
	
	initial begin
		@arbif.cb;
		arbif.cb.request <= 2'b01;
		$display("@%0t: Drove req=01", $time);
		repeat (2) @arbif.cb;
		if (arbif.cb.grant == 2'b01)
			$display("@%0t: Success: grant == 2'b01", $time);
		else
			$display("@%0t: Error: grant != 2'b01", $time);
	end
	
endprogram : test_with_cb
```

exact cycle-by-cycle breakdown of why the testbench must wait:

- **Cycle 0 (The Drive):** The testbench wakes up at the clock edge and drives `request <= 2'b01`. Because clocking block outputs are driven in the Reactive region (software phase), the active hardware phase for this specific clock tick has already passed. The physical wire updates, but the DUT's registers will not latch it yet.

- **Cycle 1 (DUT Processing):** The next clock edge hits. The DUT's RTL (`always_ff`) wakes up in the Active region, latches the `01` request, runs its arbitration logic, and updates its internal `grant` flip-flop.

- **Cycle 2 (The Read):** A second clock cycle passes. The `grant` signal is now stable on the output wire. The testbench's clocking block samples this stable value in the Preponed region. The `repeat(2)` loop finishes, and the testbench successfully evaluates `if (arbif.cb.grant == 2'b01)`.

Testbench using interface with clocking block
```
program automatic test_with_cb(arb_if.TEST arbif);
    initial fork
        #70ns  arbif.cb.request <= 3;
        #170ns arbif.cb.request <= 2;
        #250ns arbif.cb.request <= 1;
        #500ns finish;
    join
endprogram


module arb(arb_if.DUT arbif);
    initial
        $monitor("@%0t: req=%h", $time, arbif.request);
endmodule
```

![Pasted image 20260913163413.png](https://images.kvlp.in/silicon/Screenshot%20From%202026-09-13%2016-34-13.png)
Why 2 not copied here? Reason given below

Differenece between Driving the output and Sampling it.
-> Driving the output (Giving the output from Testbench to the DUT): While giving the output you have the control over the simulation time and environment so by doing `#0` u can drive your output at any second and the DUT will get it, so if u drive ur output at/on the clock edge whatever u have inteded on that exact cycle will replace old value and be sent the DUT and it will process it.

-> Sampling it (Giving the output from DUT to Testbench): As we know that the DUT takes time to settle if the output take on the exact clock edge then it could give a fault full output not the correct one since it could be in-transition period between new output and old. So to solve that we use `#1step` which takes/reads the output exactly behind like 249.9999 seconds before 250 if 250 clock edge for example. and the prev output will settled so nw.

An alternative for `repeat (2) @arbif.cb;` is `##2 arbif.cb.request <= 0;` in the interface

If you want to wait for two clock cycles before driving a signal, you can either use `repeat (2) @arbif.cb;` or use the cycle delay `##2`. This latter delay only
works as a prefix to a drive of a signal in a clocking block, as it needs to know which
clock to use for the delay.

The cycle delay of `##0` in an assignment that drives the value immediately if the clock was asserted in this time slot, according to the clocking block. If the clock was not just asserted, the signal is driven at the next active edge of the clock. The cycle delay of `##1` always waits for the next active edge of the clock, even if the clock was asserted in the current time slot.

double hash (`##`) delays execution by a specific number of clock cycles.

Bidirectional signals in a program and interface
```
interface bidir_if(input bit clk);
    wire [7:0] data;          // Bidirectional signal
    clocking cb @(posedge clk);
        inout data;
    endclocking
    modport TEST (clocking cb);
endinterface


program automatic test(bidir_if.TEST mif);
    initial begin
        mif.cb.data <= 'z;        // Tri-state the bus
        @mif.cb;
        $display(mif.cb.data);    // Read from the bus
        @mif.cb;
        mif.cb.data <= 7'h5a;     // Drive the bus
        @mif.cb;
        $display(mif.cb.data);    // Read from the bus
        mif.cb.data <= 'z;        // Release the bus
    end
endprogram
```

A clocking block ensures that your signals are driven and sampled at the specified clock edge. You can skew these times with either a default statement, or by specifying the delays for individual signals. This can be useful when simulating netlists with real delays.

Clocking block with default statement
```
clocking cb @(posedge clk);
	default input #15ns output #10ns;
	output request;
	input grant;
endclocking
```

Clocking block with delays on individual signals
```
clocking cb @(posedge clk);
	output #10ns request;
	input #15ns grant;
endclocking
```

End of the Simulation
In Verilog, simulation continues while there are scheduled events, or until a `$finish` is executed. SystemVerilog adds an additional way to end simulation. A program block is treated as if it contains a test. If there is only a single program, simulation ends when you complete the last statement in every initial block in the program, as this is considered the end of the test. Simulation ends even if there are threads still running in the program or modules. As a result, you don’t have to shut down every monitor and driver when a test is done.

If there are several program blocks, simulation ends when the last program completes. This way simulation ends when the last test completes. You can terminate any program block early by executing `$exit`. Of course you can still explicitly call $finish to end simulation, but this might cause issues if you have multiple programs.

However, simulation is not yet over. A module or program can have a `final block` that contains code to be run just before the simulator terminates, as shown in the code beloww. This is a great place to perform clean up work such as closing files, and printing a report of the number of errors and warnings encountered. You cannot schedule any events, or have any delays in a `final block` that could cause time to elapse. _You do not have to worry about freeing any memory that was allocated as this will be done automatically_.

```
program automatic test;
    int errors, warnings;
    initial begin
        ... // Main program activity
    end
    final
        $display("Test completed with %0d errors and %0d warnings",
                 errors, warnings);
endprogram
```

Bad clock generator in program block
```
program automatic bad_generator (output bit clk, data);
	initial
		forever #5 clk <= ~clk;
	initial
		forever @(posedge clk)
			data <= ~data;
endprogram
```

Avoid race conditions by always putting the clock generator in a module. If you want to randomize the generator’s properties, create a class with random variables for skew, frequency, and other characteristics.

All clock edges are generated with a blocking assignment to trigger events during the Active region.

If you must generate a clock edge at time 0, use a nonblocking assignment to set the initial value so all clock sensitive logic such as always blocks will have started before the clock changes value.

Good clock generator in module
```
module clock_generator (output bit clk);
	bit local_clk = 0;
	assign clk = local_clk;
	always #50 local_clk = ~local_clk;
endmodule
```

Connecting modules together
```
module top;
    bit clk;
    always #50 clk = ~clk;
    
    arb_if arbif(.*);          // ... arbif(clk) from Sample 4-4
    arb_with_ifc a1(.*);       // ... a1(arbif) from Sample 4-5
    test_with_ifc t1(.*);      // ... t1(arbif) from Sample 4-6
    
endmodule : top
```

Above code uses a shortcut notation `.*` (implicit port connection) that automatically connects module instance ports to signals at the current level if they have the same name and data type.

An Interface in a Port List Must Be Connected
-> SystemVerilog won't compile a single module or program that uses an interface in the port list, if u have not _instantiated it_.

Module with just port connections
```
module uses_a_port(inout bit not_connected);
	...
endmodule
```

The compiler creates wires and connects them to the dangling signals. However, a module or program with an interface in its port list must be connected to an instance of the interface.

Module with an interface
```
// This will not compile without the interface declaration
module uses_an_interface(arb_if.DUT arbif);
    initial arbif.grant = 0;
endmodule
```

Above code the compiler is not able to build even a simple interface. If you have modports or a program block using clocking blocks in an interface, the compiler has an even more difficult time. Even if you are just looking to wring out syntax bugs, you must complete the connections. This can be done as shown in code below.

Top module connecting DUT and interface
```
module top;

    bit clk;
    always #50 clk = !clk;

    arb_if arbif(clk);              // Interface with modport
    uses_an_interface u1(arbif);    // needed to compile this that arbif u write is saving u from compiler errors

endmodule
```

So basically if u only write module with an interface then compiler will throw an error cause it can't see the interface, cause of that we need to instantiate the module by writing the instance name of the interface.

A Top-level scope can be used anywhere in the hierarchy. Top-level scopes like `parameter` and `const`.
Top-level scope for arbiter design
```
// root.sv
`timescale 1ns/1ns
parameter int TIMEOUT = 1_000_000;
const string time_out_msg = "ERROR: Time out";
module top;
	test t1();
endmodule

program automatic test;
	...
	initial begin
		#TIMEOUT;
		$display("%s", time_out_msg);
		$finish;
	end
endprogram
```

the `$root` instance name to resolve scope ambiguities and make absolute cross-module references.
```
module top;
  bit clk;
  test t1(.*);
endmodule

`define TOP $root.top
program automatic test;
  initial begin
    // Absolute reference
    $display("clk=%b", $root.top.clk);$display("clk=%b", `TOP.clk);      // With macro

    // Relative reference
    $display("clk=%b", top.clk);
  end
endprogram
```

This above code sample highlights three approaches to referencing the `clk` variable located in the `top` module from within the `test` program:

- **Absolute reference:** `$root.top.clk` ensures the compiler starts its search at the absolute top-level scope (similar to `/` in Unix), bypassing any potential naming conflicts in intermediate scopes.

- **Macro absolute reference:** `` `TOP.clk `` utilizes a preprocessor macro to store the absolute path, which centralizes path management and makes the code easier to update if the hierarchy changes.

- **Relative reference:** `top.clk` relies on the compiler's default behavior, which searches the local scope first and recursively moves up the hierarchy until the target is found.

**Program-Module Interaction Principles**

- **Unidirectional Visibility:** A `program` block can read signals, write signals, and call routines within a `module`, but a module cannot see into a program. This ensures the design (DUT) remains completely independent of the testbench. Routines are `task` or `function`.

- **Backdoor Access:** Programs can execute routines defined within a module to manipulate internal signal values directly (backdoor loading).

	In hardware verification, there are two primary ways a testbench can interact with the design under test (DUT): **frontdoor** and **backdoor** access.
	
	**Frontdoor Access** This is the normal way your design operates. The testbench drives signals into the module's input ports (pins), respecting the clock cycles, protocols, and timing delays. The data ripples through the logic gates until it reaches the target register or memory. It tests the actual hardware paths but takes simulation time to execute.
	
	**Backdoor Access (Backdoor Loading)** This is a verification shortcut. The testbench instantly reaches deep into the module's internal hierarchy and overwrites a register or memory array in zero simulation time, completely bypassing the input pins, clocks, and normal logic paths.
	
	**Why use routines for backdoor loading?** If your `program` block (testbench) uses absolute paths to force internal signals (e.g., `$root.top.dut.core.alu.internal_state = 1;`), your testbench becomes fragile. If the design changes and `internal_state` is renamed or moved, your testbench breaks.

Instead, you write a routine (a function or task) _inside_ the module to handle the internal assignment, and the testbench simply calls that routine. This creates a clean API for the testbench.

- ``` Example
	  module cpu_core;
	  // Internal, non-public memory array
	  logic [31:0] instruction_memory [0:1023];
	
	  // Routine defined within the module for backdoor loading
	  function void backdoor_load_mem(int address, logic [31:0] data);
	    instruction_memory[address] = data;
	  endfunction
	endmodule
	
	
	program automatic testbench;
	  initial begin
	    // The testbench does not need to know the internal name "instruction_memory"
	    // It simply executes the routine to instantly load the memory
	    cpu_core.backdoor_load_mem(0, 32'hAABBCCDD);
	  end
	endprogram
  ```

- **Signal Forcing:** Because the SystemVerilog standard restricts `program` blocks from forcing signals directly, you must write a `task` within the design module to apply the force, which the program block can then call.

- **Encapsulated Readback:** While a testbench can read module signals directly, it is best practice to encapsulate these read operations within module functions. This isolates the testbench, preventing it from breaking or misinterpreting data if the underlying design implementation changes.

Immediate Assertions: Checking if the assertion is true when the statement is executed. Execute only once, so uses initial procedural blocks.

Checking a signal with an if-statement
```
arbif.cb.request <= 2'b01;
repeat (2) @arbif.cb;
if (arbif.cb.grant != 2'b01)
	$display("Error, grant != 2'b01");
```

Now using simple immediate assertions
```
arbif.cb.request <= 2'b01;
repeat (2) @arbif.cb;
a1: assert (arbif.cb.grant == 2'b01);
```

U can put custom error signals in an immediate signals
```
a40: assert (arbif.cb.grant == 2'b01)
else $error("Grant not asserted");
```

Concurrent Assertion: Checking if the assertion is true during the entire simulation. These are instantiated similarly to other design blocks and are active for the entire simulation. This is used in the always block cause its executed multiple times. 

```
interface arb_if(input bit clk);
    logic [1:0] grant, request;
    bit rst;
    property request_2state;
        @(posedge clk) disable iff (rst)
        $isunknown(request) == 0;  // Make sure no Z or X found
    endproperty
    assert_request_2state: assert property (request_2state);
endinterface
```

Chapter 5 will upload and till then see AV videos cause its so much better. 