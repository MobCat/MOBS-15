# MOBS-16
**MobCat's Own Basic System**<br>
**MobCat's Own Esolang Tar Pit**<br>
Where buffer overflows and monkeys with calculators are features, not bugs.

## Philosophy

MOBS-16 is a destructive, assembly-inspired esoteric programming language where buffer overflows and underflows are features, not bugs.<br>
Every operation has the potential to destroy your data (yes, even the simple random number generator).<br>
The language embraces chaos, undefined behavior, and forces programmers to think carefully about there state.

**Core Principles:**
- **Destructive by default** - Most operations modify or destroy source data in some way
- **Overflow/underflow is intentional** - Wrapping arithmetic and navigation is how you move backward
- **Circular execution** - Programs loop indefinitely until explicitly terminated with `eomf`
- **No multiplication or division** - You must implement these with addition/subtraction loops
- **Hardware-agnostic output** - The S register adapts to whatever hardware you connect it to
- **Nibble-level addressing** - All operations work on 4-bit nibbles (hex digits)

## Architecture

### Registers

MOBS-16 has four registers:

- **M, O, B** - Data registers
  - Each holds 8 nibbles (4 bytes)
  - Range: `00000000` to `FFFFFFFF`
  - Each has an internal cursor for read/write positioning
  
- **S** - Screen/Stack register
  - Unlimited capacity (limited only by hardware)
  - FIFO (First In, First Out) behavior
  - Has an internal cursor for read/write positioning
  - Hardware determines how data is displayed/stored (screen, tape, paper punch, bad apple, etc.)

### Initialization State

When a MOBS-16 program begins:
- **M, O, B** contain whatever garbage was in memory - **uninitialized**
- **All cursors** start at position 0
- **S register** behavior depends on hardware:
  - On `init S` (without value), hardware interprets this as "clear/null from now on"
    Hardware cannot null infinite tape, so it treats everything as null going forward
  - For screens: clears display
  - For tape: resets write position, treats uninitialized areas as null
  regardless of if it is actually null or not. You may end up with weird behavior where the interpreter thinks some part of S is null, but isn't. causing an S de-sync.<br>
  But this behavior should be rare, eomf fixes it. only an issue if you say swap out S with something else while the program is running. basically ripping the ram out of a computer while its running.<br>
  Hot swapping an S registor tape mid computation. MOBS-16 does not have any safety rails to prevent you from corrupting the stack, or well prevent you from doing anything really.<br>
  In fact, we may want to corrupt a register on purpose to get emergent behavior, no limits, no rails, all chaos.

**Embrace the chaos!** M, O, B contain whatever was already in that memory location. Free random numbers on boot! This also means:
- **Best practice:** `init` your registers before using them
- **Chaos practice:** Use them raw for free randomness
- You can treat uninitialized registers as a free `rand` call
> [!NOTE]  
> **Important caveat:**
> If the previous program exited with `eomf`, M, O, B will be 00000000 (cleaned up).<br>
> Free randomness only works on first run or after other programs dirty the memory!

**Details:**
```
Program starts with:
M = ???????? (whatever was in memory - could be garbage, could be 00000000)
O = ???????? (whatever was in memory)
B = ???????? (whatever was in memory)
S = whatever hardware state (usually empty/clear)
All cursors at 0

Safe approach:
init M 00000000    ~ Clean slate

Chaotic approach:
peek M to S        ~ Just use whatever garbage is there! Just shove it into the S register, no questions asked.

Guaranteed random approach:
rand M             ~ Always random, regardless of previous cleanup
```

### Data Format

All values are hexadecimal nibbles (0-9, A-F):
- One nibble = 4 bits = one hex digit
- Two nibbles = 1 byte = one ASCII character
- Eight nibbles = 4 bytes = one register's capacity

**Example:**
```
68656C6C = "hell" in ASCII
  68 = 'h'
  65 = 'e'
  6C = 'l'
  6C = 'l'
```

### Arithmetic Behavior

All arithmetic wraps on overflow/underflow:
```
FFFFFFFF + 00000001 = 00000000
00000000 - 00000001 = FFFFFFFF
```

This wrapping behavior is fundamental to MOBS-16 and is used for backward navigation and fun weird edge cases.<br>Remember it's a feature, not a bug.

### Circular Execution Model

> [!IMPORTANT]
> MOBS-16 programs execute in a continuous loop unless specified otherwise.<br>
> When the interpreter reaches the end of the source file, it wraps back to the beginning automatically. This is intentional behavior.<br>
> You can't have a buffer overflow or underflow if the buffer just loops forever.

The program only stops when:
1. It executes an `eomf` opcode (clean exit with cleanup)
2. External termination (user interrupt, power loss, Hatsune Miku, heat death of the universe, etc.) - no cleanup occurs

This means:
- You don't need explicit loop-back jumps at the end of your code
- The entire program is implicitly one giant loop
- You can place `eomf` anywhere to define the exit point
- You can jump over `eomf` and loop back to it later
- Conditionals can branch to `eomf` for early termination
- Without `eomf`, your program literally runs forever (until something external stops it)

