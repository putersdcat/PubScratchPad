# Mentec M1 PDP-11 CPU: Floating Point, Firmware, and Configuration

**Board in question:** Mentec M1 (ASIC re-implementation of the Mentec M11 Q-Bus PDP-11 CPU)  
**Date compiled:** 2026-09-14  
**Purpose:** Collected primary and secondary sources on how floating point is implemented, when it is available, how the board is configured, and what firmware actually exists.

---

## 1. Short answer

The large Intel part on the upper-left of an M1 is an **i960**, not an i970. Floating point on M11/M1 is **not** a DEC FP11 coprocessor chip. It is **emulated by the Intel i960** (IEEE format internally) together with ODT and microcode load.

FP is a **microcode revision** feature:

- Available only with **microcode 2.0 or later** (documented for M11; M1 inherits the same design).
- Rev **1.x** (field example: V 1.15) has **no FPP**.
- Configuration of the *board* (boot devices, serial, LTC, etc.) is done from the on-board FLASH via the firmware **`SETUP`** command. That does **not** install missing FP microcode.

There is **no published Mentec source tree** for the i960 FP emulator or the ASIC microcode. The closest public firmware artifacts are hobbyist ROM dumps of **M11** boards, not a drop-in M1 FLASH image.

---

## 2. Architecture lineage

| Board | Era / implementation | FP hardware | Notes |
|---|---|---|---|
| M100 | Last J-11 based Mentec CPU | Optional discrete FPU | J-11 at 19.66 MHz, 1–4 MB, 4 serial ports |
| M11 | Clean-room microcoded PDP-11 | Intel **i960**: load microcode, IEEE FP, ODT | Two TI 8832 ALUs + TI 8818 sequencer; Xilinx emulates 4× DLART |
| M1 | ASIC re-implementation of M11 ISA | Same helper-processor model (i960 still present) | Atmel 0.85 µm 5 V ASIC; still fully microcoded |

Sources: Wikipedia *Mentec* page (M100 / M11 / M1 sections) and Mentec M11 brochure.

The M11 was designed in VHDL, simulated in Mentor Graphics QuickSim II, and ran patched DEC PDP-11/23 diagnostics in simulation before silicon.

---

## 3. Primary documents

### 3.1 Mentec M11 brochure (the design the M1 copies)

Archive copies:

- https://web.archive.org/web/19970725211216/http://www.mentec.com/PDP/m11brochu.htm
- https://web.archive.org/web/19990819134502/http://mentec.com/PDP/m11brochu.htm
- Later mirror of specs page used in 2014 discussions: http://web.archive.org/web/20060324070926/http://www.mentec-inc.com/m11specs.htm

Key statement (Processor section):

> The M11 is a high performance PDP-11 similar in functionality to previous Mentec PDP11 CPU's.  
> PDP-11 instructions are implimented by a microprogrammed subsystem based on the TI 8832 ALU and the TI 8818A microsequencer.  
> **Console ODT and floating point instruction emulation are performed by the i960 processor block.**  
> **Floating point emulation is available only with microcode revisions 2.0 or later.**  
> It should be noted that internally **IEEE format** floating point is used for emulation.

That is the authoritative “when does FP exist?” sentence.

### 3.2 Mentec SBC M1 User Guide, Version 1.0 (1998)

- File: `SBC_M1_User_Manual_1998.pdf`
- Bitsavers: https://bitsavers.org/pdf/mentec/SBC_M1_User_Manual_1998.pdf  
- Also: http://bitsavers.informatik.uni-stuttgart.de/pdf/mentec/SBC_M1_User_Manual_1998.pdf
- Document IDs in the scan: **REF: MIUG**, **ISSUE: REV VI.O.**, copyright Mentec Limited 1998  
- Mentec House, Dun Laoghaire Industrial Estate, Co. Dublin, Ireland

This is the configuration manual for the M1 itself.

**Processor (Ch. 1.2)**

> The MI is a high performance PDP-11 similar in functionality to previous Mentec PDP-11 CPU's.  
> PDP-11 instructions are implemented in "Microcode" a micro programmed subsystem based on an ASIC implementation.  
> The Console ODT function is also performed in the MI's Microcode.

Note the wording change vs the M11 brochure: ODT is described as being in the M1 microcode. The i960 is still on the board (see LEDs below).

