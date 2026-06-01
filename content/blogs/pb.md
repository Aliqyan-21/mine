---
title: surpassing the limit
date: 31-05-2026
template: blog
---

# Surpassing The Limit.

---

TLDR;
PureBasic’s demo version enforces an **800 loc** limit by checking both raw file size and post-macro expansion line counts.
I used radare2 to hunt down the "Gatekeeper" functions, surgically patched the size-check helper with a `mov eax, 1; ret` success signal,
and bypassed the secondary macro-expansion check **to turn the Demo into a fully unfettered compiler.**

---

So I was eating Subway sandwich and drinking Zero Sugar Coca Cola, while watching [**Tsoding**](https://youtube.com/@tsodingdaily?si=KFscvd5Uocv6q5mg)

I was watching the video about Basic programming language, he was trying out Pure Basic which is a commercial lang, u have to pay
to program in it, here is the vod -> [Basic still has potential](https://youtu.be/7_EV22SpDY4?si=MwykHMLpZvmMPeKN).

Then in the vod he was reading the download page on pure basic site. So there is a demo version u can download, which limits the LOC that u can write...

At this moment I had this intense curiosity, how are they preventing it? Is it straightforward silly, or some other trick that they are applying???
I gulped my remaining sandwich in one go, wiped my hands on my shorts, and fired up my terminal.

## Running Pure Basic

I downloaded the demo version and ran it.

...A GUI popped up :|

![ss of gui](/assets/ss1.png)

I closed it and saw that the compilers folder have some binaries:

```bash
╰─ ❯ find compilers -type f -executable
compilers/purebasic_gtk2
compilers/pbdebugger
compilers/purebasic_qt
compilers/pbcompilerc
compilers/purebasic
compilers/fasm
compilers/pbcompiler
```

> And the binary I was interested in was `pbcompiler`
because sounds like compiler for pure basic...(innocent face)

So I wrote a simple `hello world` program in pure basic:

```bash
╰─ ❯ cat hello.pb
PrintN("Hello, World!")
```

and tried to run it.

```bash
╰─ ❯ ./pbcompiler.cracked hello.pb
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
2 lines processed.
Creating and launching executable.

- Feel the ..PuRe.. Power -

Hello, World!
```

Wow!!

Then I did something like this:

```bash
╰─ ❯ for i in {0..800}; do
echo "PrintN(\"Hello, World\")" >> big_hello.pb;
done
```

And thus I had a file with 800 lines just printing 'Hello, World!'.

And now when I tried to compile it I got this:

```bash
╰─ ❯ ./pbcompiler big_hello.pb
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
Error: Source too big for the Free version.
```

OMG Aaarghh, what I will do now, *I wanted to print 800 lines of 'Hello, World!' it was very necessary to save the Earthh!!*

Well...to see how they are handling it, I fired up one of my favorite tool...my trusty `r2`.

## the tool: radare2

r2 for short.

It's a **free**, **open source** reverse engineering framework.

Think of it as a swiss army knife for binaries. You can _disassemble_, _debug_, _patch_, _search_, _analyze_, all from a command line interface.

##### You can grab your pc now and do this along with me if u like!

To open a binary in r2:

```bash
╰─ ❯ r2 pbcompiler
 -- Ah shit, here we go again.
[0x00402a40]>
```
> I like the random quotes radare throws when we open it though, this time it hit the nail. :)

This drops you into this interactive shell.  
The address in the prompt `[0x00402a40]` is just where r2's cursor currently is, like a seek position in the file.  
The first thing you always want to do is run analysis so r2 can figure out functions, strings, cross references etc:

```bash
[0x00402a40]> aaaa
INFO: Analyze all flags starting with sym. and entry0 (aa)
INFO: Analyze imports (af@@@i)
INFO: Analyze entrypoint (af@ entry0)
INFO: Analyze symbols (af@@@s)
INFO: Analyze all functions arguments/locals (afva@@@F)
INFO: Analyze function calls (aac)
INFO: Analyze len bytes of instructions for references (aar)
INFO: Finding and parsing C++ vtables (avrr)
INFO: Analyzing methods (af @@ method.*)
INFO: Recovering local variables (afva@@@F)
INFO: Type matching analysis for all functions (aaft)
INFO: Propagate noreturn information (aanr)
INFO: Scanning for strings constructed in code (/azs)
INFO: Finding function preludes (aap)
INFO: Enable anal.types.constraint for experimental type propagation
[0x00402a40]>
```

As u can see from the output, radare2 did some analysis of the binary and `aaaa` runs the most aggressive analysis pass.
It takes a few seconds but after this r2 knows a lot about the binary.

Now the compiler said, `Source too big for the Free version.`, so let's try searching that string in the binary first...

---

## step 1: find the error string

###### I increased the fan speed...

That string must exist somewhere in the binary. Let's find it.

r2 has a command `iz` which lists strings.

We pipe it through `grep` to find what we want:

```bash
[0x00402a40]> iz | grep -i "source too big"
1543 0x0005a528 0x0045a528 36  37   .rodata ascii   Source too big for the Free version.
```

Found it!

###### Heat was increasing, it's afternoon in India in summer...

Before we move on, Let me explain this output (non nerdy people can skip):
- `1543` is the index of this string in r2's string list
- `0x0005a528` is the offset in the file on disk
- `0x0045a528` is the virtual address in memory when the binary runs
- `36` is the length
- `.rodata` is the section it lives in memory.

So the string is at virtual address `0x0045a528`.
Now in our next step we should try to find what code in the binary actually uses this string.
In RE this is called finding cross references, or xrefs.
So le's do that...

---

## step 2: find who uses the string (and hit a wall)

In r2, the command `axt [addr]` finds data/code references to this address. So let's see what code references our error string:

```bash
[0x00402a40]> axt 0x0005a528
[0x00402a40]>
```

No output. Nothing.

###### I am thirsty, very thirsty, so I open my Paper Boat drink, and take a long sip.

Okay that's weird...
The string clearly exists, the compiler clearly prints it,
but r2 can't find anything in the code that references it.
This usually means the string isn't referenced directly. Something in between is going on.

Come on trusty r2...

---

## step 3: search for the string's address in memory

So...a great man once said, "just because code doesn't reference the string directly doesn't mean the string is unreachable."

Sometimes binaries store string addresses in a **pointer table**,
which is just an array where each entry is a pointer to a string.
Code then indexes into this table rather than hardcoding the string address everywhere.

If that's the case here, then somewhere in the binary there should be the bytes `28 a5 45 00`
(the string's address `0x0045a528` in little-endian format, which is how x86 stores numbers).

Let's search for those bytes:

```bash
[0x00402a40]> /x 28a54500 00000000
0x004724a0 hit3_0 28a5450000000000
[0x00402a40]>
```

`/x` in r2 searches for a hex pattern.
And as u can see from the output...
we found a match at `0x004724a0`. Let's go there and dump what's around it:

```bash
[0x00402a40]> s 0x004724a0
[0x004724a0]> pxq 128
0x004724a0  0x000000000045a528  0x000000000045a550   (.E.....P.E.....
0x004724b0  0x000000000045a590  0x000000000045a5b8   ..E.......E.....
0x004724c0  0x000000000045a5e0  0x000000000045a610   ..E.......E.....
0x004724d0  0x000000000045a640  0x0000000000455168   @.E.....hQE.....
0x004724e0  0x000000000045a670  0x000000000045a6b0   p.E.......E.....
0x004724f0  0x000000000045a6e8  0x0000000000455182   ..E......QE.....
0x00472500  0x000000000045a708  0x000000000045a748   ..E.....H.E.....
0x00472510  0x000000000045a788  0x000000000045a7b8   ..E.......E.....
[0x004724a0]>
```

`s` means seek, it moves r2's cursor to that address. `pxq` does word/qword hexdumps.

Look at the output. A perfectly neat table of 8-byte pointers, all pointing into the `.rodata` section.
And...would u look at that? Our string `0x0045a528` is the very first entry there, so yeah, it's there in pointer table,
basically an array of error messages. Code picks an error by index, grabs the pointer, done.

Let's check what the other entries say to confirm:

```bash
[0x004724a0]> ps @ 0x000000000045a590
A procedure must begin with a '('.

[0x004724a0]> ps @ 0x000000000045a5e0
A ')' is expected to close the procedure.

[0x004724a0]>
```

`ps` commands prints pascal/wide/zero-terminated strings at a given address.

And if we see the output, they all sound like compiler messages, so yeah we sort of found the *error message pointer table*.

So now we know the structure:

- Error table lives at `0x004724a0`
- "Source too big" is at index 0
- Code somewhere loads a string from this table and prints it

But we still need to find the code that does the actual limit check and triggers this error...

---

## step 4: find the error dispatcher

Since `axt` failed on the string directly, let's try it on the table itself:

```bash
[0x004724a0]> axt 0x004724a0
[0x004724a0]>
```

Also nothing...
The table address isn't referenced directly in code either.
The compiler is using RIP-relative addressing (a way of computing addresses relative to the current instruction pointer)
which r2 didn't resolve into a proper cross reference.

Different approach. Let's find the function that actually dispatches errors.
It probably loads from the table and copies the string somewhere.
We can search for that pattern by looking at who calls `strcpy` with a destination that looks like an error buffer.

![meme](/assets/later.png)

After some digging (a lot of dead ends,
some huge 2000+ line functions that turned out to be just code generators, or some data r2 confused to be code,
some garbage disassembly because r2 was reading a data table as code), we find `fcn.004159b0`:

```asm
[0x004159b0]> pdd
/* r2dec pseudo code output (r2 5.9.8) */
/* pbcompiler @ 0x4159b0 */
#include <stdint.h>

int64_t fcn_004159b0 (char ** dest) {
    rdi = dest;
    rax = 0x492698;
    edx = *(rax);
    if (edx == 0) {
        rdx = 0x4a1998;
        if (*(rdx) == 0) {
            goto label_0;
        }
    }
    return rax;
label_0:
    *(rax) = edi;
    rax = 0x4a3d20;
    rdi = (int64_t) edi;
    rdx = *(rax);
    rax = 0x493fb0;
    *(rax) = rdx;
    rax = 0x00472220;
    strcpy (0x493420, *((rax + rdi*8)));
    return rax;
}
```

There are some very powerful command in r2, two of them are:
- `pdf` -> disassembles the code into readable asm.
- `pdd` -> disassembles the code into more readable pseudo code.

**fcn_004159b0** is the dispatcher.
You pass it an index in `edi`, it looks up `error_table[index]` and copies that string to the error buffer.

```bash
[0x004159b0]> pxq 8 @ 0x00472220 + (0x50 * 8)
0x004724a0  0x000000000045a528                       (.E.....
[0x004159b0]>
```

`0x00472220 + (0x50 * 8) = 0x004724a0` which is exactly where "Source too big" lives.
So index `0x50` (80 in decimal) is our error.

So, any code that calls `fcn.004159b0` with `edi = 0x50` is triggering the "Source too big" error.
So...finally we found what we have to find next...the error dispatcher call with our string index.

---

## step 5: find the limit check (and the eureka moment)

Now we need to find where the compiler actually checks the line count and calls the error dispatcher with index `0x50`.

The natural thing to do is search for `mov edi, 0x50` followed by a call.
In hex that's the bytes `bf 50 00 00 00 e8`:

```bash
[0x004159b0]> /x bf50000000e8
0x0041bc04 hit4_0 bf50000000e8
0x0042ad7d hit4_1 bf50000000e8
[0x004159b0]>
```

Two hits.
After checking both, they're in functions that unconditionally trigger the error, not the ones doing the actual comparison.
Dead ends... happens...

###### Here I started drinking lots and lots of water...I finished my Paper Boat juice too.

So let's think about this differently. The check must look something like:

```c
if (line_count > LIMIT) {
    error(0x50); // source too big
}
```

In assembly that's a `cmp` instruction followed by a conditional jump.
The question is, what is `LIMIT`?

PureBasic advertises **800 lines** as the free version limit on it's page.
So why not let's just search for 800 simple...(`0x320` in hex). So I searched it as immediate value for `cmp` instruction.

```bash
[0x004159b0]> /a cmp dword [rax], 800
[0x004159b0]> /a cmp eax, 800
[0x004159b0]>
```

And I found....Nothing. Zero results. Hm.

Okay so either the limit isn't stored as a direct constant (possible), or the actual number isn't 800 (liars??).

###### And so I drank more water

I had to go pee, and while peeing, I realised, just like we estimate things,
pure basics people are also humans only, what if the limit is not 800, and it's maybe nearby it, *either 799,798...something like this?*

I washed my hands and ran towards my laptop, let's do this.

I will search for all numbers coming down from 800...

I was searching manually, 799, 798, ..., then I remembered I am a fucking programmer bro...

so I wrote this nice bash command:

```bash
╰─ ❯ for i in {750..800}; do
echo "index: $i"
r2 -q -c "/a cmp dword [rax], $i" pbcompiler 2> /dev/null
done
```

Runs r2 everytime for that command and puts $i, it searches for LIMIT=750 to LIMIT=800...

And... I got this!!!

```bash
...
index: 778
index: 779
index: 780
index: 781
index: 782
0x00422a62 hit0_0 81380e030000
index: 783
index: 784
index: 785
index: 786
index: 787
index: 788
...
```

There was a hit at `index = 782`!

###### Power of thinking while doing susu?

This is the moment.
The advertised limit is **800** but the binary actually checks **782**.
Now as I know this, finding the check is trivial.

So I just searched for 782 now in r2

```bash
[0x004159b0]> /a cmp dword [rax], 782
0x00422a62 hit14_0 81380e030000
```

And there is the hit. Now we can look at it to find the actual checking code finally...

---

## step 6: reading the actual check

```bash
[0x00422a62]> pd -2
│           0x00422a5a      4c8920         mov qword [rax], r12
│           0x00422a5d      488b442418     mov rax, qword [var_18h]
```

The command `pd -2` disassembles 2 instructions before the current position.
So right before the `cmp`, `rax` is being loaded from local var `var_18h`.
That could be the line counter pointer.

Then:
```asm
[0x00422a62]> pd 5 @ 0x00422a68
│       ┌─< 0x00422a68      0f8f3b070000   jg 0x4231a9
│       │   ; CODE XREF from fcn.00422930 @ 0x4231b9(x)
│       │   0x00422a6e      48c7c0c83a..   mov rax, 0x483ac8
│       │   0x00422a75      8b00           mov eax, dword [rax]
│       │   0x00422a77      85c0           test eax, eax
│      ┌──< 0x00422a79      0f8591060000   jne 0x423110
```

So the full picture at this spot:

```asm
mov rax, [var_18h]      ; load line counter pointer
cmp dword [rax], 0x30e  ; compare line count to 782
jg 0x4231a9             ; if greater, jump to error path
                        ; otherwise continue compiling normally
```

`jg` means "jump if greater".
If the line counter exceeds 782, we jump to `0x4231a9`. Boom Boom...

I had a smile now, this could be it, let's first see what is at `0x4231a9`.

```bash
[0x00422a62]> s 0x4231a9
[0x004231a9]> pdf
Do you want to print 790 lines? (y/N) n
```

790 lines...yeah...
That's a big function, wtf!

```bash
[0x004231a9]> pd 10
│       ╎   ; CODE XREF from fcn.00422930 @ 0x422a68(x)
│       ╎   0x004231a9      bf50000000     mov edi, 0x50               ; 'P' ; 80
│       ╎   0x004231ae      c700ffffffff   mov dword [rax], 0xffffffff ; [0xffffffff:4]=-1 ; -1
│       ╎   0x004231b4      e8f727ffff     call fcn.004159b0
│       └─< 0x004231b9      e9b0f8ffff     jmp 0x422a6e
│           ; CODE XREFS from fcn.00422930 @ 0x422efc(x), 0x422f12(x)
│           0x004231be      48c7c05cf9..   mov rax, 0x4af95c
│           0x004231c5      48c7c75cf9..   mov rdi, 0x4af95c
│           0x004231cc      488d15ed2e..   lea rdx, [0x004760c0]
│           0x004231d3      8b00           mov eax, dword [rax]
│           0x004231d5      89442410       mov dword [var_10h], eax
│           0x004231d9      83c001         add eax, 1
```

Great! see the output guyzes.

First two instructions tell us everything...

`mov edi, 0x50` then `call fcn.004159b0`.
That's the error dispatcher being called with index 80,
which we already know maps to "Source too big for the Free version."
After firing the error it jumps back to `0x422a6e` which is right after the `jg`,
so the compilation loop just continues erroring.

<img class="rattip-img" src="/assets/yes.gif" width=200 height=200 />

---

## step 7: the patch (didn't work)

Now what we can do is, not let the compiler check the limit at the address,
so we NOP the jg.

`jg` here is 6 bytes long so we need 6 NOP bytes (`0x90`) for each:

So first of all we need to open r2 in write mode:

```bash
╰─ ❯ r2 -w pbcompiler
 -- Welcome to "IDA - the roguelike"
[0x00402a40]>
```

Then patch was applied:

```bash
s 0x00422a68
wx 909090909090  ; write 6 NOPs
```

`wx` writes raw hex bytes at the current position.

I verified it:

Before Patching:

```bash
[0x00422a68]> px 10
- offset -  6869 6A6B 6C6D 6E6F 7071 7273 7475 7677  89ABCDEF01234567
0x00422a68  0f8f 3b07 0000 48c7 c0c8                 ..;...H...
[0x00422a68]>
```

After patching:

```bash
[0x00422a68]> px 10
- offset -  6869 6A6B 6C6D 6E6F 7071 7273 7475 7677  89ABCDEF01234567
0x00422a68  9090 9090 9090 48c7 c0c8                 ......H...
[0x00422a68]>
```

The first 6 bytes are `9090 9090 9090` (NOP) now. The `jg` is gone. The check no longer exists.

---

## step 8: Now we run it on our big hello

```bash
╰─ ❯ ./pbcompiler big_hello.pb
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
Error: Source too big for the Free version.
```

What????? -- noooo, why?

<img class="rattip-img" src="/assets/why.gif" width=200 height=200 />

It did not worked.

The patch is in, binary is modified, but the error still fires?

###### I ate a dollop of peanut butter, and drank water while I was choking on it... Stood up, and watched my screen for some time, give up?...Naah!

Bulb erupted...*there could be another check like this somewhere maybe*, that's what triggering
the error now, let's see...

## step 9: in search of the second check

Makes sense actually.
Maybe one check per line during parsing, another one at a different stage.

but when I searched this:

```bash
[0x00402a40]> /a cmp dword [rax], 782
0x00422a62 hit7_0 81380e030000
[0x00402a40]>
```

I got that one hit only... so I tried this:

```
[0x00402a40]> /a cmp rax, 782
```

but got nothing.

Obviously some other register is being used here...

Time for another bash for loop:

```bash
╰─ ❯ for reg in rax rbx rcx rdx rsi rdi rbp; do
echo "reg: $reg"
 r2 -q -c "/a cmp dword [$reg], 782" pbcompiler 2>/dev/null
done
reg: rax
0x00422a62 hit0_0 81380e030000
reg: rbx
reg: rcx
reg: rdx
reg: rsi
reg: rdi
reg: rbp
```

Still only that one hit, that we got before.

###### Ha, my legs hurt from cycling 25 kms yesterday...

Wait...So, what if not 64 bit register, but a 32 bit register is being used?

```bash
╰─ ❯ for reg in eax ebx ecx edx esi edi ebp; do
echo "reg: $reg"
 r2 -q -c "/a cmp dword [$reg], 782" pbcompiler 2>/dev/null
done
reg: eax
0x00422a62 hit0_0 81380e030000
reg: ebx
reg: ecx
...
```

aah still the same only, ofc not `dword` wtf...umm wait:

```bash
╰─ ❯ for reg in eax ebx ecx edx esi edi ebp; do
echo "reg: $reg"
 r2 -q -c "/a cmp $reg, 782" pbcompiler 2>/dev/null
done
reg: eax
reg: ebx
reg: ecx
reg: edx
0x00422d0d hit0_0 81fa0e030000
reg: esi
reg: edi
reg: ebp
```

**Boom Baby!!** found it!

so it's register edx, see:

```bash
[0x00402a40]> /a cmp edx, 782
0x00422d0d hit10_0 81fa0e030000
[0x00402a40]>
```

Now let's check this one out...

## Step 10: Killing Another bugger

```bash
[0x00402a40]> pd 5 @ 0x00422d0d
│           ;-- hit3_0:
│           ;-- hit10_0:
│           0x00422d0d      81fa0e030000   cmp edx, 0x30e              ; 782
│       ┌─< 0x00422d13      0f8f16020000   jg 0x422f2f
│       │   ; CODE XREF from fcn.00422930 @ 0x422f51(x)
│       │   0x00422d19      418b36         mov esi, dword [r14]
│       │   0x00422d1c      85f6           test esi, esi
│      ┌──< 0x00422d1e      740c           je 0x422d2c
```

Haaa, same pattern we can see as before, `jg` jumping to an error path.

Let's patch this too.

Again open r2 in write mode and:

`jg` is at `0x00422d13` so let's NOP it.

```bash
s 0x00422d13
wx 909090909090
```

Now let's verify it...

Before:

```bash
[0x00422d13]> px 10
- offset -  1314 1516 1718 191A 1B1C 1D1E 1F20 2122  3456789ABCDEF012
0x00422d13  0f8f 1602 0000 418b 3685                 ......A.6.
```

```bash
[0x00422d13]> px 10
- offset -  1314 1516 1718 191A 1B1C 1D1E 1F20 2122  3456789ABCDEF012
0x00422d13  9090 9090 9090 418b 3685                 ......A.6.
```

`9090 9090 9090` done scene.

## The Moment of truth

```
╰─ ❯ ./pbcompiler big_hello.pb
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
802 lines processed.
Creating and launching executable.

- Feel the ..PuRe.. Power -

Hello, World
Hello, World
Hello, World
Hello, World
Hello, World
```

Wohooooo! 802 lines processed, and ran! What? **Pure Basic you are hacked bro.**

but wait...

let's try with bigger hello:

```bash
╰─ ❯ for i in {0..5000}; do
  echo "PrintN(\"Hello, World\")" >> big_hello.pb;
done


╰─ ❯ wc -l big_hello.pb
5001 big_hello.pb
```

And then I run it:

```bash
╰─ ❯ ./pbcompiler big_hello.pb -e hello
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
Error: Line 0 - Source too big for the Free version.
```

Faileddd...
Are you kidding?...Another check I see.

Notice something different this time though

`Line 0`.
Before it just said `Error: Source too big for the Free version.`
with no line number. Now it's `Line 0 - Source too big...`.

Line 0 means the check fired before the compiler even started parsing lines.
This is a different check, somewhere earlier in the pipeline.

Let's fire r2 again...it's work is not yet done ig.

## step 11: hunting the Line 0 check

###### Eating a mango...it's so windy outside now.

We are so close, like already we have done enought but still now we see this, let's find out what is this check
now, like why this one too, let's see.

First I searched for the format string responsible for printing `Line 0 - ...`:

```bash
[0x00402a40]> iz | grep -i "line %"
247  0x00054f13 0x00454f13 15  16   .rodata ascii   %s (%s line %d)
248  0x00054f23 0x00454f23 22  23   .rodata ascii   Warning: Line %d - %s\n
250  0x00054f45 0x00454f45 20  21   .rodata ascii   Error: Line %d - %s\n
1484 0x00059988 0x00459988 34  35   .rodata ascii   in included file '%s'\nLine %d - %s
1485 0x000599b0 0x004599b0 45  46   .rodata ascii   at line %d of the expanded macro (Macro.out)\n
[0x00402a40]>
```

And there it is `250  0x00054f45 0x00454f45 20  21   .rodata ascii   Error: Line %d - %s\n`.

Now xrefs:

```bash
[0x00402a40]>  axt 0x00454f45
fcn.00416910 0x416aa9 [STRN:r--] lea rdi, str.Error:_Line__d____s_n
[0x00402a40]>
```

One caller.
So `fcn.00416910` seems to be a central error printing function that handles both formats.
With and without line number.

Looked at all its callers using the XREFs at the top of the function.

For some time I traced through the whole call chain and eventually landed on the real question:
who is setting up the "Source too big" error (index `0x50`) through this new path?

![meme](/assets/later.png)

Searched for every place `0x50` is moved into `edi` before a dispatcher call:

```bash
[0x00402a40]> /a mov edi, 0x50
0x00415b37 hit3_0 bf50000000
0x0041bc04 hit3_1 bf50000000
0x00422f36 hit3_2 bf50000000
0x004231a9 hit3_3 bf50000000
0x0042ad7d hit3_4 bf50000000
0x0042d2b5 hit3_5 bf50000000
0x0042d34d hit3_6 bf50000000
0x0042f776 hit3_7 bf50000000
[0x00402a40]>
```

The interesting new one was `0x00415b37`. Let's look at what's around it:

```bash
[0x00402a40]> pdf @ fcn.00415ac0
...
```

And in the assembly I saw it, hiding in plain sight:

```asm
│           0x00415ad4      8902           mov dword [rdx], eax
│           0x00415ad6      3d54120000     cmp eax, 0x1254             ; 'T\x12'
│       ┌─< 0x00415adb      7f53           jg 0x415b30
│       │   ; CODE XREF from fcn.00415ac0 @ 0x415b47(x)
│      ┌──> 0x00415add      48c7c01c2a..   mov rax, 0x4a2a1c           ; '\x1c*J'
```

`0x1254` is **4692** in decimal.
This function is called **716 times**.
It's a token/identifier counter, not a line counter at all.

Every time the compiler processes a token it calls this function,
increments a counter, and if that counter exceeds 4692 it fires the error.

And it clicked, previously I was thinking there is a `LOC` limit right? *what if someone writes the whole code on a single
line?* Well there it is, these people are counting tokens too for that purpose, I think...Still for tokens there was higher limit...ok, understandable.

## step 12: Patch this too (Final Patchuh!)

The `jg` here is only 2 bytes (short jump), so the patch is simple:

```bash
[0x00402a40]> s 0x00415adb
[0x00415adb]> wx 9090
[0x00415adb]>
```

verify:

```bash
[0x00415adb]> px 5 @ 0x00415adb
- offset -  DBDC DDDE DFE0 E1E2 E3E4 E5E6 E7E8 E9EA  BCDEF0123456789A
0x00415adb  9090 48c7 c0                             ..H..
[0x00415adb]>
```

`9090` done.

## The REAL Moment of Truth Now

```bash
╰─ ❯ wc -l big_hello.pb
5001 big_hello.pb

╰─ ❯ ./pbcompiler big_hello.pb -e hello
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
5002 lines processed.
Creating executable "hello".

- Feel the ..PuRe.. Power -
```

**Boom Babyyy...**

Wait:

```bash
╰─ ❯ rm big_hello.pb
for i in {0..10000}; do
 echo "PrintN(\"Hello, World\")" >> big_hello.pb
done

╰─ ❯ wc -l big_hello.pb
10001 big_hello.pb

╰─ ❯ ./pbcompiler big_hello.pb -e hello
PureBasic 6.40 (Linux - x64) Free
Loading external modules...
Starting compilation...
10002 lines processed.
Creating executable "hello".

- Feel the ..PuRe.. Power -
```

<img class="rattip-img" src="/assets/yes2.gif" width=300 height=300 />

**Even 10000 lines of code is also working!** Noice. It's quite fast tbh.

## the three checketiers

Turned out the free version limit isn't enforced by one check, it's enforced by three separate tripwires:

| Patch | Address | What it checks | Limit | How to patch |
|-------|---------|---------------|-------|-------------|
| 1 | `0x00422a68` | line counter (via pointer) | 782 | 6× NOP |
| 2 | `0x00422d13` | line counter (via register) | 782 | 6× NOP |
| 3 | `0x00415adb` | token counter | 4692 | 2× NOP |

The first two were both `jg` instructions (6 bytes each) inside the main parsing loop,
checking the line count in two different ways, probably one for normal files and one for included files.

The third was the sneaky one.
A token counter buried inside a function called hundreds of times per compilation,
with a limit of **4692**
And because it fires in a pre-parse stage,
the line number in the error is always 0,
which is what tipped us off that it was a completely different beast.

**Pure Basic**: three locks on the door. We picked all of 'em.

## things worth taking away from this

##### When axt gives you nothing, look for pointer tables.
Error strings in compiled binaries are often stored in tables, not referenced directly.
When cross reference analysis fails, search for the address bytes themselves using `/x`.
The absence of xrefs is information itself.

##### The 800 lines limit.
PureBasic says 800 lines. The binary checks 782.
If we had only searched for 800 we would have found nothing and given up.
When searching for a magic number, try nearby values too.

##### There are usually multiple checks, not one.
Patching the first `jg` still gave us the error.
There were two separate comparisons.
Always search exhaustively. License checks often have redundant trip wires.

### **some r2 commands cheat sheet from this blog:**

| Command | Description |
|-------|---------|
| aaaa | run full analysis |
| iz | list strings in binary |
| axt <addr> | find cross references to an address |
| /x <hex> | search for hex bytes |
| s <addr> | seek to address |
| pxq <n> | print n bytes as 64-bit hex values |
| ps @ <addr> | print string at address |
| pd <n> | disassemble n instructions |
| wx <hex> | write hex bytes at current position |

The End...Thank You for reading till here.

---

**If you're a PureBasic dev reading this:** Hey, now you know where to look.
No hard feelings, cool language honestly.

Reference: [The Radare2 Book](https://book.rada.re/)
