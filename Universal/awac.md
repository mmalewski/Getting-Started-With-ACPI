# Fixing System Clocks

**For Intel 300 series chipsets and newer**, this also includes X299 refreshes and Icelake laptops. Common machines:

* X299X (10th Gen refresh)
* B360
* B365
* H310
* H370
* Z370 (Gigabyte and AsRock boards with newer BIOS versions)
* Z390
* 400 series (Cometlake)
* 495 series (Icelake)

So on newer Intel 300 series motherboards, manufactures started pushing for a new type of system clock: **AWAC**( **A** **W**eird **A**ss **C**lock). One small problem, macOS doesn't know what the hell an AWAC clock is instead only familiar with the legacy **RTC**(**R**eal **T**ime **C**lock). So we need to figure out how to bring back the old clock, thats where `SSDT-AWAC` and `SSDT-RTC0` come in:

* [SSDT-AWAC](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-AWAC.dsl)
  * Disables AWAC and enables RTC
  * In your DSDT, there's a variable called `STAS` used for holding either a `One` or `Zero` to determine which clock to use(`One` for RTC and `Zero` for AWAC)

* [SSDT-RTC0](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-RTC0.dsl)
  * Used for creating a fake RTC device for macOS to play with
  * In very rare circumstances, some DSDTs may not have a legacy RTC to fall back on. When this happens, we'll want to create a fake device to make macOS happy

Note: AWAC actually stands for ACPI Wake Alarm Counter/Clock for those curious, though I'll forever know it as A Weird Ass Clock ;p

## Determining which SSDT you need

To determine whether you need [SSDT-AWAC](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-AWAC.dsl) or [SSDT-RTC0](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-RTC0.dsl):

* open your decompiled DSDT and search for `Device (AWAC)`
* If **nothing shows up** then no need to continue and **no need for this SSDT** as you have no AWAC. **Otherwise, continue on!**
* If you get a result then you have an `AWAC` system clock present, then continue with the **next search for `STAS`**:

![](/images/Universal/awac-md/stas.png)

As you can see we found the `STAS` in our DSDT, this means we're able to force enable our Legacy RTC. In this case, [SSDT-AWAC](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-AWAC.dsl) will be used As-Is with no modifications required. Just need to compile. Note that `STAS` may be found in AWAC first instead of RTC like in our example, this is normal.

For systems where **no `STAS`** shows up **but** you do have `AWAC`, you can use [SSDT-RTC0](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-RTC0.dsl) though you will need to check the naming of LPC in your DSDT

By default the SSDT uses `LPCB`, you can check what your system uses by just searching for `Name (_ADR, 0x001F0000)`. This address is used for Low Pin Count devices(LPC) but the device name can vary between `LPCB`, `LBC` or `LBC0`:

![](/images/Universal/awac-md/lpc.png)

## _INI Edge Cases

Mainly seen on X299 refresh boards, there's already a `Scope (_SB) { Method (_INI...` in your DSDT. This means our SSDT-AWAC will conflict with the one found in our DSDT. For these situations, you'll want to remove `Method (_INI, 0, NotSerialized) {}` from the SSDT. You'll be left this this in the end:

```
DefinitionBlock ("", "SSDT", 2, "DRTNIA", "AWAC", 0x00000000)
{
    External (STAS, IntObj)

    Scope (_SB)
    {
        If (_OSI ("Darwin"))
        {
            STAS = One
        }
    }
}
```

You can find a prebuilt of this here: [SSDT-AWAC.aml](https://github.com/dortania/Getting-Started-With-ACPI/blob/master/extra-files/SSDT-AWAC.aml)


## [Now you're ready to compile the SSDT!](/Manual/compile.md)