**Firmware storage (Ch. 5.1)**

> The bootstrap and diagnostic firmware is contained in one FLASH EPROM on the SBC MI.  
> The EPROM is used both by the power-up circuitry and the boot/diagnostic software to retain setup information.

Module also has 32 KB FLASH called out in the product overview, 1–4 MB SRAM, four DLV11-J compatible serial lines, and a software-selectable 50 / 60 / 800 Hz or BEVENT LTC.

**Dialogue commands (Ch. 5.2 / 5.5)**

Firmware runs in:

- Automatic boot mode
- Dialogue mode

Dialogue is entered when the board is unconfigured, autoboot fails, or **Ctrl-C** is hit during diagnostics / autoboot.

Commands:

| Command | Function |
|---|---|
| `BOOT` | Boot from a specific device |
| `HELP` | Command summary |
| `LIST` | List provided bootstraps |
| `MAP` | Address-space map |
| `SETUP` | Configure the SBC M1 |
| `TEST` | Continuous test mode |

**`SETUP` sub-functions (Ch. 5.5.5)**

1. Modify Hardware Setup  
2. Modify Software Setup  
3. Save Modified Setup  
4. Initialise To Factory Setup  
5. Configure Autoboot List  
6. Configure Device Translations  
7. Exit Setup  

Hardware setup covers serial lines, baud, console, LTC, BootPROM, power-up mode, etc. Changes are written back to FLASH. This is board *configuration*, not a microcode upgrade path.

FLASH mapping uses a Page Control Register at **17777520** octal (128 windows × 256 bytes).

**LEDs that matter for FP / i960 (Ch. 1.9 and 3.5)**

| LED | Label | Color | Meaning |
|---|---|---|---|
| 1–4 | WCS Load 1–4 | Red | Writable control store / microcode load progress |
| 5 | **FP Indicator** | Red | Floating-point path indicator |
| 6 | 12V | Green | ±12 V present |
| 7 | BDCOK | Green | DC OK |
| 8 | Run | Green | CPU running |
| 9 | ODT | Red | Console ODT |
| 10 | Console | Red | Console activity |
| 11 | Memory | Red | Memory diagnostic / fault |
| 12 | CPU | Red | CPU diagnostic / fault |
| 13 | **i960 Reset** | Red | i960 helper CPU reset |

Power-up on M1 differs from M11: instead of the banner `Loading Microcode v1.x 1 2 3 4 5`, six LEDs extinguish in sequence. LED 13 extinguishes, then LED 5, then LEDs 1–4 light together and go out one by one.

**Architecture notes that mention FP (Ch. 6)**

Chapter 6 documents PDP-11 programmer-visible state, including floating-point registers that “do not have addresses” and are referenced by special instructions. Appendix B lists FP opcodes (`ADDF`, `ADDD`, `ABSF`, `ABSD`, …). That is the ISA surface; the execution engine for those ops is the i960 path described in the M11 brochure.

Related DEC documents cited by Mentec in the M1 preface: DCJ11 User Guide, Micro PDP-11 Handbook, Micro PDP-11 Interface Handbook.

### 3.3 Mentec Development Project Report (M11 → M1 ASIC)

Cited by Wikipedia for the M1 ASIC details:

- https://web.archive.org/web/20160412201307/https://www.fuse-network.com/fuse/demonstration/30/24675/24675.pdf

States the M1 is an ASIC re-implementation of the M11 PDP-11 instruction set, still fully microcoded, using an Atmel 0.85 µm ASIC for 5 V operation.

### 3.4 Wikipedia — Mentec

- https://en.wikipedia.org/wiki/Mentec

Useful for company history (1978 Ireland DEC OEM → 1994 PDP-11 OS transfer from DEC → Mentec Inc. bankrupt 2006, Mentec Ltd. acquired by Calyx, OS rights later with XX2247 LLC) and the condensed M100 / M11 / M1 technical paragraphs.

---

## 4. Field reports and firmware artifacts

### 4.1 VCFED — Mentec M11 (2022)

Thread: https://forum.vcfed.org/index.php?threads/mentec-m11.1240709/

**Hunta** on a working M11:

- Banner: `M11 Microcode Rev. V 1.15` then `Loading microcode - 1 2 3 4 5`
- Bootstrap: `M1000 SIEMENS BOOTSTRAP / DIAGNOSTIC VERSION V 2.1` with `BOOT / HELP / LIST / MAP / SETUP / TEST`
- Quote: “Unfortunately the firmware version is 1.15 and FPP is only available from version 2 or higher, so — no FPP.”
- Quote: “According to the documentation — even with the required version of the FPP microcode (implemented using the i960) it differs from the standard DEC — it has 2–3 lower digits less (binary); DEC FPP calculates more accurately.”
- Attachment: **`Mentec_M11_roms.zip`** (4 microcode ROMs + 1 boot ROM, ~151 KB). This is the only commonly cited public dump. It is an **M11** image set, not an M1 FLASH programming file.

RT-11 on that board still printed “Floating Point Microcode” / “FPU support” in some `SHOW ALL` output even when owners believed FPP was absent at 1.15 — treat OS capability bits as suggestive, not proof. RSX-11M-Plus SYSGEN identified one example as “M11 (MENTEC)” with EIS / 22-bit / cache / parity options listed; FP option presence varied with firmware.

### 4.2 VCFED — Mentec M1 PDP-11 CPU (2014)

Thread: https://forum.vcfed.org/index.php?threads/mentec-m1-pdp-11-cpu.42073/

- eBay sellers sometimes list M1 boards as **no FPU**.
- Thread starter notes the archived M11 page (microcode 2.0+) and asks whether M1 revisions split the same way.
- 2.11BSD historically assumed J-11-class CPUs have hardware FP. Without FP, the in-kernel emulator could panic on the first user FP instruction; **patch #445** fixes that emulator panic. That is an OS workaround, not Mentec firmware.

### 4.3 VCFED — Booting a Mentec M1 (2025)

Thread: https://forum.vcfed.org/index.php?threads/booting-a-mentec-m1.1255114/

Practical install notes (Q/CD backplane jumpers W1/W2, MSCP `DU0` at 172150, Emulex / UniBone boot differences). Not FP-specific, but relevant if the board never reaches `SETUP`.

### 4.4 VCFED — Mentec documentation hunt

Thread: https://forum.vcfed.org/index.php?threads/mentec-documentation.64158/

Community recovered M70 / M80 / M90 user guides and some EPROM dumps; Bitsavers still only has a thin Mentec set. The **M1 User Guide** and **M11 brochure** remain the two usable official texts for this topic.

---

## 5. How to actually enable / verify floating point

This is the operational checklist implied by the sources. It is not a substitute for the M1 User Guide.

1. **Identify the microcode revision** at power-up.  
   M11 prints `M11 Microcode Rev. V x.xx`. M1 uses the LED WCS-load sequence instead of that banner. Anything **below 2.0** should be assumed to have **no FP emulation**.

2. **Console into dialogue mode**  
   Factory baud from rotary switch SW2 (M1 User Guide Ch. 4). Break autoboot with Ctrl-C if needed.

3. **Run `SETUP`**  
   Set hardware / software options, autoboot list, device translations. **Save Modified Setup** writes FLASH. This stores *configuration*, not a new microcode image.

4. **Watch LEDs**  
   - LED 13 (i960 Reset) should drop after the helper CPU comes out of reset.  
   - LED 5 (FP Indicator) is the FP-path lamp.  
   - LEDs 1–4 track WCS / microcode load.

5. **Confirm in the OS**  
   RT-11: `SHOW ALL` / `RESORC` — look for FPU / Floating Point Microcode.  
   RSX: `ACO SHOW` / SYSGEN processor options.  
   Do not trust a single capability bit if the banner was 1.x.

6. **Know the numeric difference**  
   Even with 2.0+ microcode, Mentec FP is **IEEE internally** and field reports say it is a couple of low bits less precise than DEC FP11. Software that depends on exact DEC FP bit patterns can disagree.

7. **If you need FP and the board is 1.x**  
   `SETUP` will not create it. You need Mentec microcode ≥ 2.0 in the load path. That image was never published as a supported field-upgrade kit. The public **`Mentec_M11_roms.zip`** is a 1.15-class dump and is the wrong generation for an M1 FLASH device anyway.