## Opcodes

MOBS-16 has exactly 16 opcodes, each exactly 4 characters long.

|OP Code|Description|
|---|---|
|`init`|Initialize and overwrite|
|`adds`|Addition|
|`subs`|Subtraction|
|`move`|Destructive transfer|
|`dupe`|Semi-destructive copy|
|`jump`|Navigation|
|`ifeq`|If equal|
|`ifgt`|If greater than|
|`iflt`|If less than|
|`ifnz`|If not zero|
|`ifyz`|If yes zero|
|`peek`|Non-destructive read|
|`noop`|Do nothing/skip|
|`rand`|Random data from uninitialized memory|
|`bell`|Ring the bell or output sound|
|`eomf`|End of MOBS file. Terminate program|

### 1. `init` - Initialize

Overwrites a register with a 4 byte value, resets cursor to position 0.

**Syntax:**
```
init <register> <value>
init <register>
```

**Behavior:**
- Sets register to value (or 00000000 if no value given)
- Resets cursor to position 0
- **Destructive** - overwrites existing data

**Examples:**
```
init M DEADBEEF    ~ M = DEADBEEF, cursor at 0
init O             ~ O = 00000000, cursor at 0
init S 68656C6C    ~ S = "hell", cursor at nibble 8
```

### 2. `adds` - Addition

Adds a value to a register at cursor position, wraps on overflow.

**Syntax:**
```
adds <register> <value>
adds <register> <other_register>
adds <register>
```

**Behavior:**
- If cursor is at 0: normal addition
- If cursor is offset: adds starting from cursor position
- If only register specified: adds register to itself (doubles it)
- Carries propagate left and wrap around to nibble 7 if they overflow past nibble 0
- Wraps on overflow
- **Destructive** - modifies register

**Examples:**
```
init M 00000005
adds M 00000003    ~ M = 00000008
eomf

init M 00000002
adds M            ~ M = 00000004 (M + M)
eomf

init M 00000001
init O 00000002
adds O to M        ~ M = 00000003 (M + O)
eomf

init M FFFFFFFF
adds M 00000001    ~ M = 00000000 (overflow)
eomf

init M 12345678
jump M 00000004    ~ cursor at nibble 4
adds M 00001111    ~ adds starting from position 4
eomf               ~ M = 12346789

noop               ~ Multiple carry propagation:
init M 0000FFFF
jump M 00000004    ~ cursor at nibble 4 (first 'F')
adds M 00000001    ~ FFFF + 0001 = 10000
noop               ~ Carry propagates: FFFF→0000, carry continues left
eomf               ~ M = 00010000
```

### 3. `subs` - Subtraction

Subtracts a value from a register, wraps on underflow.

**Syntax:**
```
subs <register> <value>
subs <register> <other_register>
subs <register>
```

**Behavior:**
- If cursor is at 0: normal subtraction
- If cursor is offset: subtracts starting from cursor position
- If only register specified: subtracts register from itself (equals 0)
- Borrows propagate left and wrap around to nibble 7 if they underflow past nibble 0
- Wraps on underflow
- **Destructive** - modifies register

**Examples:**
```
init M 00000008
subs M 00000003    ~ M = 00000005
eomf

init M 00000005
subs M             ~ M = 00000000 (M - M)
eomf

init M 00000001
init O 00000002
subs O to M        ~ M = 00000001 (O - M)
eomf

init M 00000000
subs M 00000001    ~ M = FFFFFFFF (underflow)
eomf

init M 12345678
jump M 00000004    ~ cursor at nibble 4
subs M 00001111    ~ subtracts 1111 from 5678 (starting at position 4)
eomf               ~ M = 12344567 (5678 - 1111 = 4567)

noop               ~ Multiple borrow propagation:
init M 00010000
jump M 00000004    ~ cursor at nibble 4 (first '0')
subs M 00000001    ~ 0000 - 0001 needs borrows
noop               ~ Borrows cascade left: 0001→0000, nibbles 4-7→FFFF
eomf               ~ M = 0000FFFF

noop               ~ Borrow cascade example:
init M 10000000
jump M 00000004    ~ cursor at nibble 4
subs M 00000001    ~ 0000 - 0001
noop               ~ Borrows from left: 1→0, nibbles 4-7→FFFF
eomf               ~ M = 0FFFFFFF
```

### 4. `move` - Destructive Transfer

Copies data from source to destination, then nulls the source.

**Syntax:**
```
move <source> to <destination>
```

**Behavior:**
- Copies source register to destination
- Sets source to 00000000
- Resets both cursors to 0
- **Destructive** - nulls source

**Examples:**
```
init M CAFEBABE
move M to O        ~ O = CAFEBABE, M = 00000000
eomf

init M DEADBEEF
init O 12345678
move O to M        ~ M = 12345678, O = 00000000
eomf
```

### 5. `dupe` - Semi-Destructive Copy

Copies data from source to destination, source becomes same as destination.

**Syntax:**
```
dupe <source> to <destination>
```

**Behavior:**
- Copies source to destination
- Source remains equal to destination (both now identical)
- Resets both cursors to 0
- **Semi-destructive** - source is modified to match destination

