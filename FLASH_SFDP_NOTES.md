# PY25Q64HA: SFDP fixes

These fixes are already applied in this project. Check them after replacing or regenerating middleware. Return to the [setup guide](README.md).

### All board-specific changes in `stm32_sfdp_data.c`

**Compiled source:** `Middlewares/ST/STM32_ExtMem_Manager/nor_sfdp/stm32_sfdp_data.c`.

`ExtMemLoader/stm32_sfdp_data.c` is another copy. At the time of writing, both copies match. Editing only the copy under `ExtMemLoader` does not change the source selected by the current build. Keep the compiled file patched and do not compile both copies simultaneously.

All four workaround groups below are inside `SFDP_BuildGenericDriver()`. They handle missing or unusable entries in the memory's SFDP data. They are board-specific fallbacks, not universal replacements for valid SFDP information.

**1. Page size — replace the direct D11 assignment:**

```c
if ((JEDEC_Basic.size < 11u) ||
    (JEDEC_Basic.Params.Param_DWORD.D11.PageSize == 0u))
{
    Object->sfdp_private.PageSize = 256u;
}
else
{
    Object->sfdp_private.PageSize =
        ((uint32_t)1u << JEDEC_Basic.Params.Param_DWORD.D11.PageSize);
}
```

This selects a 256-byte page when D11 is absent or its page-size field is zero. Valid D11 values are still used.

**2. Erase timings — add after the existing timing calculations:**

```c
if (Object->sfdp_private.DriverInfo.EraseType1Timing == 0u)
    Object->sfdp_private.DriverInfo.EraseType1Timing = 6144u;

if (Object->sfdp_private.DriverInfo.EraseType2Timing == 0u)
    Object->sfdp_private.DriverInfo.EraseType2Timing = 24000u;

if (Object->sfdp_private.DriverInfo.EraseType3Timing == 0u)
    Object->sfdp_private.DriverInfo.EraseType3Timing = 15360u;

if (Object->sfdp_private.DriverInfo.EraseChipTiming == 0u)
    Object->sfdp_private.DriverInfo.EraseChipTiming = 120000u;
```

These are timeout budgets passed through the driver to HAL polling, in milliseconds, not measured erase durations. Existing nonzero values are retained. No Type4 fallback is added in this project. Erase sizes and opcodes still come from SFDP; these changes do not add unsupported erase commands.

**3. Write Enable — replace only the final D16 error branch:**

Replace the `EXTMEM_SFDP_ERROR_JEDECBASIC_D16` / `goto error` fallback with:

```c
Object->sfdp_private.DriverInfo.WriteWELCommand =
    SFDP_DRIVER_WRITE_ENABLE_06H_COMMAND;
```

This selects Write Enable `0x06` when the preceding SFDP cases do not provide a usable configuration. Keep those preceding cases intact.

**4. Busy polling — replace only the final D14 error branch:**

Replace the `EXTMEM_SFDP_ERROR_JEDECBASIC_D14` / `goto error` fallback with:

```c
Object->sfdp_private.DriverInfo.ReadWIPCommand =
    SFDP_DRIVER_READ_STATUS_REGISTER_COMMAND;
Object->sfdp_private.DriverInfo.WIPPosition = 0u;
Object->sfdp_private.DriverInfo.WIPBusyPolarity = 0u;
```

This uses status-register command `0x05` and WIP bit 0: 0 = ready, 1 = busy. Preserve the existing supported D14 branches.

Recheck these four patches after regenerating or replacing middleware. The loader geometry header and SFDP page-size calculation are separate settings; setting the header to 256 bytes does not fix the SFDP parser.

