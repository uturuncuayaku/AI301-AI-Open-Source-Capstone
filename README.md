# DOSBox-Staging Issue
## Can more DOS commands be added? (APPEND.EXE, SUBST.EXE, and JOIN.EXE)  

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

## Reproduction Evidence

* [Feature Branch Link](https://www.google.com/search?q=https://github.com/uturuncuayaku/dosbox-staging/tree/feature-append)

## Implementation Plan

* **Core Objective:** Develop a Minimum Viable Product (MVP) for the `APPEND` command that focuses purely on resolving the core file lookup functionality required by legacy games, rather than full MS-DOS feature parity.
* **State Management:** Define a C++ structure (`struct AppendState { bool enabled = false; std::vector<std::string> paths; };`) to maintain the global state of the command.
* **Command Parsing:** Update the command engine to intercept `APPEND` calls and parse any semicolon-separated directories provided in the arguments.
* **Read Behavior:** Configure the engine to output the currently stored paths (or an "APPEND is not configured" message) if the command is executed without arguments.
* **Write Behavior:** Configure the engine to update the `AppendState` vector when paths are provided (e.g., `APPEND C:\GAMES\WC;D:\ORIGIN\WC\GAMEDAT`) or clear the state if `APPEND ;` is executed, enabling the relative path linking necessary for game data access.