**Examples:**
```
init M DEADBEEF
dupe M to O        ~ Both M and O = DEADBEEF
eomf

init M AAAAAAAA
dupe M to O
init M 00000000    ~ "Cleans up" after copy
eomf               ~ O = AAAAAAAA, M = 00000000
```

> [!NOTE]  
> To achieve a true non-destructive copy, use `dupe` followed by `init` to reset the source.

### 6. `jump` - Navigation

Skip lines (go down the source code) or<br>
Moves cursor position in a register (go right inside a register).

**Syntax:**
```
jump <lines>
jump <register> <nibbles>
```

**Behavior:**
- Without register: jumps through code lines
- With register: moves that register's cursor
- Wraps on overflow for both skipping ligns and moving the cursor
- Used for backward navigation via overflow

**Examples:**
```
noop
jump 00000001      ~ Jump forward 2 lines of code
noop
init S 4D4F4253    ~ jumps to here
```

```    
       You are here
       |
init M DEADBEEF
jump M 00000003    ~ M's cursor now at nibble 3
M = DEADBEEF
      |
      You are now here.
Move M to O
O = ADBEEFDE
```

```
init S 68656C6C
jump S 000000A0 ~ Set S's cursor to the start of line 1 at nibble 160 (line 2 on 80-char display)
init S 6F20776F
jump S 00000140 ~ Set S's cursor to the start of line 2 from line 1
init S 726C6421
eomf

===================================================================================
4 LINE 80 CHAR SCREEN VIEW OF S
===================================================================================
0: hell
1: o wo
2: rld!
3:
====================================================================================
```

**Code Navigation:**
- Forward: Use small values (values less then the total line count of the program)
- Backward: Jump forward past the end to wrap around, or use conditionals at the end to loop

The interpreter does not have an immutable counter of how many lines are in a program.<br>
All your doing when you jump is increasing the program counter, the count of how many
lines the interpreter has processed so far.<br>And like all things in MOBS-16, this counter can overflow too.<br>
You can't just "GOTO 75" because we don't have any concept of where a static address of "75" is in our program.<br>
However, We can jump forward 75 lines from wherever we currently are.<br>to get to a new location
weather that be line 76 or 276, it all depends on where you started from.<br><br>

This also means that if your program is only 10 lines long, and your at line 1, and you jump to line 100.<br>
You are going to wrap around the whole program 90 times befor ending up at line 10 again.

> [!NOTE]
> With the circular execution model, you rarely need backward jumps.<br>
> The program automatically wraps to the beginning.


**Register Navigation:**
- M, O, B: cursor wraps at 8 nibbles
- S: cursor wraps based on hardware configuration

Example with a line of tape:

```
Was here
|
DEADBEEF00000000
    |
    Now here (after jump M 00000004)
```

Now `move M to O` gives O = `BEEF0000` (reads 8 nibbles starting from cursor).

### 7-11. Conditionals - `ifeq`, `ifgt`, `iflt`, `ifnz`, `ifyz`

Execute any single opcode if condition is true.

**Syntax:**
```
ifeq <reg1> <reg2> <opcode> [params]    ~ if equal to
ifeq <reg> <value> <opcode> [params]

ifgt <reg1> <reg2> <opcode> [params]    ~ if greater than
ifgt <reg> <value> <opcode> [params]

iflt <reg1> <reg2> <opcode> [params]    ~ if less than
iflt <reg> <value> <opcode> [params]

ifnz <reg> <opcode> [params]            ~ if not zero
ifyz <reg> <opcode> [params]            ~ if yes zero
```

**Behavior:**
- Compares values and executes the specified opcode if condition is true
- If condition is false, execution continues to next line
- Can execute **any single opcode** 
  (except nested conditionals. You will just get stuck in a death loop of if true then if true then if true.. befor anything is executed)
- Most common use is with `jump`, but works with all opcodes

**Examples with jump (traditional):**
```
~ Loop until M reaches 10
init M 00000000
adds M 00000001
iflt M 0000000A jump 00000000      ~ if M < 10, jump back 1 line
eomf

~ Conditional branch
ifeq M O jump 00000005             ~ if M == O, skip ahead 6 lines
ifgt M 00000064 jump 00000003      ~ if M > 100, jump forward 4 lines
```

