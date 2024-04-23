# Overview
This repository centers around documenting modern ways of developing logic for and programming Atmel (Now Microchip) 5V GAL PLD and CPLD parts under recent Linux (Ubuntu 22.04) and Windows 10 22H2 versions.
* ![ATF16V8](vendor-datasheets/Atmel-0425-PLD-ATF16V8C-Datasheet.pdf) (Modern/active equivalent of the PAL16V8 and GAL16V8 parts)
* ![ATF22V10](vendor-datasheets/doc0735.pdf) (Modern/active equivalent of the PAL22V10 and GAL22V10 parts)
* ![ATF1502](vendor-datasheets/Atmel-0995-CPLD-ATF1502AS(L)-Datasheet.pdf) (Active replacement for the EPM7032)
* ![ATF1504](vendor-datasheets/Atmel-0950-CPLD-ATF1504AS(L)-Datasheet.pdf) (Active replacement for the EPM7064)
* ![ATF1508](vendor-datasheets/doc0784.pdf) (Active replacement for the EPM7128)

These parts are still active and highly worth considering wherever:
* 5V logic is a requirement, avoiding level shifting, low latency (7ns), instant-on & non-volatile.
* Prototyping / Iteration (reprogrammable)
* Learning about logic: Through-hole / soldering-friendly is desired: All 16V8 or 22V10 parts are available in DIP packages; ATF150x parts are available in PLCC packages that can be placed in through-hole PLCC sockets. SMD packages are available for any of the parts.
* Replacing large quantities of various TTL/CMOS Logic Gates (74-series logic)

In short, these chips work very well wherever the above requirements are necessary, however, the software (and device programming) experience can be incredibly challenging owing to outdated and buggy software.

This repository aims to make it easier to work with these parts and hopefully keep them active for years to come.

This is a "Choose your own adventure novel". Covered here are many approaches and tradeoffs:
* <a href="#old-approach-wincupl-16v8-22v10-and-atf150x">Using the WinCUPL IDE (Erratic, unreliable)</a>
* <a href="#5vcomp-the-cupl-compiler--your-favorite-text-editor-or-ide-16v8-22v10-and-atf150x">Using just the CUPL.EXE compiler from WinCUPL directly with some wrapper scripts here (5vcomp). Works in Windows/Linux. (recommended)</a>
* <a href="#quartus-free-verilog-vhdl-schematic-capture-indirect-support-for-atf150x-linux-or-windows">Using Quartus (only for the CPLD. Works by first targeting a similar Altera CPLD and then using the POF2JED utility to convert.) Windows/Linux</a>
* <a href="#absurd-approach-fusemaps-by-hand-16v8--22v10">Making your own fusemap / .JED file with nothing more than a datasheet and text editor. Maybe need some graph paper...</a>
* Experimental approaches with Yosys (Only for the CPLD parts. EDIF is fed into the Atmel fitter)
* Several Approaches to reverse-engineering a .JED file back into logic equations.

This is mostly a collection of documentation, but if you are interested in using CUPL.EXE directly, some small scripts (5vcomp) are here that help make things easier and provide examples on how to avoid WinCUPL while still utilizing these parts:
* ![Linux Workflow (5vcomp command-line utility pointed at a .PLD file)](linux-workflow/)
* ![Windows Workflow (5vcomp.bat utility called by right-clicking a .PLD file to get a compiled/synthesized .JED file).](windows-workflow/)

<details>
<summary>Scope: Expand here for why similar parts not covered</summary>

* The intention is primarily to make it easier to work with the parts that exist so they do not go NRND and eventually disappear from the market.
  * While there are gems here for other related historical parts, this is not the focus of this repository.
* The ATF1500 is not covered because it is a more expensive part and does not support JTAG programming. It is fundamentally different from the ATF1502, ATF1504, and ATF1508
* The ATF750 and ATF2500 are also not covered for similar reasons. Other chips are almost certainly a better choice.
* We only consider true 5V parts (not merely parts with 5V tolerant inputs, of which there are many more).
  * 3.3V parts cannot supply the minimum of 3.6V to drive the input of a 5V CMOS part high, so 5V tolerant parts are not enough in many cases.
  * Notably, however, driving 5V TTL inputs from a 3.3V part, on the other hand, is not a problem. A 5V TTL input has a threshold of 2V.
* 3.3V parts are not considered: There are simply better choices that are well documented. Also, the CPLD parts have VccIO inputs, so you can technically use them in 3.3V designs just as well.
* The Greekpak devices probably should be covered here, but, they're reasonably well documented with modern tools.
* Any parts that are NRND or inactive are not covered, as we consider what can be reliably and sensibly purchased.
* Since all of the parts considered here are still in full production (as of 2023), they can be used in production designs.
* For the ATF150x CPLD parts specifically:
  * The BE and ASV devices not covered here as they seem to be difficult to obtain and are not 5V devices. If you need 3.3V or lower, there are probably better parts suited to your needs.
* For applications where 5-Volt tolerant operation is acceptable, it might be worth considering the ispMACH4000 series. Parts such as the LC4032ZE are available is a somewhat soldering friendly TQFP, however there are likely similar challenges with software.
</details>

# Background on digital logic.
This repository isn't intended to be an introduction to digital logic, but a brief review and compare/contrast to similar things is provided here.

<details>
<summary>Expand here for tutorials on Digital Logic</summary>
Ben Eater does a series of <a href="https://www.youtube.com/watch?v=KM0DdEaY5sY&list=PLowKtXNTBypGqImE405J2565dvjafglHU&index=6">Videos on Digital Logic</a> that are a really excellent introduction to some of the concepts here.
</details>

<details>
<summary>Expand here for a description of how these parts compare to ladder logic on a PLC</summary>

* Each rung's output in ladder-logic can be thought of as a single macrocell.
* The inputs on a rung can be "normally open" or "normally closed" (active high or low), and can consist of any number of inputs (or even the state of another macrocell). The inputs defined on a single rung are basically equivalent to a single product-term belonging to a macrocell. There can be multiple product terms defined that activate a given macrocell.
</details>

<details>
<summary>Expand here for details on how all of these compare to FPGAs</summary>
Such parts are the spiritual predecessors of more modern FPGAs. Key differences between FPGAs and PLDs:

* FPGAs are typically constructed from a large number of LUTs (Lookup tables). CPLDs use a sum-of-products structure.
* FPGAs typically expect to have their bitstream uploaded on powerup, requiring an external EEPROM. PLDs are typically non-volatile and instantly ready upon powerup.
* FPGAs usually support more standard means of programming, whereas many PLDs required specialized device programmers.
* There are likely exceptions to all of the above in some parts. These are not hard rules.
</details>

# Requirements
A high-level overview of what is required:
* Basic understanding of digital logic.
* The actual PLD/CPLD chip you'd like to work with from the usual suppliers (Mouser, Digikey, Octopart)
* A software workflow covered here. Highly recommended is using the 5vcomp script from here to call the CUPL.EXE compiler. This works on Linux and even Windows 10 22H2 x64!
* An EPROM/Device programmer if you wish to use the ATF16V8 or ATF22V10 parts.
* An EPROM or JTAG programmer for the ATF150x parts
![See PROGRAMMING.md](PROGRAMMING.md) for details on what it takes to program these parts in detail.

# Terminology & File Formats
<a href="https://en.wikipedia.org/wiki/Programmable_logic_device">PLD/GAL</a> - Programmable Logic Device. Small, generally DIP-package 5V programmable Logic.<br />
<a href="https://en.wikipedia.org/wiki/Programmable_logic_device#CPLDs">CPLD - </a>Complex Programmable Logic Device. Larger packages, many pins, much more complex.<br />

5vcomp - A utility in this repository (batch file for Windows / shell script for linux) that is a wrapper around the CUPL.EXE compiler<br>

<a href="https://en.wikipedia.org/wiki/Combinational_logic">Combinatorial Logic</a> - Simple logic (AND, OR, NOT, gates, etc.) that does not use flip-flops / registers / clocks. Such logic could technically be implemented with an EPROM/Memory, where a series of inputs always maps to a known set of outputs.<br>
<a href="https://en.wikipedia.org/wiki/Sequential_logic">Registered Logic</a> - Logic that uses registers (flip-flops), and can thus hold state. On the GAL16V8 and GAL22V10, each macrocell can be configured as a D-Flip-Flop, and all flip-flops share the same clock pin. On the ATF150x, much more complex types of registered logic and clocking options are available.

<a href="https://en.wikipedia.org/wiki/Macrocell_array">Macrocell</a> - Each output has a macrocell associated with it. These can often be configured as active high, active low, flip-flops, etc.<br />
<a href="https://en.wikipedia.org/wiki/Programmable_logic_device#/media/File:Programmable_Logic_Device.svg">Product Term</a> - Each macrocell has a number of product terms associated with it (typically around 5). A product term is essentially a giant AND gate with inputs to each pin on the device. Burning away fuses allows selecting which inputs are fed into this AND gate, ultimately selecting the conditions required for a product term to be activated. Product terms belonging to the same output macrocell are then combined into an OR gate before being fed into the macrocell. This means that there can be several combination of inputs that allow a given macrocell to be triggered. This architecture is called a Sum-of-Products logic array.

<a href="https://en.wikipedia.org/wiki/Programmable_Array_Logic#CUPL">CUPL</a> - A early (1983) programming language used to define the behavior of digital logic gates. "Compiler for Universal Programmable Logic.", is essentially a predecessor to languages like Verilog/VHDL. CUPL.EXE is the compiler which is used to compile .PLD files written in CUPL, ultimately to be burned into programmable logic devices.<br />
<a href="https://www.microchip.com/en-us/products/fpgas-and-plds/spld-cplds/pld-design-resources">WinCUPL</a> - A Windows front-end/IDE to the CUPL compiler and related programs. It is still part specifically that we are trying to avoid, while keeping everything else underneath/around it as it is buggy.<br />
[.dl File](device-library/) - Device Library File. This file determines what devices CUPL has the ability to compile for.<br>

