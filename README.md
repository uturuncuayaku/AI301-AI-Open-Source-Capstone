# DOSBox-Staging APPEND Command Implementation

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866  
**Status:** Phase II - In Progress

---

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
   Z:\>mkdir C:\TESTAPP
   Z:\>mkdir C:\TESTAPP\DATA
   Z:\>mkdir C:\TESTAPP\RUNFROM
   Z:\>echo This is test data > C:\TESTAPP\DATA\NEEDLE.TXT
   ```

3. **Launch DOSBox-Staging and mount test directory**
   ```
   mount C <path-to-testapp>
   C:
   ```

4. **Attempt to use APPEND command**
   ```
   C:\>APPEND C:\TESTAPP\DATA
   ```
   **Result:** Command not recognized error (APPEND is not implemented)

5. **Attempt to access file across directories**
   ```
   C:\>cd C:\TESTAPP\RUNFROM
   C:\TESTAPP\RUNFROM\>type NEEDLE.TXT
   ```
   **Result:** File not found error (file exists in DATA\ but is not accessible from RUNFROM\)

6. **Expected behavior with basic APPEND functionality** (after implementation)
   ```
   C:\TESTAPP\RUNFROM\>APPEND C:\TESTAPP\DATA
   C:\TESTAPP\RUNFROM\>APPEND
   APPEND=C:\TESTAPP\DATA
   C:\TESTAPP\RUNFROM\>type NEEDLE.TXT
   This is test data
   ```
   **Expected:** File lookup succeeds transparently in appended directory

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
