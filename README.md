# BNUY-ROM
Yet another famicom discrete logic mapper made for fun, an extension of sorts of the BNROM mapper:
* 128, 256 or 512 KiB of rewritabble PRG-FLASH (39SF Based)
* 32KiB of CHR-RAM with 4x2KiB banked windows
* Four-Screen Mirroring
* (optional) Scanline IRQ Interrupt
* (optional) Audio Expansion via a SAM2695 soundchip 

![](render.png)

# Banks
* CPU $8000-$FFFF: 32 KiB switchable PRG-FLASH
* PPU $0000-$07FF: 2 KiB switchable CHR-RAM Bank
* PPU $0800-$0FFF: 2 KiB switchable CHR-RAM Bank
* PPU $1000-$17FF: 2 KiB switchable CHR-RAM Bank
* PPU $1800-$1FFF: 2 KiB switchable CHR-RAM Bank

# Registers

## PRG-FLASH/Control Register ($8000-$9FFF, write)

```
D~7654 3210
  ---------
  ..SI BBBB
    || ++++- PRG Bank Select
    |+------ Disable IRQ Scanline Counter
    +------- SAM2695 /CS        
```
## SAM2695 Registers ($A000-$BFFF, write)

If present, this area will write to the SAM2695 soundchip, note that the SAM's Chip-Select is controlled by the respective bit in the PRG-FLASH/Control Register. 

```
$A000 (even): Data
$A001 (odd) : Control

D~7654 3210
  ---------
  DDDD DDD
  ++++-++++- 8-bit Data
```
## IRQ Counter Load ($C000-$DFFF, write)

If present, this register will load a value into a IRQ PA12-based counter which decrements once per active scanline*, the counter will trigger an IRQ whenever its value reaches 7 or lower.
To clear an IRQ it should be set to a value greater than 7. There is no reload latch, so this register must be manually reloaded by software after each use.

```
D~7654 3210
  ---------
  IIII IIII
  ++++-++++- 8-bit scanline counter value
```
* This requires setting the background to pattern table 0 and sprites to pattern table 1, counting should be disabled by the bit in the PRG-FLASH/Control Register if the CPU is accessing PPU Memory.

## CHR-RAM Banking ($E000-$FFFF, write)

These registers select one of 16 2KiB CHR-RAM bank to use for each of the 4 PPU pattern table windows.
Note that banks 14 and 15 are also shared with the 4 nametables, and should therefore be avoided unless special care is taken to avoid overlapping data.

```
$E000: CHR-RAM Bank 0 (bottom of Pattern Table 1)
$E001: CHR-RAM Bank 1 (top of Pattern Table 1)
$E002: CHR-RAM Bank 2 (bottom of Pattern Table 2)
$E003: CHR-RAM Bank 3 (top of Pattern Table 2)

D~7654 3210
  ---------
  .... BBBB
       ++++- CHR Bank Select
```