<a href="https://en.wikipedia.org/wiki/Netlist">Netlist</a> - A netlist is essentially an electrical schematic in a text file which defines connections. For the purposes here, it is an intermediary file format (Either EDIF or Berkeley PLA), which is used to describe the behavior of logic ultimately fed into the fitter.<br />
<a href="">.TT2</a> - The Berkeley PLA file format. An intermediary file which CUPL.EXE can generate that can be used by the Atmel fitters. Notably, one can use [berkeley-abc](https://github.com/berkeley-abc/abc) to work with these files.<br />
<a href="https://en.wikipedia.org/wiki/EDIF">.EDF / .EDN</a> - EDIF is another type of netlist format. The Atmel fitter can use this as both an input, as well as an output. Yosys is capable of generating this format, however, one will still need a techmap for this to work.<br />
<a href="https://en.wikipedia.org/wiki/Place_and_route">Fitter</a> - A fitter converts a netlist into the fusemap (.JED) file. Fitters are needed for the ATF150x CPLD devices. In more modern parlance, this is basically place & route.<br />
ATMEL.STD File - Part of the Atmel ATF150x fitter, the primitive/device library for PLA.[^1]<br />
APRIM.LIB File - Part of the Atmel ATF150x fitter, the primitive/device library for EDIF.[^1]

<a href="https://archive.org/details/JEDECJESD3C/mode/2up">.JED/JEDEC File</a> - A fuse map intended to be "burned/programmed" into a logic device. A JEDEC file is ultimately a text file formatted specifically to the JESD3 standard.


.SVF File - Serial Vector Format. Generated from the .JED file, the .SVF can be used by any JTAG programmer (vendor-independent) to program a device that has a JTAG interface.<br />
CSIM - A tool for simulating the behavior of logic. This takes an .SI file (test vectors) and an [.ABS file](abs-decode/). Given these it produces an .SO file. This is not concerned with timing, but simply logic states.


<a href="https://www.winehq.org/">Wine</a> - Wine is not an emulator. Allows running Windows programs under Linux.<br />


# Writing logic for these parts: Possible Workflows
Each of the subsections here represents a potential workflow to design logic equations for these parts. The majority of the focus will be on methods that avoid using the WinCUPL frontend/IDE directly (unreliable), but which do use the underlying CUPL.EXE command-line compiler which is fairly robust.

Finally, a word on preferred approach, given the options: Using the CUPL.EXE compiler via command line or Quartus are probably the best ways, especially if you are interested in using Hi-Z states. Neither Yosys nor Digital seem to have robust support for 'inout' ports, Tristate/Hi-Z states, open collector outputs, etc. If these are important to you, it may be worth considering tools and workflows which are less experimental.

## Old Approach: WinCUPL (16V8, 22V10, and ATF150x)
WinCUPL is basically an IDE, possibly before the term IDE came into existance. While logic for these parts can be written using WinCUPL itself, the experience may be fraught with difficulty as it is a quirky and often unstable Windows application (While it does run great under Wine, this doesn't really change things much). I've seen the editor itself crash just for looking at it sideways, and its copy-paste functionality behave in bewildering ways.
It does however have value in the help files / documentation / examples. It should emphasized that the CUPL compiler itself is actually pretty solid/stable, and so the troubles of WinCUPL shouldn't be equated with CUPL itself.
So, the recommended approach is to start here regardless and use it for documentation/examples/compiler/device-library and then simply avoid it for serious work by using the command line CUPL.EXE (perhaps through the 5vcomp helper scripts in this repository as in the next section). 

You can <a href="https://www.microchip.com/en-us/products/fpgas-and-plds/spld-cplds/pld-design-resources">Download WinCUPL from here</a>.

To get it working under Linux with Wine, you'll need winetricks so you can install mfc40 and mfc42. On Ubuntu Linux, this would look something like:

<code>sudo apt-get install wine winetricks playonlinux
winetricks mfc40 mfc42
</code>

Furthermore, if you are intending on working with the ATF150x parts, you should probably grab the newer fitters out of the Atmel Prochip package. The utilities in this repository will refuse to work with the old fitters.

## 5vcomp: The CUPL compiler & Your favorite text editor or IDE (16V8, 22V10, and ATF150x)
Since WinCUPL simply is a front-end / IDE on top of the CUPL.EXE compiler and related programs, one can write the desired logic in the CUPL language, save it in a .PLD file using their favorite editor and have CUPL.EXE compile it into a .JED file for programming into a PLD.

5vcomp is a simple wrapper around the CUPL compiler.
This is probably the most solid approach assuming you are OK with using CUPL as a language. You should start with the WinCUPL approach as a prerequisite since it installs the CUPL compiler and has examples/help files.
The workflows here simply make this easier/conveinent by catching a lot of common issues and providing reasonable defaults to the compiler:

* ![Linux Workflow (point 5vcomp at your .PLD file from a command line)](linux-workflow/)
* ![Windows Workflow (right-click on a .PLD to compile with 5vcomp.bat)](windows-workflow/)


## Guide to CUPL itself
Assuming you're using CUPL either through WinCUPL or 5vcomp, this section has a general reference to the language, device library details, etc.

Here's an overview of how things work:
An overview of how things work from a diagram built into WinCUPL which shows how one can go from a CUPL .PLD into the .JED files needed to program a device. Note that the Atmel Fitter (place and route) stage is only used when working with the ATF150x CPLD parts. For simpiler CPLD parts, 

![WinCUPL Data Flow Diagram](vendor-docs/WinCUPL-data-flow-diagram.png)


![A detailed User's Guide to the CUPL compiler and language reference in PDF](vendor-docs/CUPL_USERS_GUIDE.pdf)

## 🟥 Common Pitfall - Compiler Mode Selection🟥<br>
A word of warning is that the <code>Device:</code> section at the top of a .PLD file is more than just the part number you are interested in programming -- it is actually a device mnemonic which selects different macrocell configuration modes.<br>
So, if you're having trouble getting a flip-flop to work, it might be because you have selected the mnemonic for "simple mode".<br>
As an example, the compiler can be set to four different modes for the ATF16V8 (similar considerations apply to the 22V10 parts, etc):<br>
Registered - G16V8MS<br>
Complex - G16V8MA<br>
Simple - G16V8AS<br>
Auto - G16V8

<details>
<summary>Expand here for details of the command line flags for CUPL.EXE</summary>
Run CUPL using the following command line format:

<code>cupl [-flags] [library] [device] source
where
-flags is the following set of compiler options:
-j JEDEC download format
-h ASCII-HEX download format
-i HL download format
-n use input filename for output file
-a create absolute file
-l create listing file
-e create expanded macro definition file
-x create expanded product-terms in documentation file
-f create fuse plot/chip diagram in documentation file
-p create PDIF database interchange format file
-b create Berkeley PLA format file
-c create PALASM format file
-d deactivate unused OR terms
-r disable product term merging
-g program security fuse
-o treat all state machines as “one-hot”
-u use specified library for compilation
-s perform logic simulation after compilation
-w perform simulation with waveform output (MS-DOS only)
-m0 no minimization
-m1 quick minimization (default)
-m2 Quine McCluskey
-m3 Presto
-m4 Expresso
-q MIcrosoft format for error messages
-zq QuickLogic’s QDIF file
-kb Optimize product term usage for pin or pinnode variables. This overrides the DEMORGAN statement if it appears in the source file
-kd DeMorganize all pin and pinnode variables. This overrides the DEMORGAN statement if it appears in the source file
-ks Force product term sharing during minimization. This is also referred to as group reduction
-kx Do not expand XOR to AND-OR equations. This is used for device independent designs or designs targeted for fitter-supported devices where the fitter supports XOR gates
</code>
</details>

<details>
<summary>Expand here if you are interested in using VS Code as an IDE</summary>
 
Recently, two different extensions for VS Code for CUPL have been written:
* https://marketplace.visualstudio.com/items?itemName=tlgkccampbell.code-cupl
  * This one handles just syntax highlighting for CUPL .PLD files
* https://marketplace.visualstudio.com/items?itemName=VaynerSystems.VS-Cupl
  * This is an entire workflow, which has a bit more functionality beyond just syntax highlighting.
</details>

<details>
<summary>Expand here for a list of devices Atmel WinCUPL supports</summary>
 
* CBLD.EXE will allow you to see a list of devices that are supported within the CUPL.DL device library.
* Atmel WinCUPL is limited Atmel devices, however, other versions of CUPL found elsewhere will have parts from a broader array of manufacturers.

<code>wine ./cbld.exe -l
CBLD(PM): CUPL Device Library Management Program
Version 5.0a
Copyright (c) 1983, 1998 Logical Devices, Inc.
C:\Wincupl\Shared\CUPL.DL  rev:DLIB-h

Device        Rev   Pins  Fuses  Pterms
------------  ---   ----  -----  ------
v750           03    24   14394    171
v750b          02    24   14435    171
v750c          02    24   14504    171
v750cext       02    24   14504    171
v750cextppk    02    24   14504    171
v750cppk       02    24   14504    171
v2500          07    40   71648    416
v2500b         04    40   71745    416
v2500c         04    40   71816    416
v2500cppk      04    40   71816    416
f1500          01    44   15360    320
f1500t         01    44   15360    320
f1500a         01    44   15360    320
f1500at        01    44   15360    320
f1502plcc44    01    44   15360    320
f1502ispplcc44   01    44   15360    320
f1502tqfp44    01    44   15360    320
f1502isptqfp44   01    44   15360    320
f1504plcc44    01    44   15360    320
f1504ispplcc44   01    44   15360    320
f1504tqfp44    01    44   15360    320
f1504isptqfp44   01    44   15360    320
f1504plcc68    02    68   15360    320
f1504ispplcc68   02    68   15360    320
f1504plcc84    02    84   15360    320
f1504ispplcc84   02    84   15360    320
f1504qfp100    02   100   15360    320
f1504ispqfp100   02   100   15360    320
f1504tqfp100   02   100   15360    320
f1504isptqfp100   02   100   15360    320
f1508plcc84    02    84   15360    320
f1508ispplcc84   02    84   15360    320
f1508qfp100    02   100   15360    320
f1508ispqfp100   02   100   15360    320
f1508tqfp100   02   100   15360    320
f1508isptqfp100   02   100   15360    320
f1508pqfp160   01   160   15360    320
f1508isppqfp160   01   160   15360    320
atfvirtual     01    44   99999   5000
g16v8          09    20    2194     64
g16v8ma        08    20    2194     64
g16v8ms        11    20    2194     64
g16v8a         03    20    2194     64
g16v8as        02    20    2194     64
g16v8s         09    20    2194     64
g16v8cpms      01    20    2195     64
g16v8cp        01    20    2195     64
g16v8cpas      01    20    2195     64
g16v8cpma      01    20    2195     64
g20v8          03    24    2706     64
g20v8ma        03    24    2706     64
g20v8ms        03    24    2706     64
g20v8a         02    24    2706     64
g20v8as        01    24    2706     64
g20v8cp        02    24    2707     64
g20v8cps       01    24    2707     64
g20v8cpma      03    24    2707     64
g20v8cpms      03    24    2707     64
g22v10         01    24    5892    132
g22v10cp       01    24    5893    132
p22v10         17    24    5828    132
virtual        01   200   99999   5000
v750lcc        07    28   14394    171
v750blcc       02    28   14435    171
v750clcc       02    28   14504    171
v750cextlcc    02    28   14504    171
v750cextppklcc   02    28   14504    171
v750cppklcc    02    28   14504    171
v2500lcc       08    44   71648    416
v2500blcc      04    44   71745    416
v2500clcc      04    44   71816    416
v2500cppklcc   04    44   71816    416
g20v8lcc       03    28    2706     64
g20v8alcc      02    28    2706     64
g20v8aslcc     01    28    2706     64
g20v8malcc     03    28    2706     64
g20v8mslcc     03    28    2706     64
g20v8slcc      03    28    2706     64
g20v8cplcc     02    28    2707     64
g20v8cpslcc    01    28    2707     64
g20v8cpmalcc   03    28    2707     64
g20v8cpmslcc   03    28    2707     64
g22v10lcc      02    28    5892    132
g22v10cplcc    01    28    5893    132
p22v10lcc      17    28    5828    132
</code>
</details>

<details>
<summary>Expand here for a list of devices the version of CUPL provided by PLDmaster of Logical Devices, Inc. supports</summary>

<code>

wine ./cbld.exe -l -u pldmstr.dl
CBLD(PM): CUPL Device Library Management Program
Version 5.0a
Copyright (c) 1983, 1998 Logical Devices, Inc.
pldmstr.dl  rev:DLIB-h

Device        Rev   Pins  Fuses  Pterms
------------  ---   ----  -----  ------
ep300          21    20    2720     74
ep320          02    20    2916     72
ep312          07    24   13713    200
ep600          14    24    6482    160
27c64dip       01    28       0   8192
27c64plcc      01    32       0   8192
27c128dip      01    28       0   16384
27c128plcc     01    32       0   16384
27c256dip      01    28       0   32768
27c256plcc     01    32       0   32768
g16v8          09    20    2194     64
g16v8a         03    20    2194     64
g16v8as        02    20    2194     64
g16v8ma        08    20    2194     64
g16v8ms        11    20    2194     64
g16v8s         09    20    2194     64
g20v8          03    24    2706     64
g20v8s         02    24    2706     64
g20v8ma        03    24    2706     64
g20v8ms        03    24    2706     64
g20v8a         02    24    2706     64
g20v8as        01    24    2706     64
g22v10         01    24    5892    132
p10h8          07    20     320     16
p10l8          07    20     320     16
p10p8          07    20     328     16
p10p8v         01    20     664     32
p12h6          09    20     384     16
p12l6          07    20     384     16
p12p6          07    20     390     16
p12p6v         01    20     786     32
p12l10         07    24     480     20
p12p10         07    24     490     20
p14h4          07    20     448     16
p14l4          07    20     448     16
p14p4          07    20     452     16
p14p4v         01    20     908     32
p14l8          07    24     560     20
p14p8          09    24     568     20
p16c1          07    20     512     16
p16h2          07    20     512     16
p16l2          07    20     512     16
p16p2          07    20     514     16
p16p2v         01    20    1030     32
p16r4          11    20    2048     64
p16rp4         15    20    2056     64
p16rp4v        01    20    2072     64
p16l6          07    24     640     20
p16p6          07    24     646     20
p16r6          14    20    2048     64
p16rp6         15    20    2056     64
p16rp6v        01    20    2072     64
p16h8          08    20    2048     64
p16hd8         06    20    2048     64
p16l8          08    20    2048     64
p16ld8         06    20    2048     64
p16n8          01    20     512     16
p16p8          08    20    2056     64
p16p8h         07    20    2056     64
p16p8v         01    20    2072     64
p16r8          10    20    2048     64
p16rp8         14    20    2056     64
p16rp8v        01    20    2072     64
p18l4          07    24     720     20
p18p4          07    24     724     20
p20c1          07    24     640     16
p20l2          07    24     640     16
p20p2          07    24     642     16
p20r4          10    24    2560     64
p20rs4         21    24    3330     80
p20r6          10    24    2560     64
p20l8          08    24    2560     64
p20p8          01    24    2568     64
p20r8          09    24    2560     64
p20rs8         19    24    3338     80
p20l10         06    24    1600     40
p20rs10        19    24    3338     80
p20s10         17    24    3322     80
p22v10         17    24    5828    132
ra5p8          04    16     256     32
ra5p16         02    24     512     32
ra6p16         02    24    1024     64
ra8p4          04    16    1024    256
ra8p8          04    20    2048    256
ra9p4          04    16    2048    512
ra9p8          04    20    4096    512
ra10p4         04    18    4096   1024
ra10p8         04    24    8192   1024
ra11p4         04    18    8192   2048
ra11p8         04    24   16384   2048
ra12p4         04    20   16384   4096
ra12p8         04    24   32768   4096
ra13p8         04    24   65536   8192
virtual        01   200   99999   5000
ep312lcc       01    28   13713    200
ep600lcc       14    28    6482    160
g20v8lcc       03    28    2706     64
g20v8malcc     03    28    2706     64
g20v8mslcc     03    28    2706     64
g20v8slcc      03    28    2706     64
g20v8alcc      02    28    2706     64
g20v8aslcc     01    28    2706     64
g22v10lcc      02    28    5892    132
p12l10lcc      07    28     480     20
p12p10lcc      07    28     490     20
p12l10mlcc     08    28     480     20
p14l8lcc       07    28     560     20
p14p8lcc       09    28     568     20
p14l8mlcc      08    28     560     20
p16r4alcc      02    28    2048     64
p16l6lcc       07    28     640     20
p16p6lcc       07    28     646     20
p16l6mlcc      08    28     640     20
p16l8alcc      01    28    2048     64
p16r6alcc      01    28    2048     64
p16l8          08    20    2048     64
p16r8alcc      01    28    2048     64
p18l4lcc       07    28     720     20
p18p4lcc       07    28     724     20
p18l4mlcc      08    28     720     20
p20c1lcc       07    28     640     16
p20c1mlcc      08    28     640     16
p20l2lcc       07    28     640     16
p20r4lcc       11    28    2560     64
p20rs4lcc      21    28    3330     80
p20r4mlcc      11    28    2560     64
p20rs4mlcc     22    28    3338     80
p20r6lcc       11    28    2560     64
p20r6mlcc      11    28    2560     64
p20l8lcc       09    28    2560     64
p20p8lcc       01    28    2568     64
p20r8lcc       10    28    2560     64
p20l8mlcc      09    28    2560     64
p20l10lcc      06    28    1600     40
p20rs10lcc     19    28    3338     80
p20s10lcc      17    28    3322     80
p20rs10mlcc    20    28    3338     80
p20s10mlcc     17    28    3322     80
p22v10lcc      17    28    5828    132
v750           03    24   14394    171
v750b          02    24   14435    171
v750c          02    24   14504    171
v750cext       02    24   14504    171
v750cextppk    02    24   14504    171
v750cppk       02    24   14504    171
v2500          07    40   71648    416
v2500b         04    40   71745    416
v5000          02    68   163164   1232
f1500          01    44   15360    320
f1500t         01    44   15360    320
f1500a         01    44   15360    320
f1500at        01    44   15360    320
f1502plcc44    01    44   15360    320
f1502ispplcc44   01    44   15360    320
f1502tqfp44    01    44   15360    320
f1502isptqfp44   01    44   15360    320
f1504plcc44    01    44   15360    320
f1504ispplcc44   01    44   15360    320
f1504tqfp44    01    44   15360    320
f1504isptqfp44   01    44   15360    320
f1504plcc68    02    68   15360    320
f1504ispplcc68   02    68   15360    320
f1504plcc84    02    84   15360    320
f1504ispplcc84   02    84   15360    320
f1504qfp100    02   100   15360    320
f1504ispqfp100   02   100   15360    320
f1504tqfp100   02   100   15360    320
f1504isptqfp100   02   100   15360    320
f1508plcc68    02    68   15360    320
f1508ispplcc68   02    68   15360    320
f1508ispplcc84   02    84   15360    320
f1508plcc84    02    84   15360    320
f1508ispqfp100   02   100   15360    320
f1508qfp100    02   100   15360    320
f1508tqfp100   02   100   15360    320
f1508isptqfp100   02   100   15360    320
f1508pqfp160   01   160   15360    320
f1508isppqfp160   01   160   15360    320
ep324          05    40   47493    394
ep900          07    40   17402    240
ep1200         03    40   15146    236
ep1800         11    68   42490    480
p7b336         01    28     384     16
p7b337         01    28     768     32
p7b338         01    28     384     16
p7b339         01    28     768     32
p7c330         05    28   17082    258
p7c331         07    28   11934    216
p7c332         02    28    9902    194
p7c335         01    28   17082    258
xl78c800       01    24    6400     66
c258           01    28   32768   2048
c259           01    44   32768   2048
c330           01    28   17082    258
c331           01    28   11934    216
c332           01    28    9902    194
c335           01    28   17082    258
c336           01    28     384     16
c337           01    28     768     32
c338           01    28     384     16
c339           01    28     768     32
f100           10    28    1928     48
f103           09    28     297      9
f105           21    28    3553     48
f151           13    20     564     15
f153           11    20    1842     42
f155           14    20    2108     43
f157           15    20    2108     43
f159           19    20    2108     43
f161           14    24    1544     48
f162           07    24     165      5
f163           07    24     225      9
f167           23    24    3361     48
f168           05    24    3553     48
f173           01    24    2178     42
f179           03    24    2452     43
f253           01    20    2378     42
f273           02    24    2714     42
f405           06    28    5410     64
f415           03    28    5751     68
f473           03    24    1499     24
f501           02    52   15780    112
f502           01    68   23464    144
f506           06    24   10680     98
f507           05    24    7370     80
f529           01    20     128      8
f839           09    24    1094     32
f30k12         02    28    7424     72
f30s16         02    28    7236     71
f42va12        05    24    8994    105
f48n22         01    68    7008     73
f9800          02    20    1830     45
f7024          02    24   14088     80
f7128          02    28    8488     68
f7140          02    40   22103    120
f2552          01    68   51624    226
f2852          01    84   51624    226
f16v8          02    20    2617     72
f16v8d         02    20    2617     72
f16v8s         02    20    2617     72
f18v8z         07    20    2689     72
f18v8zd        07    20    2689     72
f18v8zs        07    20    2689     72
f20v8          03    24    3193     72
f20v8d         03    24    3193     72
f20v8s         03    24    3193     72
g16v8h         04    24    2226     64
g16v8hs        04    24    2226     64
g16v8hma       04    24    2226     64
g16v8hms       04    24    2226     64
g16v8cpms      01    20    2195     64
g16v8cp        01    20    2195     64
g16v8cpas      01    20    2195     64
g16v8cpma      01    20    2195     64
g16v8c         01    20    2194     64
g16v8cas       01    20    2194     64
g16v8cma       01    20    2194     64
g16v8cms       01    20    2194     64
g16vp8         01    20    2202     64
g16vp8s        01    20    2202     64
g16vp8ma       01    20    2202     64
g16vp8ms       01    20    2202     64
g16z8          03    24    2195     64
g16z8ma        03    24    2195     64
g16z8ms        03    24    2195     64
g16z8s         03    24    2195     64
g18v10         01    20    3540     96
g20vp8         01    24    2714     64
g20vp8s        01    24    2714     64
g20vp8ma       01    24    2714     64
g20vp8ms       01    24    2714     64
g20xv10        03    24    1671     40
g20xv10i       03    24    1671     40
g20xv10f       03    24    1671     40
g20ra10        03    24    3274     80
g22v10i        02    28    5892    132
gds22          01    28     219     11
g22v10cp       01    24    5893    132
g24v10         02    28    3492     80
g24v10ma       02    28    3942     80
g24v10ms       02    28    3942     80
g24v10s        02    28    3942     80
g26cv12        03    28    6432    120
g6001          15    24    8294     75
g6002          04    24    8330     75
mach110        04    44    6504    140
mach111        02    44    6504    140
mach120        04    68   12080    216
mach130        03    84   15552    280
mach131        01    84   15552    280
mach210        04    44   12832    280
mach211        02    44   12832    280
mach215        04    44   11936    256
mach220        05    68   23232    416
mach221        05    68   23232    416
mach231        03    84   30144    544
mach355        01   144   43740    480
mach435        03    84   54096    720
mach445        01   100   54129    720
mach446        01   100   54129    720
mach465        01   208   121185   1440
m16v8a         01    20    3136     64
m16v8as        01    20    3136     64
m16v8ac        01    20    3136     64
m16v8ar        01    20    3136     64
p1012c4        01    24    2056     32
p1016c4        01    28    2056     32
plsi1016ld4    03    24    2056     64
p1016lm4       02    24    2056     32
p1016p4        01    24    2056     32
p1016rd4       03    24    2056     64
p1016rm4       02    24    2056     32
p1016et6       01    24    1542     48
p1016ld8       03    24    2056     64
p1016p8        02    24    2056     64
p1016pe8       01    28    2056     64
p1016rd8       03    24    2056     64
p1020rp4       01    28    2568     32
p1020eg8       04    24    3616     80
p1020ev8       06    24    3616     90
p1020g8        02    24    1352     32
p1020p8        06    24    1352     32
p6l16          02    24     192     16
p8l14          02    24     224     16
p14r21         01    24    3137     86
p16p4c         01    24    2056     32
p16rsp4        01    20    2176     64
p16rsp6        01    20    2180     64
p16p8c         01    24    2056     64
p16ra8         05    20    2056     64
p16rsp8        01    20    2184     64
p16sp8         01    20    2168     64
p18cv8         02    20    2696     74
p18g8          01    20    2624     72
p18n8          01    20     304      8
p18p8          03    20    2600     72
p18u8          02    20    2688     72
p18v8          01    20    2752     74
p19r4r         06    24    2443     64
p19r4t         02    24    2443     64
p19r6r         06    24    2443     64
p19r6t         02    24    2443     64
p19l8r         07    24    2443     64
p19l8t         02    24    2443     64
p19r8r         06    24    2443     64
p19r8t         02    24    2443     64
p20rp4         01    24    2568     64
p20rp4a        02    24    3450     86
p20rsp4        01    24    2688     64
p20x4          11    24    1600     40
p20xrp4        02    24    3450     86
p20rp6         01    24    2568     64
p20rp6a        03    24    3370     84
p20rsp6        01    24    2692     64
p20xrp6        01    24    3370     84
p20rp8         01    24    2568     64
p20rp8a        04    24    3290     82
p20rsp8        01    24    2696     64
p20sp8         01    24    2680     64
p20x8          12    24    1600     40
p20xrp8        01    24    3290     82
p20cg10        02    24    4088     92
p20g10         03    24    3990     90
p20ra10        15    24    3210     80
p20rp10a       02    24    3210     80
p20x10         08    24    1600     40
p20xrp10       01    24    3210     80
p22cv10z       02    24    5873    132
p22ip6         01    24    3294     72
p22p10a        02    24    3970     90
p22rx8         05    24    3616     82
p22v10s        03    24    6657    132
p22vp10        05    24    5838    132
p22xp10        02    24    3970     90
p23s8          06    20    6234    135
p23sv8         01    20    6242    135
p24r4          01    28    3840     80
p24r8          01    28    3840     80
p24l10         02    28    3840     80
p24r10         02    28    3840     80
p26v12         04    28    7848    150
p29m16         14    24   11040    188
p29ma16        15    24   10460    188
p32r16         12    40    8466    128
p32vx10        10    24    9738    152
p64r32         16    84   33316    256
pc224          01    24    3204     72
pc508          02    28     256      8
pc960          01    28     256     16
pld9000        09   100   58104    200
plx448         13    24    5116     98
virtual        01   200   99999   5000
v750lcc        07    28   14394    171
v750blcc       02    28   14435    171
v750clcc       02    28   14504    171
v750cextlcc    02    28   14504    171
v750cextppklcc   02    28   14504    171
v750cppklcc    02    28   14504    171
v2500lcc       08    44   71648    416
v2500blcc      04    44   71745    416
ep324lcc       03    44   47493    394
ep900lcc       08    44   17402    240
f167lcc        23    28    3361     48
f168lcc        05    28    3553     48
f173lcc        01    28    2178     42
f179lcc        03    28    2452     43
f473lcc        03    28    1499     24
f506lcc        06    28   10680     98
f507lcc        05    28    7370     80
f42va12lcc     01    28    8994    105
f7024lcc       01    28   14088     80
f7128lcc       01    28    8488     68
f7140lcc       01    44   22103    120
g16v8hlcc      03    28    2226     64
g16v8hslcc     03    28    2226     64
g16v8hmalcc    03    28    2226     64
g16v8hmslcc    03    28    2226     64
g16z8lcc       03    28    2195     64
g16z8malcc     03    28    2195     64
g16z8mslcc     03    28    2195     64
g16z8slcc      03    28    2195     64
g20vp8lcc      01    28    2714     64
g20vp8slcc     01    28    2714     64
g20vp8malcc    01    28    2714     64
g20vp8mslcc    01    28    2714     64
g20xv10lcc     03    28    1671     40
g20xv10ilcc    03    28    1671     40
g20xv10flcc    03    28    1671     40
g20ra10lcc     02    28    3274     80
g22v10cplcc    01    28    5893    132
g6001lcc       15    28    8294     75
g6002lcc       04    28    8330     75
p1016ld4lcc    01    28    2056     64
p1016rd4lcc    01    28    2056     64
p1016et6lcc    01    28    1542     48
p1016rd8lcc    01    28    2056     64
p1020eg8lcc    04    28    3616     80
p1020ev8lcc    06    28    3616     90
p1020g8lcc     02    28    1352     32
p1020p8lcc     06    28    1352     32
p16p4clcc      01    28    2056     32
p16p8clcc      01    28    2056     64
p19r4rlcc      06    28    2443     64
p19r4tlcc      02    28    2443     64
p19r6rlcc      06    28    2443     64
p19r6tlcc      02    28    2443     64
p19l8rlcc      07    28    2443     64
p19l8tlcc      02    28    2443     64
p19r8rlcc      06    28    2443     64
p19r8tlcc      02    28    2443     64
p20rp4lcc      01    28    2568     64
p20rp4alcc     02    28    3450     86
p20rsp4lcc     01    28    2688     64
p20x4lcc       11    28    1600     40
p20xrp4lcc     02    28    3450     86
p20x4mlcc      12    28    1600     40
p20rp6lcc      01    28    2568     64
p20rp6alcc     03    28    3370     84
p20rsp6lcc     01    28    2692     64
p20xrp6lcc     01    28    3370     84
p20rp8alcc     04    28    3290     82
p20rs8lcc      19    28    3338     80
p20rsp8lcc     01    28    2696     64
p20sp8lcc      01    28    2680     64
p20x8lcc       12    28    1600     40
p20xrp8lcc     01    28    3290     82
p20x8mlcc      13    28    1600     40
p20cg10lcc     02    24    4088     92
p20g10lcc      04    28    3990     90
p20ra10lcc     15    28    3210     80
p20rp10alcc    02    28    3210     80
p20x10lcc      08    28    1600     40
p20xrp10lcc    01    28    3210     80
p20g10mlcc     03    28    3990     90
p20ra10mlcc    15    28    3210     80
p20ra10slcc    15    28    3210     80
p20rp8lcc      01    28    2568     64
p20x10mlcc     09    28    1600     40
p22p10alcc     02    28    3970     90
p22rx8lcc      05    28    3616     82
p22cv10lcc     02    28    5838    132
p22v10tlcc     01    28    5828    132
p22v10slcc     03    28    6657    132
p22vp10lcc     05    28    5838    132
p22xp10lcc     02    28    3970     90
p29m16lcc      14    28   11040    188
p29ma16lcc     15    28   10460    188
p32r16lcc      12    44    8466    128
p32vx10lcc     10    28    9738    152
p64r32pga      16    88   33316    256
pc224lcc       02    28    3204     72
</code>
</details>

<details>
<summary>Expand here for a list of devices the version of CUPL provided by Total Designer of Logical Devices, Inc. supports</summary>

<code>
wine ./cbld.exe -l -u totaldes.dl 
CBLD(PM): CUPL Device Library Management Program
Version 5.0a
Copyright (c) 1983, 1998 Logical Devices, Inc.
totaldes.dl  rev:DLIB-h

Device        Rev   Pins  Fuses  Pterms
------------  ---   ----  -----  ------
ep300          21    20    2720     74
ep320          02    20    2916     72
ep312          07    24   13713    200
ep600          14    24    6482    160
g16v8          09    20    2194     64
g16v8a         03    20    2194     64
g16v8as        02    20    2194     64
g16v8ma        08    20    2194     64
g16v8ms        11    20    2194     64
g16v8s         09    20    2194     64
g20v8          03    24    2706     64
g20v8s         02    24    2706     64
g20v8ma        03    24    2706     64
g20v8ms        03    24    2706     64
g20v8a         02    24    2706     64
g20v8as        01    24    2706     64
g22v10         01    24    5892    132
p10h8          07    20     320     16
p10l8          07    20     320     16
p10p8          07    20     328     16
p10p8v         01    20     664     32
p12h6          09    20     384     16
p12l6          07    20     384     16
p12p6          07    20     390     16
p12p6v         01    20     786     32
p12l10         07    24     480     20
p12p10         07    24     490     20
p14h4          07    20     448     16
p14l4          07    20     448     16
p14p4          07    20     452     16
p14p4v         01    20     908     32
p14l8          07    24     560     20
p14p8          09    24     568     20
p16c1          07    20     512     16
p16h2          07    20     512     16
p16l2          07    20     512     16
p16p2          07    20     514     16
p16p2v         01    20    1030     32
p16r4          11    20    2048     64
p16rp4         15    20    2056     64
p16rp4v        01    20    2072     64
p16l6          07    24     640     20
p16p6          07    24     646     20
p16r6          14    20    2048     64
p16rp6         15    20    2056     64
p16rp6v        01    20    2072     64
p16h8          08    20    2048     64
p16hd8         06    20    2048     64
p16l8          08    20    2048     64
p16ld8         06    20    2048     64
p16n8          01    20     512     16
p16p8          08    20    2056     64
p16p8h         07    20    2056     64
p16p8v         01    20    2072     64
p16r8          10    20    2048     64
p16rp8         14    20    2056     64
p16rp8v        01    20    2072     64
p18l4          07    24     720     20
p18p4          07    24     724     20
p20c1          07    24     640     16
p20l2          07    24     640     16
p20p2          07    24     642     16
p20r4          10    24    2560     64
p20rs4         21    24    3330     80
p20r6          10    24    2560     64
p20l8          08    24    2560     64
p20p8          01    24    2568     64
p20r8          09    24    2560     64
p20rs8         19    24    3338     80
p20l10         06    24    1600     40
p20rs10        19    24    3338     80
p20s10         17    24    3322     80
p22v10         17    24    5828    132
ra5p8          04    16     256     32
ra5p16         02    24     512     32
ra6p16         02    24    1024     64
ra8p4          04    16    1024    256
ra8p8          04    20    2048    256
ra9p4          04    16    2048    512
ra9p8          04    20    4096    512
ra10p4         04    18    4096   1024
ra10p8         04    24    8192   1024
ra11p4         04    18    8192   2048
ra11p8         04    24   16384   2048
ra12p4         04    20   16384   4096
ra12p8         04    24   32768   4096
ra13p8         04    24   65536   8192
virtual        01   200   99999   5000
ep312lcc       01    28   13713    200
ep600lcc       14    28    6482    160
g20v8lcc       03    28    2706     64
g20v8malcc     03    28    2706     64
g20v8mslcc     03    28    2706     64
g20v8slcc      03    28    2706     64
g20v8alcc      02    28    2706     64
g20v8aslcc     01    28    2706     64
g22v10lcc      02    28    5892    132
p12l10lcc      07    28     480     20
p12p10lcc      07    28     490     20
p12l10mlcc     08    28     480     20
p14l8lcc       07    28     560     20
p14p8lcc       09    28     568     20
p14l8mlcc      08    28     560     20
p16r4alcc      02    28    2048     64
p16l6lcc       07    28     640     20
p16p6lcc       07    28     646     20
p16l6mlcc      08    28     640     20
p16l8alcc      01    28    2048     64
p16r6alcc      01    28    2048     64
p16l8          08    20    2048     64
p16r8alcc      01    28    2048     64
p18l4lcc       07    28     720     20
p18p4lcc       07    28     724     20
p18l4mlcc      08    28     720     20
p20c1lcc       07    28     640     16
p20c1mlcc      08    28     640     16
p20l2lcc       07    28     640     16
p20r4lcc       11    28    2560     64
p20rs4lcc      21    28    3330     80
p20r4mlcc      11    28    2560     64
p20rs4mlcc     22    28    3338     80
p20r6lcc       11    28    2560     64
p20r6mlcc      11    28    2560     64
p20l8lcc       09    28    2560     64
p20p8lcc       01    28    2568     64
p20r8lcc       10    28    2560     64
p20l8mlcc      09    28    2560     64
p20l10lcc      06    28    1600     40
p20rs10lcc     19    28    3338     80
p20s10lcc      17    28    3322     80
p20rs10mlcc    20    28    3338     80
p20s10mlcc     17    28    3322     80
p22v10lcc      17    28    5828    132
v750           03    24   14394    171
v750b          02    24   14435    171
v750c          02    24   14504    171
v750cext       02    24   14504    171
v750cextppk    02    24   14504    171
v750cppk       02    24   14504    171
v2500          07    40   71648    416
v2500b         04    40   71745    416
v5000          02    68   163164   1232
f1500          01    44   15360    320
f1500t         01    44   15360    320
f1500a         01    44   15360    320
f1500at        01    44   15360    320
f1504a         01    44   15360    320
f1504aj        01    44   15360    320
f1504at        01    44   15360    320
f1504atj       01    44   15360    320
f1504plcc68    02    68   15360    320
f1504ispplcc68   02    68   15360    320
f1504plcc84    02    84   15360    320
f1504ispplcc84   02    84   15360    320
f1504qfp100    02   100   15360    320
f1504ispqfp100   02   100   15360    320
f1504tqfp100   02   100   15360    320
f1504isptqfp100   02   100   15360    320
f1508plcc68    02    68   15360    320
f1508ispplcc68   02    68   15360    320
f1508ispplcc84   02    84   15360    320
f1508plcc84    02    84   15360    320
f1508ispqfp100   02   100   15360    320
f1508qfp100    02   100   15360    320
f1508tqfp100   02   100   15360    320
f1508isptqfp100   02   100   15360    320
f1508pqfp160   01   160   15360    320
f1508isppqfp160   01   160   15360    320
ep324          05    40   47493    394
ep900          07    40   17402    240
ep1200         03    40   15146    236
ep1800         11    68   42490    480
p7b336         01    28     384     16
p7b337         01    28     768     32
p7b338         01    28     384     16
p7b339         01    28     768     32
p7c330         05    28   17082    258
p7c331         07    28   11934    216
p7c332         02    28    9902    194
p7c335         01    28   17082    258
xl78c800       01    24    6400     66
c258           01    28   32768   2048
c259           01    44   32768   2048
c330           01    28   17082    258
c331           01    28   11934    216
c332           01    28    9902    194
c335           01    28   17082    258
c336           01    28     384     16
c337           01    28     768     32
c338           01    28     384     16
c339           01    28     768     32
f100           10    28    1928     48
f103           09    28     297      9
f105           21    28    3553     48
f151           13    20     564     15
f153           11    20    1842     42
f155           14    20    2108     43
f157           15    20    2108     43
f159           19    20    2108     43
f161           14    24    1544     48
f162           07    24     165      5
f163           07    24     225      9
f167           23    24    3361     48
f168           05    24    3553     48
f173           01    24    2178     42
f179           03    24    2452     43
f253           01    20    2378     42
f273           02    24    2714     42
f405           06    28    5410     64
f415           03    28    5751     68
f473           03    24    1499     24
f501           02    52   15780    112
f502           01    68   23464    144
f506           06    24   10680     98
f507           05    24    7370     80
f529           01    20     128      8
f839           09    24    1094     32
f30k12         02    28    7424     72
f30s16         02    28    7236     71
f42va12        05    24    8994    105
f48n22         01    68    7008     73
f9800          02    20    1830     45
f7024          02    24   14088     80
f7128          02    28    8488     68
f7140          02    40   22103    120
f2552          01    68   51624    226
f2852          01    84   51624    226
f16v8          02    20    2617     72
f16v8d         02    20    2617     72
f16v8s         02    20    2617     72
f18v8z         07    20    2689     72
f18v8zd        07    20    2689     72
f18v8zs        07    20    2689     72
f20v8          03    24    3193     72
f20v8d         03    24    3193     72
f20v8s         03    24    3193     72
g16v8h         04    24    2226     64
g16v8hs        04    24    2226     64
g16v8hma       04    24    2226     64
g16v8hms       04    24    2226     64
g16v8cpms      01    20    2195     64
g16v8cp        01    20    2195     64
g16v8cpas      01    20    2195     64
g16v8cpma      01    20    2195     64
g16v8c         01    20    2194     64
g16v8cas       01    20    2194     64
g16v8cma       01    20    2194     64
g16v8cms       01    20    2194     64
g16vp8         01    20    2202     64
g16vp8s        01    20    2202     64
g16vp8ma       01    20    2202     64
g16vp8ms       01    20    2202     64
g16z8          03    24    2195     64
g16z8ma        03    24    2195     64
g16z8ms        03    24    2195     64
g16z8s         03    24    2195     64
g18v10         01    20    3540     96
g20vp8         01    24    2714     64
g20vp8s        01    24    2714     64
g20vp8ma       01    24    2714     64
g20vp8ms       01    24    2714     64
g20xv10        03    24    1671     40
g20xv10i       03    24    1671     40
g20xv10f       03    24    1671     40
g20ra10        03    24    3274     80
g22v10i        02    28    5892    132
gds22          01    28     219     11
g22v10cp       01    24    5893    132
g24v10         02    28    3492     80
g24v10ma       02    28    3942     80
g24v10ms       02    28    3942     80
g24v10s        02    28    3942     80
g26cv12        03    28    6432    120
g6001          15    24    8294     75
g6002          04    24    8330     75
mach110        04    44    6504    140
mach111        02    44    6504    140
mach120        04    68   12080    216
mach130        03    84   15552    280
mach131        01    84   15552    280
mach210        04    44   12832    280
mach211        02    44   12832    280
mach215        04    44   11936    256
mach220        05    68   23232    416
mach221        05    68   23232    416
mach231        03    84   30144    544
mach355        01   144   43740    480
mach435        03    84   54096    720
mach445        01   100   54129    720
mach446        01   100   54129    720
mach465        01   208   121185   1440
m16v8a         01    20    3136     64
m16v8as        01    20    3136     64
m16v8ac        01    20    3136     64
m16v8ar        01    20    3136     64
p1012c4        01    24    2056     32
p1016c4        01    28    2056     32
plsi1016ld4    03    24    2056     64
p1016lm4       02    24    2056     32
p1016p4        01    24    2056     32
p1016rd4       03    24    2056     64
p1016rm4       02    24    2056     32
p1016et6       01    24    1542     48
p1016ld8       03    24    2056     64
p1016p8        02    24    2056     64
p1016pe8       01    28    2056     64
p1016rd8       03    24    2056     64
p1020rp4       01    28    2568     32
p1020eg8       04    24    3616     80
p1020ev8       06    24    3616     90
p1020g8        02    24    1352     32
p1020p8        06    24    1352     32
p6l16          02    24     192     16
p8l14          02    24     224     16
p14r21         01    24    3137     86
p16p4c         01    24    2056     32
p16rsp4        01    20    2176     64
p16rsp6        01    20    2180     64
p16p8c         01    24    2056     64
p16ra8         05    20    2056     64
p16rsp8        01    20    2184     64
p16sp8         01    20    2168     64
p18cv8         02    20    2696     74
p18g8          01    20    2624     72
p18n8          01    20     304      8
p18p8          03    20    2600     72
p18u8          02    20    2688     72
p18v8          01    20    2752     74
p19r4r         06    24    2443     64
p19r4t         02    24    2443     64
p19r6r         06    24    2443     64
p19r6t         02    24    2443     64
p19l8r         07    24    2443     64
p19l8t         02    24    2443     64
p19r8r         06    24    2443     64
p19r8t         02    24    2443     64
p20rp4         01    24    2568     64
p20rp4a        02    24    3450     86
p20rsp4        01    24    2688     64
p20x4          11    24    1600     40
p20xrp4        02    24    3450     86
p20rp6         01    24    2568     64
p20rp6a        03    24    3370     84
p20rsp6        01    24    2692     64
p20xrp6        01    24    3370     84
p20rp8         01    24    2568     64
p20rp8a        04    24    3290     82
p20rsp8        01    24    2696     64
p20sp8         01    24    2680     64
p20x8          12    24    1600     40
p20xrp8        01    24    3290     82
p20cg10        02    24    4088     92
p20g10         03    24    3990     90
p20ra10        15    24    3210     80
p20rp10a       02    24    3210     80
p20x10         08    24    1600     40
p20xrp10       01    24    3210     80
p22cv10z       02    24    5873    132
p22ip6         01    24    3294     72
p22p10a        02    24    3970     90
p22rx8         05    24    3616     82
p22v10s        03    24    6657    132
p22vp10        05    24    5838    132
p22xp10        02    24    3970     90
p23s8          06    20    6234    135
p23sv8         01    20    6242    135
p24r4          01    28    3840     80
p24r8          01    28    3840     80
p24l10         02    28    3840     80
p24r10         02    28    3840     80
p26v12         04    28    7848    150
p29m16         14    24   11040    188
p29ma16        15    24   10460    188
p32r16         12    40    8466    128
p32vx10        10    24    9738    152
p64r32         16    84   33316    256
pc224          01    24    3204     72
pc508          02    28     256      8
pc960          01    28     256     16
pld9000        09   100   58104    200
plx448         13    24    5116     98
virtual        01   200   99999   5000
ep324lcc       03    44   47493    394
ep900lcc       08    44   17402    240
f167lcc        23    28    3361     48
f168lcc        05    28    3553     48
f173lcc        01    28    2178     42
f179lcc        03    28    2452     43
f473lcc        03    28    1499     24
f506lcc        06    28   10680     98
f507lcc        05    28    7370     80
f42va12lcc     01    28    8994    105
f7024lcc       01    28   14088     80
f7128lcc       01    28    8488     68
f7140lcc       01    44   22103    120
g16v8hlcc      03    28    2226     64
g16v8hslcc     03    28    2226     64
g16v8hmalcc    03    28    2226     64
g16v8hmslcc    03    28    2226     64
g16z8lcc       03    28    2195     64
g16z8malcc     03    28    2195     64
g16z8mslcc     03    28    2195     64
g16z8slcc      03    28    2195     64
g20vp8lcc      01    28    2714     64
g20vp8slcc     01    28    2714     64
g20vp8malcc    01    28    2714     64
g20vp8mslcc    01    28    2714     64
g20xv10lcc     03    28    1671     40
g20xv10ilcc    03    28    1671     40
g20xv10flcc    03    28    1671     40
g20ra10lcc     02    28    3274     80
g22v10cplcc    01    28    5893    132
g6001lcc       15    28    8294     75
g6002lcc       04    28    8330     75
p1016ld4lcc    01    28    2056     64
p1016rd4lcc    01    28    2056     64
p1016et6lcc    01    28    1542     48
p1016rd8lcc    01    28    2056     64
p1020eg8lcc    04    28    3616     80
p1020ev8lcc    06    28    3616     90
p1020g8lcc     02    28    1352     32
p1020p8lcc     06    28    1352     32
p16p4clcc      01    28    2056     32
p16p8clcc      01    28    2056     64
p19r4rlcc      06    28    2443     64
p19r4tlcc      02    28    2443     64
p19r6rlcc      06    28    2443     64
p19r6tlcc      02    28    2443     64
p19l8rlcc      07    28    2443     64
p19l8tlcc      02    28    2443     64
p19r8rlcc      06    28    2443     64
p19r8tlcc      02    28    2443     64
p20rp4lcc      01    28    2568     64
p20rp4alcc     02    28    3450     86
p20rsp4lcc     01    28    2688     64
p20x4lcc       11    28    1600     40
p20xrp4lcc     02    28    3450     86
p20x4mlcc      12    28    1600     40
p20rp6lcc      01    28    2568     64
p20rp6alcc     03    28    3370     84
p20rsp6lcc     01    28    2692     64
p20xrp6lcc     01    28    3370     84
p20rp8alcc     04    28    3290     82
p20rs8lcc      19    28    3338     80
p20rsp8lcc     01    28    2696     64
p20sp8lcc      01    28    2680     64
p20x8lcc       12    28    1600     40
p20xrp8lcc     01    28    3290     82
p20x8mlcc      13    28    1600     40
p20cg10lcc     02    24    4088     92
p20g10lcc      04    28    3990     90
p20ra10lcc     15    28    3210     80
p20rp10alcc    02    28    3210     80
p20x10lcc      08    28    1600     40
p20xrp10lcc    01    28    3210     80
p20g10mlcc     03    28    3990     90
p20ra10mlcc    15    28    3210     80
p20ra10slcc    15    28    3210     80
p20rp8lcc      01    28    2568     64
p20x10mlcc     09    28    1600     40
p22p10alcc     02    28    3970     90
p22rx8lcc      05    28    3616     82
p22cv10lcc     02    28    5838    132
p22v10tlcc     01    28    5828    132
p22v10slcc     03    28    6657    132
p22vp10lcc     05    28    5838    132
p22xp10lcc     02    28    3970     90
p29m16lcc      14    28   11040    188
p29ma16lcc     15    28   10460    188
p32r16lcc      12    44    8466    128
p32vx10lcc     10    28    9738    152
p64r32pga      16    88   33316    256
pc224lcc       02    28    3204     72
v750lcc        07    28   14394    171
v750blcc       02    28   14435    171
v750clcc       02    28   14504    171
v750cextlcc    02    28   14504    171
v750cextppklcc   02    28   14504    171
v750cppklcc    02    28   14504    171
v2500lcc       08    44   71648    416
v2500blcc      04    44   71745    416
max5032        03   200   99999   5000
epm7032        04   200   99999   5000
c371           02    44    6504    140
c372           02    44   131224    140
c373           02    84   15552    280
c373t          02   100   15552    280
c374           02    84   15552    280
c374t          02   100   15552    280
c375           02   160   15552    280
nfx740_44      01    44   15852    512
nfx740_68      01    68   15852    512
nfx780_84      01    84   31704   1024
kufx780_132    01   132   31704   1024
ispl1048_80lq120   02   120   57600    960
ispl1048_70lq120   02   120   57600    960
ispl1048_50lq120   02   120   57600    960
ispl1048_50lq20i   02   120   57600    960
ispl1048c_70lq28   01   128   57600    960
ispl1048c_50lg33   01   128   57600    960
ispl1048c_50lq28   01   128   57600    960
ispl1048c_50lq8i   01   128   57600    960
ispl1032_90lj84   02    84   34560    640
ispl1032_90lt100   01   100   34560    640
ispl1032_80lj84   02    84   34560    640
ispl1032_80lt100   01   100   34560    640
ispl1032_60lg84   02    84   34560    640
ispl1032_60lj84   02    84   34560    640
ispl1032_60lj84i   02    84   34560    640
ispl1032_60lt100   01   100   34560    640
ispl1024_90lj68   02    68   24480    480
ispl1024_80lj68   02    68   24480    480
ispl1024_60lh68   02    68   24480    480
ispl1024_60lj68   02    68   24480    480
ispl1024_60lj68i   02    68   24480    480
ispl1016_110lj44   02    44   15360    320
ispl1016_90lj44   02    44   15360    320
ispl1016_90lt44   02    44   15360    320
ispl1016_80lj44   02    44   15360    320
ispl1016_80lt44   02    44   15360    320
ispl1016_60lh44   02    44   15360    320
ispl1016_60lj44   02    44   15360    320
ispl1016_60lj44i   02    44   15360    320
ispl1016_60lt44   02    44   15360    320
plsi1048_80lq120   02   120   57600    960
plsi1048_70lq120   02   120   57600    960
plsi1048_50lq120   02   120   57600    960
plsi1048_50lq12i   02   120   57600    960
plsi1048c_70lq28   01   128   57600    960
plsi1048c_50lg33   01   128   57600    960
plsi1048c_50lq28   01   128   57600    960
plsi1048c_50lq8i   01   128   57600    960
plsi1032_90lj84   02    84   34560    640
plsi1032_90lt100   01   100   34560    640
plsi1032_80lj84   02    84   34560    640
plsi1032_80lt100   01   100   34560    640
plsi1032_60lg84   02    84   34560    640
plsi1032_60lj84   02    84   34560    640
plsi1032_60lj84i   02    84   34560    640
plsi1032_60lt100   01   100   34560    640
plsi1024_90lj68   02    68   24480    480
plsi1024_80lj68   02    68   24480    480
plsi1024_60lh68   02    68   24480    480
plsi1024_60lj68   02    68   24480    480
plsi1024_60lj68i   02    68   24480    480
plsi1016_110lj44   02    44   15360    320
plsi1016_90lj44   02    44   15360    320
plsi1016_90lt44   02    44   15360    320
plsi1016_80lj44   02    44   15360    320
plsi1016_80lt44   02    44   15360    320
plsi1016_60lh44   02    44   15360    320
plsi1016_60lj44   02    44   15360    320
plsi1016_60lj44i   02    44   15360    320
plsi1016_60lt44   02    44   15360    320
ispl2128_10lm160   01   160   99999   5000
ispl2128_80lm160   01   160   99999   5000
ispl2096_12lq128   01   128   57600    960
ispl2096_10lq128   01   128   57600    960
ispl2096_80lq128   01   128   57600    960
ispl2064_125lj84   01    84   34560    640
ispl2064_12lt100   01   100   34560    640
ispl2064_100lj84   01    84   34560    640
ispl2064_10lt100   01   100   34560    640
ispl2064_80lj84   01    84   34560    640
ispl2064_80lt100   01   100   34560    640
ispl2032_150lj44   01    44   99999    160
ispl2032_150lt44   02    44   15360    320
ispl2032_135lj44   01    44   99999    160
ispl2032_135lt44   02    44   15360    320
ispl2032_110lj44   01    44   99999    160
ispl2032_110lt44   02    44   15360    320
ispl2032_80lj44   01    44   99999    160
ispl2032_80lt44   02    44   15360    320
plsi2128_10lm160   01   160   99999   5000
plsi2128_80lm160   01   160   99999   5000
plsi2096_12lq128   01   128   57600    960
plsi2096_10lq128   01   128   57600    960
plsi2096_80lq128   01   128   57600    960
plsi2064_125lj84   01    84   34560    640
plsi2064_100lj84   01    84   34560    640
plsi2064_80lj84   01    84   34560    640
plsi2032_150lj44   01    44   99999    160
plsi2032_150lt44   02    44   15360    320
plsi2032_135lj44   01    44   99999    160
plsi2032_135lt44   02    44   15360    320
plsi2032_110lj44   01    44   99999    160
plsi2032_110lt44   02    44   15360    320
plsi2032_80lj44   01    44   99999    160
plsi2032_80lt44   02    44   15360    320
ispl3256_70lg167   01   167   99999   5000
ispl3256_50lg167   01   167   99999   5000
ispl3256_70lm160   01   160   99999   5000
ispl3256_50lm160   01   160   99999   5000
plsi3256_70lg167   01   167   99999   5000
plsi3256_50lg167   01   167   99999   5000
plsi3256_70lm160   01   160   99999   5000
plsi3256_50lm160   01   160   99999   5000
plsi           01   255   99999   5000
isplsi         01   255   99999   5000
mapl128        01    28   14833    132
mapl144        01    44   14721    132
pz3032_plcc44   01    44   15360    320
pz3032_qfp44   01    44   15360    320
pz3064_plcc44   01    44   15360    320
pz3064_qfp44   01    44   15360    320
pz3064_plcc68   01    68   15360    320
pz3064_plcc84   01    84   15360    320
pz3064_qfp100   01   100   15360    320
pz3128_plcc84   01    84   15360    320
pz3128_qfp100   01   100   15360    320
pz3128_lqf128   01   128   15360    320
pz3128_qfp160   01   160   15360    320
pz5032_plcc44   01    44   15360    320
pz5032_qfp44   01    44   15360    320
pz5064_plcc44   01    44   15360    320
pz5064_qfp44   01    44   15360    320
pz5064_plcc68   01    68   15360    320
pz5064_plcc84   01    84   15360    320
pz5064_qfp100   01   100   15360    320
pz5128_plcc84   01    84   15360    320
pz5128_qfp100   01   100   15360    320
pz5128_lqf128   01   128   15360    320
pz5128_pqf160   01   160   15360    320
ga2000         01   200   99999   5000
ga3000         01   200   99999   5000
ga4000         01   200   99999   5000
xc73108pq100   03   100   99999   5000
xc7272apg84    03    84   99999   5000
xc73108bg225   02   225   99999   5000
xc73108pg144   02   144   99999   5000
xc7318pc44     03    44   99999   5000
xc73108pq160   03   160   99999   5000
xc73108pc84    03    84   99999   5000
xc7272pg84     03    84   99999   5000
xc7272pc68     03    68   99999   5000
xc7272pc84     03    84   99999   5000
xc7272apc68    03    68   99999   5000
xc7372pc68     03    68   99999   5000
xc7372pc84     03    84   99999   5000
xc7318pq44     03    44   99999   5000
xc7354pc68     03    68   99999   5000
xc7354pc44     03    44   99999   5000
xc7336pc44     03    44   99999   5000
xc7336pq44     03    44   99999   5000
xc7236pc44     03    44   99999   5000
xc7236apc44    03    44   99999   5000
xc7272apc84    03    84   99999   5000
xc95108pq160   02   160   99999   5000
xc95108pc84    02    84   99999   5000
xc95108pq100   02   100   99999   5000
xc95144pq100   02   100   99999   5000
xc95144pq160   02   160   99999   5000
xc95180pq160   02   160   99999   5000
xc95180hq208   02   208   99999   5000
xc95216pq160   02   160   99999   5000
xc95216hq208   02   208   99999   5000
xc95288hq208   02   208   99999   5000
xc9536pc44     02    44   99999   5000
xc9536vq44     02    44   99999   5000
xc9572pc84     02    84   99999   5000
xc9572pq100    02   100   99999   5000
p8x12          01   200   99999   5000
p8x12a         01   200   99999   5000
virtual        01   200   99999   5000
</code>
</details>

If one is trying to utilize the ATF150X devices, using the appropriate fitter is required.
* ![ATF15xx Family Device Fitter User's Manual](vendor-docs/fitter.pdf)

<details>
<summary>Expand for command line options for the latest known version of the ATF1502.EXE fitter.</summary>
<code>Atmel ATF1502 Fitter Version 1918 (3-21-07)
Copyright 1999,2000 Atmel Corporation
 Usage: FIT1502.EXE [-i] input_file[.tt2] {options}
 Options:
   -help
   -o output_file_name (for *.tt3 and *.jed)
   -device package_type (PLCC44/TQFP44)
   -tech tech_name (ATF1502AS/ATF1502ASV/ATF1502BE)
   -module module_name
   -preassign TRY|keep|ignore (pin preassignment options)
   -silent (no message on screen)
   -h2 (advanced help option)
   -has (advanced help option for AS)
   -hbe (advanced help option for BE)
</code>

Advanced help options:
<code>
Atmel ATF1502 Fitter Version 1918 (3-21-07)
Copyright 1999,2000 Atmel Corporation
   -strategy c [command file name]
   -strategy ifmt (input file format) [TT | edif]
   -strategy lib (library file name for edif input)
   -strategy open_collector = [   OFF |   on  | = pin_name1 pin_name2...]
   -strategy JTAG = [   off |   ON ]
   -strategy pd1 [   OFF |   on ] (power down 1)
   -strategy pd2 [   OFF |   on ] (power down 2)
   -strategy TDI_pullup = [   OFF |   on ]
   -strategy TMS_pullup = [   OFF |   on ]
   -strategy DEBUG = [   on |   OFF ]
   -strategy output_fast [on | OFF | = pin_name1 pin_name2...]
   -strategy pin_keep [ off | = pin_name1 pin_name2...]
   -strategy ues [value ] (2 ASCII characters)
   -strategy security [ OFF | on ]
   -strategy tPD = [ 5 | 7 ]
   -strategy voltage_level_A [ 1.8 | 2.5 | 3.3]
   -strategy voltage_level_B [ 1.8 | 2.5 | 3.3]
   -strategy fast_inlatch [ OFF | on | = pin_name1 pin_name2...]
   -strategy schmitt_trigger [ OFF | = pin_name1 pin_name2...]
   -strategy pull_up [ OFF | = pin_name1 pin_name2...]
   -strategy unused_To_PinKeeper [ off | ON ]
   -strategy pull_up_unused [ OFF | on]
   -strategy unused_To_Ground [ OFF | on]
   -strategy pull_down [ OFF | = dedicated_pin1 dedicated_pin2...]
   -strategy Latch_Synthesis [ON | off ]
   -strategy Optimize [ON | off]
   -strategy Cascade_Logic [ON | off |= pin_name1 ..pin_nameN]
   -strategy Foldback_Logic [ON | off |= node_name1 ..node_nameN]
   -strategy Soft_Buffer [on | OFF |= node_name1 ..node_nameN]
   -strategy XOR_Synthesis [on | OFF |= pin_name1 ..pin_nameN]
   -strategy Push_Gate [on | OFF]
   -strategy Verilog_sim [sdf | Verilog | OFF]
   -strategy Vhdl_sim [sdf | vhdl | OFF]
   -strategy Out_Edif [on | OFF]
   -strategy Global_Fold [node_name1 ..node_nameN]
   -strategy Global_OE [node_name1 ..node_nameN]
   -strategy OE_node [node_Number1..node_NumberN]
   -strategy logic_doubling [on | OFF]
   -strategy twoclock [clockname]
   -strategy pinfile
</code>
</details>


Other people's workflows:
* https://github.com/willie68/WCPLD
* https://github.com/Manawyrm/PAL-GAL-CI

## Absurd approach: Fusemaps by hand (16V8 / 22V10)
One can literally create a fusemap by hand for a PLD.
* See this <a href="https://blog.frankdecaire.com/2017/01/22/generic-array-logic-devices/">blog post</a> by Frank DeCaire, where he documents his journey of doing so.

While not the easiest approach, just as one can write G-Code in notepad or Assembly code in a hex editor, manually creating a fusemap is technically possible. This assumes that you have a datasheet for your device which has a description of the fusemap and the details of how the macrocells work. With this in hand, one could write a JEDEC file with the desired functionality and a text editor. This would be non-trivial and error-prone (if double-negatives confuse you, this is even more exciting), but it demonstrates that such a thing could be done, at least with the older PLDs (16V8, 22V10), and even with the ATF750 (some datasheets actually had the fusemap for this part).

It is worth noting that the fusemap for the ATF150x parts has been recently documented in <a href="https://github.com/whitequark/prjbureau">prjbureau</a>. Given the complexity of these devices over PLDs, writing a fusemap by hand for these parts would probably be a bad idea.

## Other languages: ABEL, PALASM
These will only be covered very briefly:
* ABEL: "Advanced Boolean Expression Language" was created in 1983 by Data I/O Corporation.
* PALASM: Introduced by Monolithic Memories, Inc. (MMI) in the 1980's
  * A modern version of this is called <a href="https://github.com/daveho/GALasm">GALASM</a> which is a continuation of something called GALer. This might be worth considering if you are happy with just PLDs.

## Atmel Prochip (Not Free, Verilog/VHDL support for ATF150x)
![PDF: Example Verilog Design flows with using ProChip 5.0.1](vendor-docs/CPLD_Mentor_Verilog_tutorial[1].pdf)<br />
Atmel Prochip is not free, however, you can <a href="https://ww1.microchip.com/downloads/en/DeviceDoc/ProChip5.0.1.zip">download it from here</a>, and may be able to <a href="https://www.microchip.com/prochiplicensing/#/">request a trial license from Microchip</a>. This workflow supports Verilog/VHDL, which is great if one wants to move away from CUPL entirely and can afford to purchase a license.

Prochip should be downloaded regardless because there are newer fitters for the ATF150x devices that can be extracted from this installation, and these fitters are required in every other approach mentioned here. The newer versions of the fitters should mention version 1918 (3-21-07) when invoked from a command line. (The fitters that come with WinCUPL are old and should be replaced with the ones from this package).

## Quartus (Free, Verilog, VHDL, Schematic Capture). Indirect support for ATF150x. Linux or Windows.
* It turns out that the Altera (Now Intel) Quartus II 13.0sp1 Web Edition can be used to produce a .POF file targeting various CPLD chips made by Altera in the MAX EPM3K/EPM7K series, which can be converted to target an ATF150x CPLD.
  * <a href="https://www.intel.com/content/www/us/en/software-kit/711791/intel-quartus-ii-web-edition-design-software-version-13-0sp1-for-windows.html?">Intel® Quartus&reg; II Web Edition Design Software Version 13.0sp1 for Windows</a>
  * <a href="https://www.intel.com/content/www/us/en/software-kit/711790/intel-quartus-ii-web-edition-design-software-version-13-0sp1-for-linux.html?">Intel® Quartus&reg; II Web Edition Design Software Version 13.0sp1 for Linux</a>
  * When installing, you only need "MAX II/V, MAX3000/7000" under device support. Unchecking the other devices can save ~2GB.
  * Finally, you may have trouble running it as libpng12.so.0 may be required. See <a href="https://silverdrs.wordpress.com/2020/11/24/running-older-altera-quartus-on-modern-64bit-gnu-linux/">here</a>
  * You can move the .desktop shortcut into ~/.local/share/applications/Quartus II 13.0sp1 (64-bit) Web Edition.desktop
    * I found I had to set Terminal=true for it to work.
* The resulting .POF file can be converted using a utility called <a href="http://ww1.microchip.com/downloads/archive/pof2jed.zip">POF2JED</a> from Atmel (Now Microchip). This is further detailed in <a href="http://ww1.microchip.com/downloads/en/AppNotes/DOC0916.PDF">this application note.
* Important!: Newer versions of Quartus will not work. v13.0sp1 is the last version that had support for the MAX EPM3K/EPM7K chips. Support for these chips has been removed from newer versions of Quartus. You MUST use the old version.

## Digital (free, use schematics instead of logic equations / programming)
"Digital is an easy-to-use digital logic designer and circuit simulator designed for educational purposes."

This is an interesting option as one can create a schematic and have a .JED file generated for a GAL16V8 or GAL22V10. If one provides the fitters to Digital, it can produce .JED files for the ATF150x series as well. Note that this is more of an educational tool for learning about logic. You may have trouble if you expect fullly featured support of these devices (Tri-state pins, Bi-directional IO, etc.)
* https://github.com/hneemann/Digital
If this appeals to you, you might be interested in similar software (though no support for the Atmel parts):
* <a href="http://www.cburch.com/logisim/">Logisim</a>
* <a href="https://github.com/logisim-evolution/logisim-evolution">Logisim Evolution</a>

## Yosys (Open Source with Atmel Fitters for ATF150x, experimental)
In theory, one can use Yosys Open SYnthesis Suite (Yosys) with the help of the Atmel Fitters a specific CPLD and a techmap to produce .JED files. This is a bit more experimental, but some have managed to make this work. This allows an almost entirely open-source workflow using Verilog, and probably <a href="https://icestudio.io/">Icestudio</a> if one prefers schematic capture as well. A good place to start would be using the <a href="https://github.com/YosysHQ/oss-cad-suite-build">OSS CAD Suite</a> to get the big parts of the suite set up. After that, there are two approaches to making this work:
* https://github.com/whitequark/prjbureau
  * prjbureau demonstrates going from RTLIL to a .JED file
* https://github.com/hoglet67/atf15xx_yosys/
  * This example goes from plain old verilog into a .JED file by implementing a techmap.
* https://github.com/michaelhunsberger/JsonToCupl/
  * This is an example of how to use Yosys to generate CUPL code.
  * Potentially interesting as one could use this to generate a .PLD even for the simpiler 16V8 or 22V10 devices
  * http://forum.6502.org/viewtopic.php?f=10&t=7601

Finally, since yosys is extremely complex, a section on understanding the basics is in order especially from the context of these devices. For the moment, however, others have written guides for different parts:
* https://github.com/Ravenslofty/yosys-cookbook
  * https://github.com/Ravenslofty/74xx-liberty/
  * https://github.com/Ravenslofty/74xx-liberty/tree/master/kicad

# Programming / Burning and Device Information
There are a few choices on how the part can actually be programmed depending on whether it supports JTAG.

![A detailed overview of ways to program a given device.](PROGRAMMING.md)


# Reversing a JED file back into logic equations
Finally, if one is able to read a .JED out of a device, this can sometimes be reversed back into equations. These devices all have security fuses, however, which can disable any ability to read out the device. Given a .JED file, the following approaches can be taken to arrive at the equations:
* By hand, comparing the .JED file to the fusemap / macrocells in the datasheet. See this <a href="https://blog.frankdecaire.com/2017/01/22/generic-array-logic-devices/">blog post</a> by Frank DeCaire.
* `JED2EQN.EXE` - A DOS utility floating around on the internet.
* `jedutil` - MAME can be compiled with a utility called jedutil which does something similar. Sometimes it is broken out into a seperate package "mame-tools"
* Brute Force - A PLD that is strictly combonatorial can be read out as though it is an EPROM by stepping through all combinations of possible inputs. Once state/registers are involved, this becomes much more challenging.

# Simulation
CSIM.EXE can be fed test vectors and be used to simulate the behavior of a particular chip, or even a virtual device. The following things are required to do this successfully:
* You must create an .SI file containing the desired test vectors.
* Provide an "[Absolute file](abs-decode/)". The .ABS file is generated by CUPL.EXE when the -a flag is passed to it. This generates a binary file based on the logic equations from the source CUPL .PLD file.
CSIM will then generate a .SO output file, and optionally append test vectors to an existing .JED file for testing purposes.
<details>
<summary>Expand here for command-line options to CSIM.EXE</summary>
<code>csim [-flags] [library] [device] source
where
-flags is the following set of simulator options:
-l create listing file.
-j append test vectors to JEDEC file.
-n use source filename for JEDEC file.
-v display simulation results to terminal.
-u use specified library for simulation.
library is the library name and path name if the -u flag is being used to specify a
library other than the default library.
device must be the same device mnemonic as was used in the CUPL compilation.
Specifying the device is optional; if a device is not specified, CSIM uses the device
CUPL compiled (contained in the .ABS file).
source is the user-created ASCII test specification file (filename.SI). The
extension .SI is assumed for the source file and may be omitted when giving the
CSIM command.</code>
</details>


Creating a .SI file:<br />
* An .SI file should have the same header information as the original .PLD source file. If not, this will generate warnings.
* Comments begin with a /* and end with a */
* An .SI file can have the following keywords/statements: ORDER, BASE, and VECTORS
  * The ORDER keyword is used to list the variable / inputs and outputs to be used in the simulation table, and to define how they are displayed. Typically, the variable names are the same as those in the corresponding CUPL logic description file.
  * The BASE keyword specifies a number base. Hexadecimal is the default if unspecified.
  * The VECTORS keyboard specifies a list of test vectors (signals that are applied and expected outputs).
* If you simply want to see what will happen on the outputs rather than setting a pre-determined expected value, set the outputs to *

<details>
<summary>Expand for a list of valid Test Values used in a test vector</summary>
<code>0 Drive input LO (0 volts) (negate active-HI input)
1 Drive input HI (+5 volts) (assert active-HI input)
C Drive (clock) input LO, HI, LO
K Drive (clock) input HI, LO, HI
L Test output LO (0 volts) (active-HI output negated)
H Test output HI (+5 volts) (active-HI output asserted)
Z Test output for high impedance
X Input HI or LO, output HI or LO Note: Not all device programmers treat X on inputs the same; some put it to 0, some allow input to be pulled to 1, and some leave it at the previous value.
N Output not tested
P Preload internal registers (value is applied to !Q output)
* Outputs only -simulator determines test value and substitutes in vector
' ' Enclose input values to be expanded to a specified BASE (octal, decimal, or hex). Valid values are 0-F and X.
“ ” Enclose output values to be expanded to a specified BASE (octal, decimal, or hex.) Valid values are 0-F, H, L, Z, and X.
</code>
</details>

# References
[^1]: From the Readme.txt of fit5_0.zip (an older version of the Atmel Fitters)

# Acknowledgements
This repository is merely a bunch of tips, tricks, helper scripts and documentation. The real work comes from:
* Whitequark for putting together <a href="https://github.com/whitequark/prjbureau">Prjbureau</a>, which documents the fusemap for these devices, provides an ability to go from a .JED file to an .SVF, documentation and more.
* Yosys
* hoglet67 for putting together <a href="https://github.com/hoglet67/atf15xx_yosys">atf15xx_yosys</a>, which shows a workflow using yosys and provides a techmap.
* Countless other tips, tools, contributions and from all over the web and plenty of trial-and-error working around the quirks of WinCUPL and the fitters themselves.
