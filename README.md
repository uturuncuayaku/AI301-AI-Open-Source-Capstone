# Phase I

## DOSBox-Staging APPEND Command Implementation

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866  
**Status:** Phase I - In Progress

---

## Why I Chose This Issue

I'm choosing this issue because I used to play Sierra Online Games within the Mac terminal and MS-DOS terminal when I was younger. To this day I still play video games, and a retro gaming simulator allows people to continue their love for video games if they get tired of playing modern titles. Additionally, not everyone has access to very powerful PCs, but with this emulator they can still enjoy games with friends. The maintainers have given clear guidance on how they'd like this solved, and I believe in giving back to communities that promote fun and engaging activities on PCs—because that was my first exposure to computers.

---

## Problem Statement

The MS-DOS `APPEND` command enables applications to search for data files across multiple directories without requiring the full path. Many legacy DOS applications rely on this functionality to locate resources.

Currently, DOSBox-Staging does not implement the `APPEND` command, which limits compatibility with software that depends on it.

---

## Solution Approach

**Goal:** Implement the MS-DOS v4.0 `APPEND` command to enable file lookup across multiple directories.

### Reference Implementation

- **MS-DOS v4.0 APPEND Source:** [microsoft/ms-dos — APPEND.ASM](https://github.com/microsoft/ms-dos/blob/master/v40/dev/append/append.asm)

### APPEND Command Behavior

```
APPEND [d:]path[;[d:]path...]     set the search list
APPEND ;                          clear the search list
APPEND                            display the current list
```

**Key Features:**
- Parse and store semicolon-separated directory paths
- Transparently redirect failed file opens to appended directories
- Support INT 2Fh multiplex handler for state querying
- Minimal viable implementation focused on core functionality

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

- Review research findings
- Implement integration hooks in DOSBox-Staging
- Testing and validation
- Submit to upstream dosbox-staging project

---

## References

- **Original Issue:** [dosbox-staging#4866](https://github.com/dosbox-staging/dosbox-staging/issues/4866)
- **Research Branch:** `iteration-2-phase-1` (contains detailed technical analysis and implementation details)
- **Upstream Fork:** [uturuncuayaku/dosbox-staging](https://github.com/uturuncuayaku/dosbox-staging)