8. **If the OS requires FP and you cannot get 2.0+ firmware**  
   Use a software emulator path (2.11BSD patch #445 and similar). Compatibility is then “good enough for many programs,” not “J-11 + FP11.”

---

## 6. What “firmware” exists and what does not

| Artifact | Public? | What it is |
|---|---|---|
| M1 User Guide 1998 (MIUG Rev VI.O) | Yes — Bitsavers PDF | Board config, FLASH `SETUP`, LEDs, architecture |
| M11 brochure | Yes — Wayback | i960 FP + “microcode 2.0 or later” |
| M11 Development / FUSE project PDF | Partial — Wayback | VHDL / ASIC / Atmel 0.85 µm statement |
| On-board FLASH contents of a given M1 | Only if you dump that card | Boot + diagnostics + saved `SETUP` |
| M11 discrete microcode + boot ROMs | Hobby dump: `Mentec_M11_roms.zip` on VCFED | Not an M1 programmer file |
| i960 FP emulator source | **Not published** | Proprietary Mentec |
| Official Mentec “enable FP” upgrade image for M1 | **Not found** | No known public distribution |

Mentec Inc. (US, PDP-11 OS rights) went bankrupt in 2006. Mentec Ltd. was acquired by Calyx. There is no current vendor firmware portal.

---

## 7. Related manuals worth keeping next to this note

- Mentec *SBC M1 User Guide* (1998) — Bitsavers path above  
- Mentec M11 brochure — Wayback URLs above  
- DEC *DCJ11 User Guide*  
- DEC *Micro PDP-11 Handbook*  
- DEC *Micro PDP-11 Interface Handbook*  
- Wikipedia: https://en.wikipedia.org/wiki/Mentec  
- Wikipedia: https://en.wikipedia.org/wiki/Intel_i960  
- Intel 80960KB / 80960MC family docs (i960 with on-chip FPU; Mentec used the i960 as a helper, not as the PDP-11 ISA CPU)

Bitsavers Mentec directory:

- https://bitsavers.org/pdf/mentec/

---

## 8. Source list (compact)

1. Mentec Limited. *M1 Single Board Computer User Guide*, Version 1.0, REF: MIUG, Issue Rev VI.O, 1998. https://bitsavers.org/pdf/mentec/SBC_M1_User_Manual_1998.pdf  
2. Mentec. *M11 Q-BUS SBC brochure*. Archived 1997-07-25. https://web.archive.org/web/19970725211216/http://www.mentec.com/PDP/m11brochu.htm  
3. Mentec. *M11 brochure* (1999 mirror). https://web.archive.org/web/19990819134502/http://mentec.com/PDP/m11brochu.htm  
4. Mentec Inc. *M11 specs page* (2006 archive used in 2014 VCFED thread). http://web.archive.org/web/20060324070926/http://www.mentec-inc.com/m11specs.htm  
5. “Mentec.” *Wikipedia*. https://en.wikipedia.org/wiki/Mentec  
6. Mentec / FUSE *Development Project Report* (M11 VHDL, M1 Atmel ASIC). https://web.archive.org/web/20160412201307/https://www.fuse-network.com/fuse/demonstration/30/24675/24675.pdf  
7. Hunta et al. “Mentec M11.” *Vintage Computer Federation Forums*, 2022. Includes `Mentec_M11_roms.zip`. https://forum.vcfed.org/index.php?threads/mentec-m11.1240709/  
8. “Mentec M1 PDP-11 CPU.” *VCFED*, 2014. https://forum.vcfed.org/index.php?threads/mentec-m1-pdp-11-cpu.42073/  
9. “Booting a Mentec M1.” *VCFED*, 2025. https://forum.vcfed.org/index.php?threads/booting-a-mentec-m1.1255114/  
10. “MENTEC Documentation.” *VCFED*. https://forum.vcfed.org/index.php?threads/mentec-documentation.64158/  
11. “Intel i960.” *Wikipedia*. https://en.wikipedia.org/wiki/Intel_i960  

---

## 9. Disclaimer

This note reconstructs Mentec’s published behavior and hobbyist dumps. It is not a programming specification for the i960 helper firmware and is not a license to copy Mentec microcode. Octal addresses, LED numbers, and command names are taken from the 1998 M1 User Guide scan and from archived Mentec marketing text; OCR of that scan is imperfect in places. When in doubt, trust the PDF page over a secondary summary.