**Examples with other opcodes:**
```
~ Conditional exit
ifyz M eomf                        ~ if M is zero, terminate program
ifgt M 00000064 eomf               ~ if M > 100, exit

~ Conditional bell
ifnz M bell                        ~ if M not zero, ring bell
ifeq M 0000002A bell M             ~ if M == 42, ring bell with M's data

~ Conditional initialization
ifyz O init S 5A45524F             ~ if O is zero, display "ZERO"
ifgt M 00000005 init B DEADBEEF    ~ if M > 5, set B to DEADBEEF

~ Conditional arithmetic
ifeq M O adds B 00000001           ~ if M == O, increment B
iflt M 00000064 subs M 00000001    ~ if M < 100, decrement M

~ Conditional data movement
ifnz M move M to S                 ~ if M not zero, output it to S
ifeq O 00000000 dupe M to O        ~ if O is zero, copy M to O

~ Conditional randomization
ifyz B rand B                      ~ if B is zero, randomize it

~ Chain conditionals (not nested, just sequential)
ifgt M 0000000A init S 42494721    ~ if M > 10, S = "BIG!"
iflt M 00000005 init S 736D616C6C  ~ if M < 5, S = "small"
ifeq M 00000007 bell               ~ if M == 7, ring bell

~ Infinite evaluation loop
ifyz M ifnz O bell    ~ BUG: This is broken!
~ Interpreter: "if M is zero, then check if O is not zero,
~  but wait, I need to check if M is zero first,
~  but to do that I need to check if O is not zero,
~  but wait, if..."
This will not cause a crash or an error, your program will just get stuck on this line, forever.
```

**Important Notes:**
- **Cannot nest conditionals** - No `ifyz M ifnz O jump 5`
- **Cannot use compound conditions** - No `ifyz M or ifyz O jump 5`
- **Can only execute ONE opcode** per conditional
- If you need complex logic, use multiple sequential conditionals

**Advanced usage:**
```
~ Conditional loop with exit
init M 00000000
adds M 00000001               ~ Increment M
iflt M 0000000A jump FFFFFFFE ~ Loop if M < 10
ifyz O eomf                   ~ Exit if O is zero (separate check)

~ Conditional output without jump
init M CAFEBABE
ifnz M move M to S ~ Output M only if non-zero, continue either way
bell               ~ This always executes
```

### 12. `peek` - Non-Destructive Read

Reads from a register's cursor position without modifying source or cursors.

**Syntax:**
```
peek <source> to <destination>
```

**Behavior:**
- Reads starting from source's cursor position
- Stores at destination's current cursor position
- Wraps around source to fill destination completely and vice versa for a total of 4 bytes at a time.
- **Non-destructive** - does NOT modify source or move any cursors
- Your ONLY truly non-destructive operation

**Examples:**
```
init M DEADBEEF
jump M 00000002    ~ cursor at nibble 2
peek M to O        ~ reads: AD, BE, EF, DE (wraps)
                   ~ O = ADBEEFDE, M unchanged, M cursor still at 2

init S 1234567890ABCDEF
jump S 0000000A    ~ cursor at nibble 10 ('A')
peek S to M        ~ reads: AB, CD, EF, 12 (wraps to get 8 nibbles)
                   ~ M = ABCDEF12
                   ~ S cursor still at 10
```

**Pro tip:** Use `peek` to treat S like RAM (or LAM - Linear Access Memory). Peek from S to M/O/B, manipulate the data, then append back to S.

### 13. `noop` - No Operation

Does nothing.

**Syntax:**
```
noop
```

**Behavior:**
- Executes but performs no action
- Useful for timing, padding, or as jump targets
- Useful to anchor comments, as all comments need to be attached to *something*

**Examples:**
```
noop ~ Does nothing
noop ~ Still does nothing
noop ~ Yep, you guessed it. Still does nothing
noop ~ However this program takes 4 "ticks" of your clock to run, even know it does nothing.
```

### 14. `rand` - Random Number Generation

Fills a register with random nibbles from uninitialized memory. 4 bytes at a time.

**Syntax:**
```
rand <register>
```

**Behavior:**
- Fills register with random/garbage data
- Pulls from uninitialized memory (implementation dependent)
- Appears Resets cursor to 0, However on M, O and B your have just overflowed back to the start.
- S registor cursor is now 8 nubbles over.
- **Destructive** - overwrites register
- Results are unpredictable and may vary between runs

**Examples:**
```
rand M             ~ M = 5EA9029A (why? who knows?)
rand O             ~ O = B7F3D581 (different garbage)
rand B             ~ B = D8A90E76 (where is this data even coming from?)
rand S             ~ S = 8BA93E51???????????????? (The rest of S is still uninsilized)
```

> [!NOTE]  
> M, O, B start uninitialized anyway, so on program boot they already contain random garbage!<br>
> You can use this as free randomness without calling `rand` at all. Just don't `init` them first.

> [!WARNING]  
> ```
> rand S
> ```
> This simple one line program will keep filling the S register until the hardware says stop!<br>
> as you never said `eomf`
> If the hardware never tells us to stop, we never stop!<br>
> If your S register is 9 miles of tape, we will fill it, 4 bytes at a time. Until you run out of tape, or the heat death of the universe.<br>
> Whichever comes first.

Garbage collection? Safe memory initialization? Nah, ima just use it as is, nom nom nom. It's just hex data after all.

### 15. `bell` - Bell/Speaker Output

Rings a bell or outputs sound. Can be fed data from any register.

**Syntax:**
```
bell
bell <register>
```

**Behavior:**
- Without argument: Simple single bell ring or one second beep
- With register: Feeds register's data to bell or speaker starting from cursor position
- How data is interpreted as sound is hardware-dependent
- Feeds entire register from cursor position: 4 bytes for M/O/B, infinite for S

