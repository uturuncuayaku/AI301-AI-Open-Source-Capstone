# Phase I
### DOSBox-staging, Can more DOS commands be added? (APPEND.EXE, SUBST.EXE, and JOIN.EXE)  
--- 
![installer_screen](https://raw.githubusercontent.com/uturuncuayaku/AI301-AI-Open-Source-Capstone/refs/heads/main/WINCMDR_INSTALL_ERROR.png)  

**Student:** Andres  
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866#issuecomment-4685769935  
**Status:** Phase II [In Progress]  

## Why I Chose This Issue

I'm choosing this issue because the maintainers and supporters have given hints to how they would like this solved, and I believe in giving back to communities that promote fun and engaging activities to do on PC's because that was my first exposure to computers. I particularly would like to understand how the emulator works so well with the current command set and help new games work within this repository.

## Understanding the Issue
A DOS game expects an entire DOS environment. Although modern x86-64 processors remain compatible with much of the original Intel 8086 instruction set. The 8086 instruction set remained a selling point for the modern CPU to take off but now is left for backward-compatibility. The difficult bit is to emulate the machine around the CPU. The emulator takes CPU instruction and converts it to C++ code, while implementing the memory access of a 1970s-1990s era PC. This emulator is an important aspect of the game playing system because games running strictly off the host PC would run with too-permissive options and might disable interrupts, program hardware ports, manipulate memory directly, or install interrupt handlers and become unstable or become a security issue for the rest of the system's users. So, emulating this environment with C++ is very important.

Now that we are up to speed with the current landscape of the MS-DOS emulator solutions, we can look at the source code to patch in our three commands needed for the particular game mentioned above. With this background, we can examine how DOSBox-Staging currently implements DOS commands and investigate how support for APPEND.EXE, and JOIN.EXE could be added to improve compatibility with installers and applications that depend on these utilities. I have found that SUBST command has already been implemented within the emulator but we will continue with the rest of the issue's request as that is part of the installation requirements dictacted by the game's installer.

#### DOS Game

- INT 21h (DOS API)
- INT 10h (Video BIOS)
- INT 13h (Disk BIOS)
- INT 16h (Keyboard BIOS)
- EMS/XMS memory
- Sound Blaster
- AdLib
- VGA hardware
- PIC interrupts
- PIT timers  

### Problem Description
The Origin Wing Commander Deluxe CD-ROM installer expects standard MS-DOS utilities such as APPEND.EXE, SUBST.EXE, and JOIN.EXE to be available during installation. DOSBox Staging currently provides some DOS utilities but does not implement these commands. As a result, the installer cannot complete successfully, reducing compatibility with legacy DOS software.

### Expected Behavior
The installer finds the commands and installs the game.

### Current Behavior
The installer fails due to DOSBox not needing the commands for any other game.

### Affected Components

- DOSBox command interpreter (internal commands and executable utilities)
- DOS-similar filesystem emulation layer
- Drive and path management subsystem
- DOS utility command implementations (similar to existing XCOPY|SUBST support)
- Compatibility layer for legacy DOS installers

---
# Phase II  
--- 

## Steps to Reproduce

### Reproduction Process

1. **Set up the build environment:** Compile DOSBox-Staging on Windows 11 Pro using Visual Studio 2022 BuildTools, PowerShell, and Ninja (Python3 is optional).
2. **Launch the game installer:** Mount the necessary local drives and attempt to run the Wing Commander installation sequence.
3. **Bypass the initial utility check:** Create empty placeholder files named `APPEND.EXE`, `JOIN.EXE`, and `SUBST.EXE` in the DOS environment to allow the installer to proceed past its dependency check.
4. **Trigger the executable:** Allow the installer to proceed until it attempts to execute the game using the generated command: `WC.EXE CD="D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT;" DD="C:\ORIGIN\WINGCMDR"`
5. **Observe the failure state:** Note the resulting crash and error message: `Redirected Exec failed -- "WC.EXE"`. The failure occurs because the game requires the relative file lookup behavior and semicolon-separated search paths that the actual `APPEND` command provides, which dummy executables cannot replicate.

### Solution Approach

**Understand**
Legacy games like Wing Commander require the `APPEND` command to locate data files across multiple directories using relative paths. Currently, the game crashes during execution because it attempts to use a CD-ROM search path (`D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT`) that relies on `APPEND` for proper file lookup. The goal is to build an MVP that stores these semicolon-separated paths and routes file requests through them when a file isn't found in the current working directory, avoiding a full 1:1 MS-DOS feature replication in favor of targeted game compatibility.

**Match**
This behavior is highly similar to the internal `PATH` environment variable parsing and the `mount` command (which already maps `SUBST` functionality). I will look at how the DOSBox command engine parses `PATH` variables and how it handles virtual filesystem lookups in `src/dos/` or the internal shell command handlers to mirror that logic for `APPEND`.

**Plan**
1. **Define State:** Create an internal structure (`struct AppendState { bool enabled = false; std::vector<std::string> paths; };`) to hold the global state of the command.
2. **Command Registration:** Register `APPEND` as an internal shell command.
3. **Parse Arguments:** Implement logic to parse semicolon-separated directories from the user's input.
4. **Read/Write Behavior:** * If executed with paths (e.g., `APPEND C:\GAMES\WC;D:\ORIGIN\WC\GAMEDAT`), populate the `AppendState` vector.
    * If executed with `;`, clear the vector.
    * If executed without arguments, print the currently stored paths to the console.
5. **Filesystem Hook:** Modify the file lookup routine (where `fopen` or internal DOS file opening occurs) to check the directories stored in `AppendState` if the initial file lookup in the current directory fails.


## Implementation Plan
* [Feature Branch Link](https://www.google.com/search?q=https://github.com/uturuncuayaku/dosbox-staging/tree/feature-append)
* **Core Objective:** Develop a Minimum Viable Product (MVP) for the `APPEND` command that focuses purely on resolving the core file lookup functionality required by legacy games, rather than full MS-DOS feature parity.
* **State Management:** Define a C++ structure (`struct AppendState { bool enabled = false; std::vector<std::string> paths; };`) to maintain the global state of the command.
* **Command Parsing:** Update the command engine to intercept `APPEND` calls and parse any semicolon-separated directories provided in the arguments.
* **Read Behavior:** Configure the engine to output the currently stored paths (or an "APPEND is not configured" message) if the command is executed without arguments.
* **Write Behavior:** Configure the engine to update the `AppendState` vector when paths are provided (e.g., `APPEND C:\GAMES\WC;D:\ORIGIN\WC\GAMEDAT`) or clear the state if `APPEND ;` is executed, enabling the relative path linking necessary for game data access.


**Review**
I will self-review my code against the DOSBox-Staging `CONTRIBUTING.md` file, ensuring I adhere to modern C++ conventions, memory safety guidelines, and the project's specific commit message formatting rules before opening a Draft PR.

* **Unit Tests:** I will write automated tests to verify the string parsing of semicolon-separated paths and the clearing of state using `APPEND ;`.
* **End-to-End Verification:** I will manually run the Wing Commander installation sequence to confirm the `WC.EXE CD="..."` execution successfully locates the game data and proceeds without the `Redirected Exec failed` error.

---
# Phase III - Testing & Verification

## Implementation Progress

### Code and Test Enhancements
* **`src/dos/dos_append.cpp`** ([Link to file](file:///c:/Users/andtr/Documents/GitHub/dosbox-staging/src/dos/dos_append.cpp))
  * Fixed a potential logic flaw where paths that evaluated to empty strings after backslash stripping (e.g. a single backslash `"\"`) were still pushed to the active paths vector, turning the append state active. Added a strict `!trimmed_path.empty()` check prior to registration.
* **`tests/dos_files_tests.cpp`** ([Link to file](file:///c:/Users/andtr/Documents/GitHub/dosbox-staging/tests/dos_files_tests.cpp))
  * Created the new `DOS_Append` unit test suite to verify nominal path parsing, defensive input sanitization, and security path traversal validation.

### Key Commit
* **Commit Hash**: `6be83f4791c8e06c9b7b06e96f7b950dba7b754b`
* **Message**: `Fixing file path implementation` (Committed both the parser check and the test suite).

---

## Challenges Faced

1. **Drive Isolation in Standalone Test Fixtures**: 
   Initially, using standard drive `C:` inside the traversal tests caused immediate failures because `C:` is not mounted in the GoogleTest console environment (only drive `Z:` is).
   * **Solution**: Dynamically instantiated a temporary `localDrive` pointing to the test assets folder and registered it inside the global `Drives` array at index `2` (`Drives.at(2) = local_drive`). This simulated a real cross-drive mount environment end-to-end, enabling complete test validation across arbitrary drives.
2. **Buffer Limits and Undefined Behavior Prevention**: 
   Safeguarding against undefined behavior when executing C-style operations (like `.pop_back()` on empty strings) required implementing pre-checks inside `SetPaths`.

---

## Testing Strategy

We added 3 major GoogleTest test cases targeting the `DOS_Append` namespace:
1. **`DOS_Append_Path_Parsing_Nominal`**: Tests normal semicolon-separated path inputs and trailing slash stripping.
2. **`DOS_Append_Weird_And_Invalid_Inputs`**: Tests robustness under malformed string inputs, whitespace padding, duplicate delimiters, and single characters.
3. **`DOS_Append_Security_And_Boundary_Tests`**:
   * **Path Traversal & Sandbox Escape**: Appends traversal paths (`C:\GAMES\..\..\..\ETC`) and validates that `DOS_MakeName` constrains the lookup to the drive root, preventing sandbox escapes.
   * **Buffer Sizing Boundaries**: Tests paths of exactly `DOS_PATHLENGTH - 1` (valid) and `DOS_PATHLENGTH` (invalid / fails gracefully) to protect against off-by-one errors.
   * **Resource Limits / DoS**: Passes 500 consecutive semicolons to verify no crashes or resource exhaustion.

### Validation Results
All 3 tests compiled and passed successfully:
```text
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from DOS_FilesTest
[ RUN      ] DOS_FilesTest.DOS_Append_Path_Parsing_Nominal
[       OK ] DOS_FilesTest.DOS_Append_Path_Parsing_Nominal (94 ms)
[ RUN      ] DOS_FilesTest.DOS_Append_Weird_And_Invalid_Inputs
[       OK ] DOS_FilesTest.DOS_Append_Weird_And_Invalid_Inputs (16 ms)
[ RUN      ] DOS_FilesTest.DOS_Append_Security_And_Boundary_Tests
[       OK ] DOS_FilesTest.DOS_Append_Security_And_Boundary_Tests (17 ms)
[----------] 3 tests from DOS_FilesTest (128 ms total)

[==========] 3 tests from 1 test suite ran. (128 ms total)
[  PASSED  ] 3 tests.
```

---

## Branch Link
* [Feature Branch Link on Fork](https://github.com/uturuncuayaku/dosbox-staging/tree/feature-append)

**Phase III Complete.**

