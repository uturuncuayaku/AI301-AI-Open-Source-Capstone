# Phase I

## DOSBox-Staging APPEND Command Implementation

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866  
**Status:** Phase I - In Progress

---

## Why I Chose This Issue

I'm choosing this issue because I used to play Sierra Online Games within the Mac terminal and MS-DOS terminal when I was younger. To this day I still play video games, and a retro gaming simulator allows people to continue their love for video games if they get tired of playing modern titles. Additionally, not everyone has access to very powerful PCs, but with this emulator they can still enjoy games with friends. The maintainers have given clear guidance on how they'd like this solved, and I believe in giving back to communities that promote fun and engaging activities on PCs—because that was my first exposure to computers. I particularly want to understand how the emulator works so well with the current command set and help bring more games into compatibility with this repository.

---

## Understanding the Issue

A DOS game expects an entire DOS environment. Although modern x86-64 processors remain compatible with much of the original Intel 8086 instruction set, the difficult part is emulating the machine around the CPU. The emulator translates CPU instructions into C++ code while implementing the memory access patterns of a 1970s-1990s era PC. This is critical because games running directly on the host PC would have too-permissive access—they might disable interrupts, program hardware ports directly, manipulate memory, or install interrupt handlers—leading to instability or security issues. Emulating this environment safely with C++ is essential.

### DOS Game Requirements

Legacy DOS games depend on several core system components:
- INT 21h (DOS API)
- INT 10h (Video BIOS)
- INT 13h (Disk BIOS)
- INT 16h (Keyboard BIOS)
- EMS/XMS memory
- Sound Blaster and AdLib audio
- VGA hardware
- PIC interrupts and PIT timers

### Problem Description

The Origin Wing Commander Deluxe CD-ROM installer expects standard MS-DOS utilities including **APPEND.EXE**, SUBST.EXE, and JOIN.EXE to be available during installation. DOSBox Staging currently provides some DOS utilities but does not implement these commands. As a result, the installer cannot complete successfully, reducing compatibility with legacy DOS software.

**Note:** SUBST has already been implemented; this effort focuses on APPEND and JOIN.

### Expected Behavior
The installer finds the required commands and completes installation successfully.

### Current Behavior
The installer fails because DOSBox doesn't implement the required utilities that the game's installer checks for.

### Affected Components

- DOSBox command interpreter (internal commands and executable utilities)
- DOS-similar filesystem emulation layer
- Drive and path management subsystem
- DOS utility command implementations (similar to existing SUBST support)
- Compatibility layer for legacy DOS installers

---

## Problem Deep Dive

Legacy games like Wing Commander require the `APPEND` command to locate data files across multiple directories using relative paths. The game generates commands like:

```
WC.EXE CD="D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT;" DD="C:\ORIGIN\WINGCMDR"
```

This command relies on `APPEND` for proper file lookup across semicolon-separated search paths. Without it, the game crashes with:

```
Redirected Exec failed -- "WC.EXE"
```

Currently, placeholder `.EXE` files can fool the installer's dependency check, but they cannot replicate the actual file-search behavior that the game needs to run.

---

## Solution Approach

**Goal:** Build a Minimum Viable Product (MVP) for the `APPEND` command that provides transparent file lookup across appended directories without requiring full MS-DOS feature parity.

### APPEND Command Behavior (MVP)

```
APPEND [d:]path[;[d:]path...]     set the search list
APPEND ;                          clear the search list
APPEND                            display the current list
```

**Key Features:**
- Parse and store semicolon-separated directory paths
- Transparently redirect failed file opens to appended directories
- Support INT 2Fh multiplex handler for state querying and detection
- Provide automated test suite for verification

### Design Rationale

This behavior mirrors existing DOSBox features:
- Similar to `PATH` environment variable parsing
- Similar to `mount` command filesystem mapping
- Focused on game compatibility rather than full MS-DOS feature replication

---

## Research & Design

A comprehensive technical analysis has been completed, covering:

1. **Execution Path Analysis**
   - Startup initialization
   - Command configuration
   - File opening and resolution

2. **Integration Points**
   - Build system modifications (meson.build)
   - Multiplex handler registration (dos_misc.cpp)
   - File open hook (dos_files.cpp)
   - INT 2Fh side channel support
   - Truename rewrite handling (dos.cpp)
   - Program registration (dos_programs.cpp)

3. **Behavioral Test Suite**
   - Installation check verification
   - State readback validation
   - Directory-list access testing
   - Actual file-resolution testing
   - Automated test fixtures (APPTEST.C, TESTAPP.BAT)

---

## Next Steps

- Review and refine research findings
- Implement integration hooks in DOSBox-Staging
- Run behavioral test suite
- Code review against CONTRIBUTING.md guidelines
- Submit to upstream dosbox-staging project

---

## References

- **Original Issue:** [dosbox-staging#4866](https://github.com/dosbox-staging/dosbox-staging/issues/4866)
- **Research Branch:** `iteration-2-phase-1` (contains detailed technical analysis and proof-of-concept)
- **Upstream Fork:** [uturuncuayaku/dosbox-staging](https://github.com/uturuncuayaku/dosbox-staging)