**Examples:**
```
bell               ~ Simple beep
bell M             ~ Feed M's hex to bell (musical notes?)
bell S             ~ Feed entire stack to bell (NOISE!)
```

**Creative Uses:**
```
~ Musical sequence
init M 00000440    ~ Musical note at 440Hz frequency for eg.
bell M

~ Cursor-based sound
init M 12345678
jump M 00000004    ~ Cursor at nibble 4
bell M             ~ Outputs 45678123 to bell

~ Absolute chaos
rand S
bell S             ~ MAXIMUM NOISE
```

### 16. `eomf` - End of File

Terminates program execution and cleans up state.

**Syntax:**
```
eomf
```

**Behavior:**
- **Total cleanup** - Resets M, O, B to 00000000
- **Resets all cursors** to position 0 (including S)
- **Leaves S register intact** - S data persists (hardware-dependent display)
- Immediately terminates the program
- Can be placed anywhere in the source code
- Can be executed directly or via conditional (e.g., `ifyz M eomf`)
- Can be jumped to as a line target
- The only way to stop a MOBS-16 program (besides external termination)
- Multiple `eomf` statements can exist in a program


Even know MOBS-16 is based on chaos, we should leave the park cleaner then we found it when the party is over. Always clean up our mess when we're done.<br>
`eomf` is a total nuke that resets everything except S (which contains your output/results). Returns everything to a clean state on exit.<br>
An interesting side affect of this is that if you exit a MOBS-16 program with `eomg` but then Immediately run the program again, This time when the program loads it will load into a clean state (M, O, B will all to 00000000).<br>
The "free random number generation" only works on the first run, or after waiting for other programs to dirty that memory again.<br>
Because we cleaned up the park before we left the last time we where here, there's no garbage left to use for the next run.

This means:
- **First run:** M, O, B contain garbage (free randomness!)
- **Second run (immediate):** M, O, B contain 00000000 (cleaned by previous `eomf`)
- **Nth run (after other programs):** M, O, B contain garbage again (someone else dirtied the park and the memory. How rude.)

**Examples:**

**Basic termination:**
```
init S 48656C6C    ~ "Hell"
eomf               ~ Program ends, M/O/B reset to 00000000, S shows "Hell"
```

**Conditional exit (via conditional execution):**
```
init M 00000005
ifgt M 00000003 eomf    ~ If M > 3, terminate immediately
init S 4E6F             ~ "No" - only prints if M <= 3
eomf                    ~ Clean exit: M/O/B = 00000000
```

**Conditional exit (via jump to eomf line):**
```
init M 00000005
ifgt M 00000003 jump 00000001    ~ If M > 3, jump to eomf
init S 4E6F         ~ "No" - only prints if M <= 3
eomf                ~ Clean exit: M/O/B = 00000000
```

**eomf in the middle:**
```
init M 00000000
adds M 00000001      ~ Line 1
iflt M 0000000A jump 00000002    ~ Skip eomf if M < 10
eomf                 ~ Line 3: Exit when M >= 10, everything reset
~ Execution wraps back to line 1 and continues looping
```

**Multiple conditional exits:**
```
init M 00000005
ifeq M 00000000 eomf    ~ Exit if zero
ifgt M 00000064 eomf    ~ Exit if > 100
~ Do work here
adds M 00000001
~ Program wraps to beginning automatically
```

**State preservation demonstration:**
```
init S 52657375      ~ "Resu"
init M DEADBEEF      ~ Some computation
adds M 12345678      ~ More computation
move M to S          ~ Store result in S
eomf                 ~ Exit: M/O/B cleared, but S still shows "Resu" + result
```

**Note:** Without `eomf`, your program will loop infinitely. This is by design!

## Comments

Comments begin with `~` and continue to end of line.

**Examples:**
```
init M DEADBEEF    ~ This is a comment
adds M 00000001    ~ Comments can explain code
noop               ~ You must anchor comments to code
bell               ~ Ring that bell!
eomf               ~ All done!
```

You must attach or anchor your comments to a line of code. No floating blocks of comments allowed.

## The S Register in Detail

The S (Screen/Stack) register is special and hardware-dependent.<br>
Think of the S register as an infinitely large <i>thing</i> we just so happen to be able to see with our human eyes, because it exists in the real world.<br>
Where as the M, O and B registers are inside the computer and can't be touched by humans directly.<br>
The S register can be touched by humans directly.<br>
You can turn off the display, you can eat the tape, you can burn down your house. All real world things affect the S register.<br>
The S register can be anything that stores and on and off state in a line. said line can be read back into M, O or B, 4 bytes at a time.<br>
What is 4 bytes? well thats up to the storage medium.<br>
A 16x16 core memory grid?<br>
32 mine craft torches?<br>
12 inchs of paper printed at 400 DPI?<br>
32 base pairs of DNA (ATCG maps to 2 bits each)?<br>
32 Dominoes. Standing or fallen, eaten or un-eaten.<br>
16 Dominoes. Each number on a Domino can be a hex nibble<br>
 a double-six set containing 28, we only need 16 for 0 to F, one nibble at a time.<br>
