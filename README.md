## Communication

**Hook: Win32k Shadow SSDT (****`W32pServiceTable`****)**

- Hooks `NtUserMessageCall` using its shadow SSDT index, pre-resolved from `win32u.dll` by the mapper and written into the driver's `.data` section before `DriverEntry` is called
- Sentinel: `xpfnProc == 0xDEAD` — filters all legitimate `NtUserMessageCall` traffic at near-zero cost. Only probes the `CommandRequest` struct if the sentinel matches
- Fallback: APC command channel via `winlogon.exe` thread — fully operational if SSDT hook fails
- Does **not** touch `KiServiceTable` (main SSDT) — PatchGuard monitors that table

**Command Types dispatched in kernel:**

- `Ping` — liveness check
- `Read` — physical memory read from target process
- `Write` — physical memory write to target process
- `GetBase` — module base + size lookup by name via PEB LDR walk (kernel-attach, no handle)

---

## Memory Engine

**CR3 / Page-Table Walk (no ****`MmCopyVirtualMemory`****)**

- Resolves `EPROCESS.DirectoryTableBase` dynamically at load by matching `__readcr3()` against the System process EPROCESS blob — handles all Windows 10/11 field offset variants without a hardcoded offset
- Full 4-level page table walk: PML4 → PDPT → PD → PT
- Handles 1GB large pages and 2MB large pages correctly
- Reads via `MmCopyMemory(..., MM_COPY_MEMORY_PHYSICAL)` — kernel-safe, no MDL
- Writes via `MmMapIoSpaceEx(..., PAGE_READWRITE)` + `RtlCopyMemory` — bypasses write protection on normally read-only pages
- Crosses page boundaries transparently — a single `Read(addr, size)` spanning two pages works correctly

---

## Stealth — Load-Time (DriverEntry, runs once)

**Module list unlink**

- Removes `LDR_DATA_TABLE_ENTRY` from all 3 lists:
  - `InLoadOrderLinks`
  - `InMemoryOrderLinks`
  - `InInitializationOrderLinks`
- Zeroes `BaseDllName`, `FullDllName`, `DllBase`, `SizeOfImage` fields
- Nulls `DriverObject->DriverSection` — kernel callbacks can't re-find the entry
- List entries pointed to themselves (safe self-loop, no dangling pointers)

**PE header wipe**

- Allocates MDL over `DriverObject->DriverStart` for `SizeOfHeaders` bytes
- Maps as writable via `MmMapLockedPagesSpecifyCache`
- `RtlZeroMemory` — header is blank in memory. No MZ/PE signature to scan for

**Driver object name erasure**

- `ObjDir::EraseDriverName` removes the driver's name from the kernel object namespace — `\Driver\SiegeKm` does not exist after load

**ntoskrnl base resolution**

- Anchors on `&PsInitialSystemProcess` (always inside ntoskrnl `.data`)
- Scans backwards page-by-page checking for `MZ`/`PE` signature
- Resolves without any `ZwQuerySystemInformation` call that AC monitors

**AC callback blinding**

- Nulls slots in `PsSetLoadImageNotifyRoutine`, `PsSetCreateProcessNotifyRoutine`, `PsSetCreateThreadNotifyRoutine` callback arrays
- Runs before the module list unlink — AC never gets a "new driver loaded" notification

---

## Stealth — Runtime (Guard Loop, continuous)

**KTIMER + KDPC — fires every 5 seconds**

- `KTIMER`/`KDPC` is the standard kernel timer mechanism — PatchGuard-safe by construction. Does not touch SSDT, does not modify code pages

**Re-blind AC callbacks** (every 5 seconds)

- BEDaisy re-registers its callback slots after certain system events (boot, session changes)
- Guard loop calls `CallbackBlind::BlindAll()` on each tick — any re-registered slots are nulled again

**Pool tag rotation** (every \~17 seconds)

- `POOL_HEADER.Tag` is at `allocation - 0x10 + 0x04`
- Cycles through 8 tags: `PfnD`, `MmSt`, `NtFs`, `CcBc`, `IoPa`, `ObNm`, `LpcP`, `MmPg` — all mimic legitimate kernel allocations
- Written via `InterlockedExchange` — atomic, no window where tag is invalid

**Working set eviction** (every 5 seconds)

- Calls `MmAdjustWorkingSetSize(0, 0, FALSE)` — hints MM to evict driver pages from the working set
- Pages not in the working set don't appear in `NtQueryVirtualMemory(WorkingSetExInformation)` — the API some ACs use to find anonymous executable pool pages.....


## For sale
$450 with cheat source too $350 for just driver 
