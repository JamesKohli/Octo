---
title: Octo Manual
---

Octo
====

Octo is a simple high-level assembler for the Chip8 virtual machine, complete with an environment for testing programs. While a program is running, you can press escape to return to the editor. The chip8 keypad is represented on your keyboard as follows:

	Chip8 Key   Keyboard
	---------   ---------
	 1 2 3 C     1 2 3 4
	 4 5 6 D     q w e r
	 7 8 9 E     a s d f
	 A 0 B F     z x c v

Octo syntax is in some ways inspired by Forth- a series of whitespace-delimited tokens. Labels are defined with `:` followed by a name, and simply using a name by itself will perform a call. `;` terminates subroutines with a return. `#` indicates a single-line comment. Numbers can use `0x` or `0b` prefixes to indicate hexadecimal or binary encodings, respectively. Whenever numbers are encountered outside a statement they will be compiled as literal bytes. Names must always be defined before they can be used- programs are written in "reading" order. When forward references are absolutely necessary, the `:proto` statement followed by a name defines that name as a forward prototype, which can then be used as a target for jumps, calls and initializing i. Prototypes must eventually be resolved with a `:` label definition. An entrypoint named `main` must be defined.

Numeric constants can be defined with `:const` followed by a name and then a value, which may be a number, another constant or a (non forward-declared) label. Registers may be given named aliases with `:alias` followed by a name and then a register. The `i` register may not be given an alias, but v registers can be given as many aliases as desired. Here are some examples of constants and aliases:

	:alias x v0
	:alias CARRY_FLAG vF
	:const iterations 16
	:const sprite-height 9

Chip8 has 16 general-purpose 8-bit registers named `v0` to `vF`. `vF` is the "flag" register", and some operations will modify it as a side effect. `i` is the memory index register and is used when reading and writing memory via `load`, `save` and `bcd`, and also provides the address of the graphics data drawn by `sprite`. Sprites are drawn by xoring their pixels with the contents of the screen. Drawing a sprite sets `vF` to 0, or to 1 if drawing the sprite toggles any pixels which were previously on.

In the following descriptions, `vx` and `vy` refer to some register name (v0-vF), `l` refers to a (forth-style) identifier and `n` refers to some number.

Statements
----------

- `return`          return from the current subroutine. (alias for ;)
- `clear`           clear the screen.
- `bcd vx`          decode vx into BCD at I, I+1, I+2.
- `save vx`         save registers v0-vx to I.
- `load vy`         load registers v0-vx from I.
- `sprite vx vy n`  draw a sprite at x/y position, n rows tall.
- `jump n`          jump to address.
- `jump0 n`         jump to address n + v0.

Assignments
-----------

The various chip8 copy/fetch/arithmetic opcodes have been abstracted to mostly fit into a consistent `<dest-reg> <operator> <source>` format. For some instructions, `<source>` can have several forms.

- `delay := vx`    set delay timer to register value.
- `buzzer := vx`   set sound timer to register value.
- `i  := n`        set I to constant.
- `i  := hex vx`   set I to hex char corresponding to register value.
- `i  += vx`       increment I by a register value.
- `vx := vy`       copy register to register.
- `vx := n`        set register to constant.
- `vx := random n` set register to random number AND n.
- `vx := delay`    set register to delay timer.
- `vx := key`      block for a keypress and then store code in register.
- `vx += n`        add constant to register.
- `vx += vy`       add register to register. (set vF to 1 if result overflows, else 0)
- `vx -= vy`       subtract y from x, store in x (set vF to 1 if result underflows, else 0)
- `vx =- vy`       subtract x from y, store in x (set vF to 1 if result underflows, else 0)
- `vx |= vy`       bitwise OR register with register. 
- `vx &= vy`       bitwise AND register with register.
- `vx ^= vy`       bitwise XOR register with register.
- `vx >>= vy`      shift vy right by 1 and store result in vx. (set vF to LSB of vy)
- `vx <<= vy`      shift vy left by 1 and store result in vx. (set vF to MSB of vy)

Control Flow
------------
The Chip8 conditional opcodes are all conditional skips, so Octo control structures have been designed to map cleanly to this approach. The following conditional expressions can be used with `if...then` or `while`:

- `vx == n`
- `vx != n`
- `vx == vy`
- `vx != vy`
- `vx key` (true if the key indicated by vx is pressed)
- `vx -key` (true if the key indicated by vx is not pressed)

`if...then` conditionally executes a single statement. For example,

	if v0 != 5 then v1 += 2

`loop...again` is an unconditional infinite loop. `loop` marks the address of the start of the loop and produces no code, while `again` compiles a jump instruction based on the address provided by `loop`. Since `again` is itself a statement, we can use an `if...then` at the end of a loop to skip over the backwards jump and efficiently break out of the loop. The following loop will execute 5 times:

	v0 := 0
	loop
		# do something...
		v0 += 1
		if v0 != 5 then
	again

The other way to break out of a loop is `while`. `while` creates a conditional skip around a forward jump. These forward jumps are resolved by `again` to point to the address immediately outside their loop. `while` will thus exit the current loop if its condition is not true. You can have as many `while` statements in a loop as you want. Here is an example of `while` which is similar to the previous except for when the condition is checked.

	v0 := 0
	loop
		v0 += 1
		while v0 != 5
		# do something...
	again