16, 16 sided dice. Thats kinda cheating, but you get the idea.<br>

The S register can be <i>anything</i> as long as your interpreter can read it, and do something with it to get it into hex data, 4 bytes at a time.

### Display Modes

S can be connected to various hardware like:

**Screen/Terminal:**
- Hardware defines character width (e.g., 80 chars = 160 nibbles per line)
- Jumping beyond line width moves to next line
- Example: 80×25 terminal = 2000 chars = 4000 nibbles total.<br>
2kb is a lot of memory to use. more so when you can only use it at 4bytes at a time.

**Tape:**
- Non-volatile storage
- Can be arbitrarily long
- Your "RAM" persists when powered off

**Paper Punch:**
- Physical output
- One-way (can only write once, must wait for data to come back before reading)
- Permanent record

**Other:**
- Any FIFO storage device
- Network socket (boring)
- File system (too vanilla)
- Bad Apple video (yes)
- Quantum entanglement (fun but not recommended)

### Line Addressing

Hardware determines where lines begin. Because of MOBS-16's circular nature, you can conceptually start and end your program anywhere, but from a source code point of view, you start at the top of the page and end at the `eomf` opcode (or loop forever if no `eomf`).

### Unlimited Capacity

S has no theoretical limit. To address positions beyond FFFFFFFF:

```
init S
jump S FFFFFFFF    ~ At position 4,294,967,295
jump S 00000001    ~ At position 4,294,967,296
~ Continue as needed for larger positions
```

Since M, O, B can only hold values up to FFFFFFFF, you must chain jumps to reach arbitrarily large positions.

## Programming Examples

### Hello World

**Simple version:**
```
init S 68656C6C    ~ "hell"
adds S 6F20776F    ~ "o wo"
adds S 726C6421    ~ "rld!"
eomf               ~ Now "hello world!" is in the S register, we can exit.
```

### Counter (0 to 9)

```
init M 00000000    ~ Counter starts at 0
dupe M to O
move O to S        ~ Display current count
adds M 00000001    ~ Increment
iflt M 0000000A jump 4 ~ Loop back 3 lines if M < 10
eomf               ~ Stop when counter reaches 10
```

### Multiplication (3 × 4 = 12)

```
init M 00000003      ~ Multiplicand (3)
init O 00000004      ~ Multiplier (4)
init B 00000000      ~ Result accumulator
adds B M             ~ Add M to result
subs O 00000001      ~ Decrement counter
ifnz O jump 00000006 ~ Loop back 2 lines if O != 0
move B to S          ~ Display result (12)
eomf
```

### Division (20 ÷ 4 = 5)

```
init M 00000014        ~ Dividend (20)
init O 00000004        ~ Divisor (4)
init B 00000000        ~ Quotient
subs M O               ~ Subtract divisor from dividend
adds B 00000001        ~ Increment quotient
ifgt M O jump 00000008 ~ Loop back 2 lines if M > O
iflt M O jump 00000001 ~ Skip next line if M < O
adds B 00000001        ~ Add 1 more if M == O
move B to S            ~ Display result (5)
eomf
```

### Fibonacci Sequence (first 10 numbers)

```
init M 00000000       ~ F(0) = 0
init O 00000001       ~ F(1) = 1
init B 0000000A       ~ Counter (10 iterations)
move M to S           ~ Output F(0)
move O to S           ~ Output F(1)
peek M to M           ~ Save M (non-destructive prep)
adds M O              ~ M = M + O
move M to S           ~ Output next Fibonacci
dupe O to M           ~ Swap values
dupe M to O           ~ Complete the swap
subs B 00000001       ~ Decrement counter
ifnz B jump 000000005 ~ Loop back 7 lines if counter != 0
eomf
```

### Infinite Counter (no eomf)

```
init M 00000000 ~ Counter
adds M 00000001 ~ Increment
dupe M to O
move O to S     ~ Display current count
noop            ~ No eomf - program wraps to beginning and continues forever!
```

### Conditional String Output

```
init M 00000005
ifgt M 00000003 jump 00000002    ~ If M > 3, skip "small"
init S 736D616C6C                ~ "small"
jump 00000001                    ~ Skip "large"
init S 6C61726765                ~ "large"
eomf
```

**Using conditional execution (no jumps):**
```
init M 00000005
iflt M 00000004 init S 736D616C6C    ~ if M < 4, S = "small"
ifgt M 00000003 init S 6C61726765    ~ if M > 3, S = "large"
eomf                                 ~ Both conditions can't be true
```

### Random Number Generator with Bell

```
rand M         ~ Get random number (or just use uninitialized M!)
bell M         ~ Ring bell with random tone
jump 00000001  ~ Loop back 1 line (or remove for auto-loop)
noop           ~ Without eomf, generates infinite random bell tones!
```

**Using boot garbage:**
```
~ M, O, B might be random on boot - unless previous program cleaned up!
bell M             ~ Ring bell with whatever was in memory
bell O             ~ Different random tone (or silence if zeros)
bell B             ~ Another random tone (or silence if zeros)
eomf               ~ Clean up for next run - now M, O, B = 00000000

~ Second run of this program will ring silent bells!
~ Third run after another program may have garbage again
```

