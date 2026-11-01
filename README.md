# Nokia-G240W-F-Research

**Updated:** 2026-11-01  

**Current Progress** discontinued (archived)

**OpenWRT Port** not yet

**Purpose:** Handoff context for continued investigation of the Nokia G-240W-F configuration, bootloader, rescue page, and shell access.

## Device and version evidence

| Item | Evidence |
|---|---|
| Model / board | Startup log identifies product `G-240W-F` and board `G240WF-200MW`. |
| SoC | Startup log identifies `EN751221`; the earlier handoff's `EN7526GT` identification is unverified and conflicts with this log. |
| RAM | 256 MiB. |
| NAND | Macronix SPI NAND, manufacturer ID `0xc2`, device ID `0x12`, capacity 128 MiB. |
| Firmware | This startup capture reports `3FE46606BFHB21`; earlier handoff lists `3FE46606CGBA08`. Confirm which build was used for the unsuccessful config attempt. |
| Linux | Kernel 3.18.21, GCC 4.9.3. Startup reaches the `user>` / `user#` CLI. |

The workspace did not contain the original config, extracted XML, `nokia.py`, firmware dump, or serial log when checked. Details below are from the user's pasted handoff and serial output, not local artifact inspection.

## NAND partition map from startup log

Offsets are the kernel's reported MTD partition boundaries. They are not proof of a validated raw NAND dump or physical OOB layout.

| Partition | Start | Size |
|---|---:|---:|
| bootloader | `0x00000000` | `0x00040000` |
| romfile | `0x00040000` | `0x00040000` |
| kernel | `0x00080000` | `0x00300000` |
| rootfs | `0x00380000` | `0x01F00000` |
| kernel_slave | `0x02280000` | `0x00300000` |
| rootfs_slave | `0x02580000` | `0x01F00000` |
| kernel_oflt | `0x04480000` | `0x00300000` |
| rootfs_oflt | `0x04780000` | `0x00C00000` |
| config | `0x05380000` | `0x00800000` |
| log | `0x05B80000` | `0x00C00000` |
| extfs | `0x06780000` | `0x00600000` |
| bosa | `0x06D80000` | `0x00040000` |
| flag | `0x06DC0000` | `0x00040000` |
| flagback | `0x06E00000` | `0x00040000` |
| ri | `0x06E40000` | `0x00040000` |
| riback | `0x06E80000` | `0x00040000` |

The `config` and `log` MTD partitions are UBI/UBIFS mounted in the boot log. The config partition has one UBI volume (`configfs`); log has `logfs`.

## Serial / bootloader session

- User connected a SerialPRO Reader at `/dev/ttyUSB0` in the Codex execution environment. It appeared as a CH341 UART adapter (USB ID `1a86:7523`), accessible for reading and writing.
- Minicom is open at 115200 baud. The baud rate came from the user's earlier explicit instruction; it was not auto-detected.
- The device is currently at the bootloader prompt `bldr>` (last observed). Keep the serial session open until the user asks to disconnect.
- User interrupted/rebooted the router to reach the bootloader. No reboot was initiated by the agent.
- `httpd` was run from the bootloader at the user's request. Serial output stated `tcp_bind: bind to port 80`; the user confirmed the rescue page works from their PC at `http://192.168.1.1` after setting a static IP.
- The rescue page has a login; the user does not know the credentials. The agent's in-app browser access to the local URL was declined by the browser permission gate. The user can access it locally.

### Bootloader help, confirmed commands

Read/inspection-related commands shown by `help`: `memrl`, `dump`, `imginfo`, `nandrd`, `chkmem`, `bdshow`, `miir`, `ritool`, `mtd` (the latter commands' behavior/details are not established). Other commands include state-changing or potentially destructive commands: `reset`, `serial disable`, `memwl`, `jump`, `flash`, `nandit`, `nander`, `bdstore`, `bdswitch`, `ddrcalswitch`, `drambistswitch`, `xmdm`, `miiw`, `cpufreq`, `ipaddr`, `httpd`, and `ddrdrv`. Do not run write/erase commands.

`bdshow` was run as read-only and returned: `back door config is not support NAND Flash`. This did not reveal rescue credentials.

`imginfo` returned `os1:3FE46606BFHB21`, followed by garbled output that could not be interpreted.

## NAND reads attempted

- `nandrd` with no arguments returned usage: `nandrd <src> <len>`.
- `nandrd 0 40` returned a 64-byte hex sample at offset `0x0` beginning `0b f0 00 12 ...`; this confirmed the read command emits hex to the serial console.
- `nandrd 5380000 100` attempted a 256-byte read at the reported config partition start. Output was garbled/incomplete and ended with `retlen : 0`; do not treat it as a reliable dump.
- No NAND write or erase command has been run.
- A complete 128 MiB dump via the text serial console would be very large and slow at 115200 baud. A validated capture/transfer mechanism is not yet established. Do not claim a full dump exists.

## Linux CLI and root-shell investigation

- Linux startup shows a Zebra VTY help banner. At `user#`, `help` displayed the Zebra VTY context-help explanation. `?`, `show ?`, and `sh?` each returned `% Unknown command.`
- A Ctrl-C was sent while attempting to wake the initially silent minicom screen. It produced a kernel `task blocked for more than 120 seconds` warning and returned the `user#` prompt. Avoid sending control characters casually.
- The prior G-2425G-A `enable shell` / `Password2` method is unverified on this G-240W-F and should not be assumed compatible.
- The attempted `LimitAccount_ONTUSER` change from true to false did not produce the expected result, per the user. XML and repack artifacts are not present in the workspace, so no independent comparison has been done.

## Rescue webpage / vulnerability research

- The bootloader rescue HTTP server listens on port 80, based on serial output. The login page has not been inspected by the agent; no source or screenshot has been provided.
- Public research found a G-240W-A normal web-interface writeup and Tenable vulnerabilities for the I-240W-Q. These are different models/interfaces and do **not** establish an exploit for this G-240W-F bootloader rescue page.
- Do not try guessed credentials or send exploit requests. Best next evidence: user-provided screenshot or saved HTML source from the rescue login page, with cookies, credentials, serial numbers, and identifiers redacted.

## Recommended next steps

1. Have the user capture the rescue login page (screenshot or saved HTML source), redacting secrets. Inspect the actual page before drawing exploit conclusions.
2. Confirm whether the failed config change was made on firmware `3FE46606BFHB21` or `3FE46606CGBA08`.
3. Obtain the XML/config and `nokia.py` artifacts if offline analysis is desired; keep `ConfigOrig.cfg` untouched and work on copies.
4. For NAND, first establish a reliable capture method and validate a small read end-to-end. Prefer a targeted partition read only after address semantics and transfer path are proven. Keep all writes/erases off-limits unless the user explicitly changes scope.

