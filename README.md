# BNUY-ROM

**Everything in this repo is a WIP, this is not a 100% final design, this also includes many potential alternative designs that may be used or added as optional board configurations.**

An extension of sorts of the BNROM mapper:

* Banked PRG-FLASH in a 32KiB window:
* * **39SFxx** - Up to 512 KiB, partially rewritable
* * **29Fxxx** - Up to 2048KiB, fully rewritable
* (optional) Four-Screen Mirroring
* (optional) 32KiB or 128KiB of CHR-RAM in 4x2KiB banked windows
* (optional) PPU Scanline IRQ
* (optional) Audio Expansion via a SAM2695 soundchip
* (optional) 32KiB of PRG-RAM

As an in-development mapper, there is currently no iNES number attributed, nor emulator support.

## Why?

Bnuy-Rom was developed for my personal homebrew projects, with the goal of being a mapper with a similar level of capabilities to ASIC mappers like the MMC3, while only using generic 74x logic chips I could source and assemble myself. Many of its features are left optional, for instance, the game I'm currently developing for this mapper will likely not make use of PRG-RAM or the SAM2695 sound expansion.

This repo includes 60-pin famicom boards fitted for MMC cartridge shells. They have currently not been fully tested.

![](render.png)
![](render-tht.png)
![](schematic.png)

# Banks
* CPU $6000-$7FFF: 8KiB switchable PRG-RAM
* CPU $8000-$FFFF: 32 KiB switchable PRG-FLASH
* PPU $0000-$07FF: 2 KiB switchable CHR-RAM Bank
* PPU $0800-$0FFF: 2 KiB switchable CHR-RAM Bank
* PPU $1000-$17FF: 2 KiB switchable CHR-RAM Bank
* PPU $1800-$1FFF: 2 KiB switchable CHR-RAM Bank

# Registers

## PRG-FLASH/CTRL Register ($8000-$9FFF, write)

This register controls both banking of PRG Memory, the IRQ Generator and SAM soundchip.

```
D~7654 3210
  ---------
  ISbb BBBB
  |||| ++++--- PRG-Flash Bank
  ||++-------- shared PRG-Flash Bank and PRG-RAM bank
  |+---------- SAM2695 /CS
  +----------- Freeze IRQ Scanline Counter
```

Note the shared banking bits for PRG-Flash and PRG-Ram for cases where more than 512K of PRG-Flash are used.

### Potential Alternative A

```
D~7654 3210
  ---------
  bbBB BBBB
  ||++-++++--- PRG-Flash Bank
  ++-----------PRG-Ram Bank
```

Here the upper 2 bits are instead used for PRG-RAM, with the IRQ Counter now always being enabled, and no enable for the potential soundchip. If there is no prg-ram and/or prg-rom is not larger than 512KiB, physical registers of 6 or 4 bits could be used instead.

## SAM2695 Registers ($A000-$BFFF, write)

**This chip is now EOL, so it will not be used. This register region will therefore be free for something else to use.**

If present, this area will write to the SAM2695 soundchip, note that the SAM's Chip-Select is controlled by the respective bit in the PRG-FLASH/CTRL Register.

```
$A000 (even): Data
$A001 (odd) : Control

D~7654 3210
  ---------
  DDDD DDDD
  ++++-++++- 8-bit Data
```
## IRQ Counter Load ($C000-$DFFF, write)
  
If present, this register will load a value into a scanline counter which decrements once per active scanline, the counter will continuously trigger an IRQ whenever its value is 0.
There is no reload latch, so this register must be manually reloaded by software after each use.

To acknowledge an IRQ it should either be set to a value different from zero, or disabled by clearing the I bit in PRG-BANK, which will lock the counter at 255.

```
D~7654 3210
  ---------
  IIII IIII
  ++++-++++- 8-bit scanline counter value
```

The Counter is triggered by 4 successive reads with PPUA13=1, which occurs once per active scanline, on ppu cycle 4, this is similar to MMC5's scanline counter. Unlike the MMC3, this allows sprites to freely use either pattern table, including both while in 8x16 mode.

It uses a x4520 to detect scanlines, and either a x40103 or two x191/x193s to hold the counter value. 

## CHR-RAM Banking ($E000-$FFFF, write)

If present, these registers select one of 16 2KiB CHR-RAM banks, to use for each of the 4 PPU pattern table windows.

In **shared mode**, Banks 14 and 15 are shared with the 4 nametables, and should be avoided unless special care is taken. This leaves 14 banks that can be freely distributed across the 4 windows. 

In **independent mode**, the register is still 4 bits wide, but now each window has their own unique set of 16 banks, with the first 2 windows sharing banks 15 with the nametables. This effectively quadruples the amount of chr-ram without adding a second register chip to the board.

```
$E000: CHR-RAM Bank 0 ($0000-$07FF)
$E001: CHR-RAM Bank 1 ($0800-$0FFF)
$E002: CHR-RAM Bank 2 ($1000-$17FF)
$E003: CHR-RAM Bank 3 ($1800-$1FFF)

D~7654 3210
  ---------
  .... BBBB
       ++++- CHR Bank Select
```

This uses a single x670 4x4 register. A second register could be added to the design for an oversize variant of 512KiB (shared) or 2048KiB (independent). CHR-ROM could be used instead, at the cost of losing 4-screen nametable arrangement.

### Potential Alternative A

```
$E000: CHR Banks

D~7654 3210
  ---------
  BBBB BBBB
  |||| ++++- Pattern 0 Bank Select
  ++++------ Pattern 1 Bank Select
```

Here there are only 2 windows, the x670 is replaced with an 8bit register and a x157 8-to-4 multiplexer, Bank 0 of pattern 0 is the bank shared with the Nametables.

### Potential Alternative B

```
$E000: CHR Bank

D~7654 3210
  ---------
       BBBB
       ++++- CHR Bank Select
```

Here there is only 1 window, the x670 is replaced with either a x173 (128KiB) or x574 (2048KiB). Bank 0 or 15 is shared with the nametables (via resistors).

### Potential Alternative C

No Register! Instead there is a fixed 8KiB of CHR-RAM, 4 Nametables and ~4KiB of bonus ram at ppu $3000-3FFF. (Assuming a common 32KiB RAM chip)