**Exploiting the cleanup cycle:**
```
~ Program 1: Intentionally leave garbage
init M CAFEBABE
init O DEADBEEF
init B 12345678
noop            ~ No eomf - runs forever until externally terminated
noop            ~ When killed, M, O, B stay dirty (no cleanup happened)

~ Program 2 (run immediately after): Gets free random numbers!
peek M to S    ~ Gets CAFEBABE (if Program 1 was killed before changing M)
peek O to S    ~ Gets DEADBEEF
peek B to S    ~ Gets 12345678
eomf           ~ Clean up - now M, O, B = 00000000

~ Program 2 (run again): Gets zeros
peek M to S    ~ Gets 00000000 (cleaned by previous eomf)
peek O to S    ~ Gets 00000000
peek B to S    ~ Gets 00000000
eomf

~ Program 3 (no eomf): Runs forever, never cleans up
init M BAADF00D
~ Loops forever, externally terminated eventually
~ M stays BAADF00D (or whatever it became) for next program
```

### Prime Number Checker

```
init M 00000011    ~ Number to test (17)
init O 00000002    ~ Divisor
init B 00000000    ~ Temp for division
dupe M to B        ~ Copy M to B
subs B O           ~ B = M - O
ifyz B jump 00000004    ~ If divisible, not prime
adds O 00000001    ~ Next divisor
iflt O M jump 00000008    ~ Keep testing if O < M
init S 7072696D65    ~ "prime"
eomf
init S 6E6F7421      ~ "not!"
eomf
```

**Using conditional execution:**
```
init M 00000011    ~ Number to test (17)
init O 00000002    ~ Divisor
dupe M to B        ~ Copy M to B
subs B O           ~ B = M - O
ifyz B init S 6E6F7421    ~ If divisible, S = "not!" and continue
ifyz B eomf               ~ Exit if not prime
adds O 00000001           ~ Next divisor
iflt O M jump 00000008    ~ Keep testing
init S 7072696D65         ~ "prime"
eomf
```

### Using Circular Execution (Blinking Display)

```
init S 4F4E    ~ "ON"
bell           ~ Beep
init S         ~ Clear S
bell           ~ Beep again
noop           ~ No eomf - loops forever creating blinking effect!
```

## Implementation Notes

### Minimal Interpreter Requirements

A MOBS-16 interpreter must:
1. Allocate memory for M, O, B registers but **leave them uninitialized** (containing garbage)
2. Initialize all cursors to position 0
3. Maintain 4 registers (M, O, B, S) with cursors during execution
4. Parse 16 opcodes
5. Handle hex arithmetic with wrapping
6. Implement circular execution (wrap to start at EOF unless `eomf` encountered)
7. Support comments (~) only attached to code lines
8. Implement hardware-specific S register behavior
9. On `eomf`: reset M, O, B to 00000000, reset all cursors to 0, leave S intact, terminate cleanly

MOB-16 op codes are very unpredictable whether they will or will not reset the cursor.<br>
There is no rhyme or reason, just whatever I felt like at the time of making the op code.<br>
So you as the system need to keep track of the cursor, if not your op codes are going to act even more strangely then they already do.
   
### Memory Management

More memory usage, we don't manage anything, and should be able to use any trash left for us as valid hex data.

- M, O, B: Fixed 8 nibbles (4 bytes) each
- S: Dynamic, grows as needed, minimum 24 nibbles (12 bytes) to fit all base registers into S at once.
- Cursors: Separate position tracking for each register. But you only have to count up for this one. Remember we can overflow around to the start.
- Program counter: Same as above but we count down the page in one counter. Remember we can overflow back to the top.

### Error Handling

lol no.<br>

MOBS-16 embraces chaos. There are no runtime errors, only syntax error:
- Overflow/underflow wraps
- Invalid jumps wrap
- Out-of-bounds wraps
- Uninitialized memory is valid (for `rand`)
- Programs without `eomf` loop infinitely (intentional!)

The only errors are **syntax errors** (malformed opcodes, floating comments, invalid register names).

And even then, the system will still do its darndest to execute whatever you gave it, valid or not.

Try and put things in an X registor? Well MOBS-16 will put that data in a registor somewhere, but who knows where as X does not excist.

### Performance Considerations

- Infinite loops are common and intentional
- Large S register may consume significant real world time to use. (find where you are up to on the tape, spin up the drumb, etc.)
- Multiplication/division require loops (no native ops, no shortcuts.)
- Cursor tracking adds overhead
- Circular execution adds minimal overhead (just a program counter wrap)
- **Timing is hardware-dependent** - loop execution speed varies by implementation
- No guaranteed execution speed or timing precision

> [!IMPORTANT]
> MOBS-16 does not specify execution speed.<br>
> A timing loop that works on one system may be completely different on another.<br>
> You can go as fast or as slow as you want. You are still processing the program one line at a time (including `noop`s).<br>
> How many lines you can churn though in a sec, or a millisecond is totally up to you. Lines per day, lines per full moon. Its all relative.<br>
> Anything can be your clock, all it has to do is advance the program counter once per n. If you want to use the half-life of Cesium-137. Go for it.

