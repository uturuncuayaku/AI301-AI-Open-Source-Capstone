# DOSBox-Staging APPEND Command Implementation

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866  
**Status:** Phase III - In Progress
Currently implementing MS-DOS APPEND as a native feature in DOSBox
Staging (https://github.com/dosbox-staging/dosbox-staging). Reference for
original behavior: MS-DOS v4.0 source, src/CMD/APPEND/APPEND.ASM
(https://github.com/microsoft/MS-DOS).

---
## What APPEND Does
APPEND is "PATH for data files." `APPEND C:\DATA;C:\LIB` makes failed file
opens transparently retry in those directories. A program in C:\EMPTY doing
`open("REPORT.TXT")` finds C:\DATA\REPORT.TXT without knowing it.

## Building Solution or Minimal Viable Product

### Challenges
- Translating the open source APPEND.ASM into C++.
- Integrating the basic functionality into DOSBox-Staging.
- Scope creep, because APPEND.ASM has lots of different flags that determine it's functionality. So for now basic implementation is required. No flag's added to the command line command. Just directories and clearing the directory list.
- Native implementation is required because many game developers expected APPEND quirks such as not enough memory to hold a lot of directories for a game data path or file path resolution problems that were common.
- Understanding a lot of the architecture within the codebase. Sometimes it's completely true to MS-DOS and other times there are C++ abstractions so not understanding the entire codebase and where APPEND would work is taking a lot longer for me to verify due to the complexity of the architecture. For instance, there are multiple interrupts that must be caught and there is a program that is similar to APPEND so copying and learning from those programs first takes some time due to the complexity of emulating an entire operating system using x8086 CPU registers and RAM. The memory layout is new to me and the way files are opened and searched for is advanced but a fun learning experience.

## Core Architectural Insight

Real DOS APPEND was a TSR: one binary, two personalities.
- RESIDENT half: hooked INT 21h + INT 2Fh vectors, stayed in guest memory
  via INT 21h AH=31h (terminate-and-stay-resident), kept a 128-byte
  directory-list buffer in guest RAM.
- TRANSIENT half: command-line parser. On second invocation it detected the
  resident copy (INT 2Fh AX=B700h → AL=FFh) and edited the resident list
  through a pointer (AX=B704h → ES:DI) instead of re-installing.

DOSBox Staging inverts this: the "resident half" is native C++ compiled into
the emulator (dos_append.cpp). It is "installed" at DOSBox startup when
DOS_SetupMisc() calls APPEND_Init() — NOT when the user runs APPEND.COM.
There is no INT 21h AH=31h, no guest-side resident code, no vector saving.
State lives in C++ objects (std::string append_dir_list, uint16_t mode_flags).

The transient half is Z:\APPEND.COM (program_append.cpp), registered via
PROGRAMS_MakeFile("APPEND.COM", ProgramCreate<APPEND_PROGRAM>). Each run is a
fresh instance; it calls APPEND_SetDirList()/APPEND_GetDirList() directly and
exits. Last writer wins.

## Command Categories in DOSBox (context for where APPEND fits)
1. Shell built-ins (DIR, TYPE, CD): hardcoded in the shell, no binary.
2. Installed programs (CHKDSK, Z:\APPEND.COM): transient, run and exit.
3. Resident utilities (MSCDEX, SHARE, APPEND): persistent interrupt-servicing
   logic. In DOSBox these are native C++ registered once at startup.
APPEND is a HYBRID: resident C++ half + transient program half.

## Z:\APPEND.COM Grammar (this build)
  APPEND [d:]path[;[d:]path...]   set list (tolerates leading '=', AN008)
  APPEND ;                        clear list
  APPEND                          display "APPEND=..." or "No Append"

## Verification Methodology (for defending the design)
1. Code trace: follow TYPE MISSING.TXT through DOS_21Handler → DOS_OpenFile →
   failure exit → APPEND_ResolveName → recursive retry.
2. Reference comparison: APPEND.ASM hooks exactly two vectors (21h, 2Fh);
   they map to the C++ failure hook and the multiplex handler.
3. Instrumentation: counters on multiplex_calls vs resolve_calls prove the
   multiplex path is rare and resolution is failure-driven only.
4. Guest-side probes: DEBUG → AX=B700 INT 2F → AL=FF; B706 → BX=mode word.
5. Regression guard: file nowhere + APPEND active must still yield error 2
   with CF set.
6. Write-through test: real MS-DOS APPEND.EXE inside the emulator detects
   "already installed" via B700h and edits through B704h; internal APPEND
   must then display the edit (exercises sync_list_from_guest_memory).

## Steps to Reproduce

### Prerequisites
- DOSBox-Staging compiled and running locally
- Development environment: Windows 11, Visual Studio 2022, CMake, Python 3
- A test DOS environment with multiple directories
- A simple test program that attempts to open a file not in the current directory

### Reproduction Process

1. **Set up DOSBox-Staging build environment**

   **Option A: Use Release Version**
   - Download from [DOSBox-Staging Releases](https://github.com/dosbox-staging/dosbox-staging/releases)
   - Extract and run the pre-built executable
   
   **Option B: Compile from Source (Windows 11)**
   
   a. **Install build dependencies:**
      ```powershell
      # Visual Studio 2022 Community
      winget install --id Microsoft.VisualStudio.2022.Community --exact
      # During installation, select: "Desktop development with C++"
      
      # CMake (3.25+)
      winget install --id Kitware.CMake --exact
      
      # Python (3.12+)
      winget install --id Python.Python.3.12 --exact
      # Ensure "Add Python to PATH" is checked during install
      
      # Git
      winget install --id Git.Git --exact
      ```
   
   b. **Clone and configure:**
      ```powershell
      git clone https://github.com/dosbox-staging/dosbox-staging.git
      cd dosbox-staging
      cmake --preset debug-windows
      ```
   
   c. **Build:**
      ```powershell
      cmake --build --preset debug-windows
      ```
   
   d. **Verify:**
      ```powershell
      .\build\debug-windows\dosbox.exe --version
      ```

2. **Create test directory structure**
   ```
   Z:\>mkdir TESTAPP
   Z:\>mkdir TESTAPP\DATA
   Z:\>mkdir TESTAPP\RUNFROM
   Z:\>echo This is test data > TESTAPP\DATA\NEEDLE.TXT
   ```

3. **Attempt to use APPEND command (currently fails)**
   ```
   Z:\>APPEND Z:\TESTAPP\DATA
   Bad command or file name
   ```

4. **Attempt to access file from different directory (fails)**
   ```
   Z:\>cd TESTAPP\RUNFROM
   Z:\TESTAPP\RUNFROM\>type NEEDLE.TXT
   File not found
   ```

5. **Expected behavior with basic APPEND functionality** (after implementation)
   ```
   Z:\TESTAPP\RUNFROM\>APPEND Z:\TESTAPP\DATA
   Z:\TESTAPP\RUNFROM\>type NEEDLE.TXT
   This is test data
   ```

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

### Testing Strategy

1. Verify APPEND command is recognized
2. Test path configuration (set, clear, display)
3. Test transparent file lookup across directories
4. Verify INT 2Fh multiplex handler responses
5. Test error handling (file truly not found)
6. Automated test suite using provided testware

---

---

# Phase I

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

## References

- **Original Issue:** [dosbox-staging#4866](https://github.com/dosbox-staging/dosbox-staging/issues/4866)
- **Research Branch:** `iteration-1-denied-pr` (contains detailed technical analysis and implementation details)
- **Phase II Branch:** `iteration-2-phase2` (reproduction steps and implementation plan)
- **Upstream Fork:** [uturuncuayaku/dosbox-staging](https://github.com/uturuncuayaku/dosbox-staging)
