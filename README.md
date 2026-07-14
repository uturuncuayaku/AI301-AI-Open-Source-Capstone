# Phase II

## DOSBox-Staging APPEND Command Implementation

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866  
**Status:** Phase II - In Progress

---

## Why I Chose This Issue

I'm choosing this issue because I used to play Sierra Online Games within the Mac terminal and MS-DOS terminal when I was younger. To this day I still play video games, and a retro gaming simulator allows people to continue their love for video games if they get tired of playing modern titles. Additionally, not everyone has access to very powerful PCs, but with this emulator they can still enjoy games with friends. The maintainers have given clear guidance on how they'd like this solved, and I believe in giving back to communities that promote fun and engaging activities on PCs—because that was my first exposure to computers.

---

## Problem Statement

The MS-DOS `APPEND` command enables applications to search for data files across multiple directories without requiring the full path. Many legacy DOS applications rely on this functionality to locate resources.

Currently, DOSBox-Staging does not implement the `APPEND` command, which limits compatibility with software that depends on it.

---

## Steps to Reproduce

### Prerequisites
- DOSBox-Staging compiled and running locally
- A test DOS environment with multiple directories
- A simple test program that attempts to open a file not in the current directory

### Reproduction Process

1. **Set up DOSBox-Staging build environment**
   - Clone DOSBox-Staging repository
   - Install build dependencies (Meson, Ninja, C++ compiler)
   - Compile DOSBox-Staging from source

2. **Create test directory structure**
   - Create `C:\TESTAPP\` directory
   - Create `C:\TESTAPP\DATA\` subdirectory
   - Create `C:\TESTAPP\RUNFROM\` subdirectory
   - Place test file `NEEDLE.TXT` in `C:\TESTAPP\DATA\`

3. **Launch DOSBox-Staging and mount test directory**
   ```
   mount C <path-to-testapp>
   C:
   ```

4. **Attempt to use APPEND command**
   ```
   APPEND C:\TESTAPP\DATA
   ```
   **Result:** Command not recognized error (APPEND is not implemented)

5. **Attempt to access file across directories**
   ```
   cd C:\TESTAPP\RUNFROM
   type NEEDLE.TXT
   ```
   **Result:** File not found error (file exists in `DATA\` but is not accessible from `RUNFROM\`)

6. **Verify expected behavior with placeholder**
   - Create a dummy `APPEND.EXE` file
   - APPEND command is now "recognized" but provides no functionality
   - File lookup still fails because dummy APPEND has no effect

---

## Reproduction Evidence

- **Phase II Branch:** [iteration-2-phase2](https://github.com/uturuncuayaku/AI301-AI-Open-Source-Capstone/tree/iteration-2-phase2)
- **Research Branch:** [iteration-1-denied-pr](https://github.com/uturuncuayaku/AI301-AI-Open-Source-Capstone/tree/iteration-1-denied-pr) (contains detailed technical analysis)

---

## Implementation Plan

### Core Strategy
Implement a native APPEND command that intercepts file-open operations and transparently searches appended directories when the initial lookup fails.

### Implementation Steps

1. **Create APPEND state management**
   - Add `dos_append.h` and `dos_append.cpp` to `src/dos/`
   - Define state structure to store enabled flag and directory list
   - Implement list parsing (semicolon-separated paths)
   - Implement list storage and retrieval

2. **Register INT 2Fh multiplex handler**
   - Update `src/dos/dos_misc.cpp` to call `APPEND_Init()`
   - Register APPEND multiplex handler in the chain
   - Support INT 2Fh AH=B7h queries (install check, version, state)

3. **Hook file-open operations**
   - Update `src/dos/dos_files.cpp` to intercept failed opens
   - When a file is not found, attempt resolution through appended directories
   - Return resolved path on first match (transparent to caller)
   - Preserve original error if file not found in any appended directory

4. **Create transient APPEND program**
   - Add `program_append.cpp` and `program_append.h` to `src/dos/`
   - Register as internal `Z:\APPEND.COM` command
   - Parse command-line arguments (paths, clear command, display)
   - Update internal state via `APPEND_SetDirList()`

5. **Update build system**
   - Add new source files to `src/dos/meson.build`
   - Ensure proper compilation and linking

6. **Implement guest memory mirror**
   - Allocate guest RAM for directory list
   - Maintain sync between host state and guest-accessible memory
   - Allow direct guest-side modifications via INT 2Fh pointer

### Reference Implementation

- **MS-DOS v4.0 APPEND Source:** [microsoft/ms-dos — APPEND.ASM](https://github.com/microsoft/ms-dos/blob/master/v40/dev/append/append.asm)
- **Integration Guide:** See `iteration-1-denied-pr` branch for detailed technical specifications

### Testing Strategy

1. Verify APPEND command is recognized
2. Test path configuration (set, clear, display)
3. Test transparent file lookup across directories
4. Verify INT 2Fh multiplex handler responses
5. Test error handling (file truly not found)
6. Automated test suite using provided testware

---

## Next Steps

- Finalize implementation based on this plan
- Create pull request for upstream review
- Iterate on feedback