Hardware determines:
- How fast the program counter advances
- How quickly loops execute
- Display refresh rates
- S register write/read speeds (how many 4 byte chunks can you eat in a sec?)
- S registor size. Both in bytes and in physical space in the real world.

### Optimization Hints

1. **Use circular execution** - Remove unnecessary loop-back jumps
2. **Minimize large jumps** - Small forward jumps are efficient
3. **Reuse registers** - You only have 3 fast data registers
5. **S registor is your friend** - You can peek from S and append from M, O and B back to S as many times as you want.
4. **Place eomf strategically** - Can have multiple exit points
5. **Chain operations** - Operations flow naturally with cursors

## Turing Completeness

MOBS-16 is Turing complete?

✅ **Arbitrary memory** - S register has unlimited capacity  
✅ **Conditional branching** - Five conditional opcodes  
✅ **Loops** - Built-in circular execution + conditionals + jump  
✅ **Arithmetic** - Addition and subtraction (multiplication/division via loops)  
✅ **Termination** - `eomf` provides halting capability

Any computable function can theoretically be implemented in MOBS-16, though it may require creativity, patience and a lot of coffee.

## Philosophy & Design Goals

MOBS-16 embodies several deliberate design choices:

**Destructive Operations:**
- Most ops modify or destroy source data
- Forces careful state management
- Only `peek` is truly non-destructive

**Circular Execution:**
- Programs loop by default
- Cursors loop by default
- Embraces the "overflow" concept at the program level
- Must explicitly terminate with `eomf`
- Reduces boilerplate loop-back code

**No Multiply/Divide:**
- Must implement via addition/subtraction loops
- "You can only go up or down, you can't skip around"
- Embraces computational fundamentals

**Overflow as Feature:**
- Wrapping is how you navigate backward
- No error conditions, only different states
- Embraces undefined behavior

**Hardware Agnostic:**
- S register adapts to any FIFO device
- Display width determines line breaks
- Non-volatile storage possible and encouraged. Get creative with the S register. It can be <i>anything</i>.

**Minimal Instruction Set:**
- Exactly 16 opcodes (MOBS-16)
- Each exactly 4 characters
- Simple enough "to teach a monkey with a calculator"
  

**Philosophy:** "Buffer overflows are a feature, not a bug. Undefined behavior is a feature, not a bug"<br>
"Your S registor can be <i>anything</i> that stores an on or off, 1 or 0 value"<br><br>
**Inspiration:** Assembly language, Commodore 64 PEEK/POKE, circular buffers, drumb memory, undefined behavior

## ASCII Reference Table
Back of the book doodles...<br>
For convenience when programming in MOBS-16.

However, ASCII rendering for S register is dependent on hardware (char map, char chip, dictionary, bitmap folder, etc.).

```
Common Characters:
Space = 20    !  = 21    "  = 22    #  = 23
$  = 24       %  = 25    &  = 26    '  = 27
(  = 28       )  = 29    *  = 2A    +  = 2B
,  = 2C       -  = 2D    .  = 2E    /  = 2F
0  = 30       1  = 31    2  = 32    3  = 33
4  = 34       5  = 35    6  = 36    7  = 37
8  = 38       9  = 39    :  = 3A    ;  = 3B
<  = 3C       =  = 3D    >  = 3E    ?  = 3F
@  = 40       A  = 41    B  = 42    C  = 43
D  = 44       E  = 45    F  = 46    G  = 47
H  = 48       I  = 49    J  = 4A    K  = 4B
L  = 4C       M  = 4D    N  = 4E    O  = 4F
P  = 50       Q  = 51    R  = 52    S  = 53
T  = 54       U  = 55    V  = 56    W  = 57
X  = 58       Y  = 59    Z  = 5A    [  = 5B
\  = 5C       ]  = 5D    ^  = 5E    _  = 5F
`  = 60       a  = 61    b  = 62    c  = 63
d  = 64       e  = 65    f  = 66    g  = 67
h  = 68       i  = 69    j  = 6A    k  = 6B
l  = 6C       m  = 6D    n  = 6E    o  = 6F
p  = 70       q  = 71    r  = 72    s  = 73
t  = 74       u  = 75    v  = 76    w  = 77
x  = 78       y  = 79    z  = 7A    {  = 7B
|  = 7C       }  = 7D    ~  = 7E
```

## Conclusion

MOBS-16 forces programmers to think carefully about state, embrace chaos, implement complex operations from simple primitives, and decide when (or if) their programs should ever stop.

The circular execution model means every MOBS-16 program is fundamentally a loop. The party only stops when you say `eomf`.

It's intentionally difficult yet easy to grasp the basics, gloriously destructive, beautifully circular, and surprisingly capable.

Welcome to MOBS-16. May your overflows be intentional, your loops be infinite, and your bells be loud.

---

**Questions? Found a bug? Want to contribute?**

Remember: In MOBS-16, all bugs are features. Especially buffer overflows and infinite loops.
