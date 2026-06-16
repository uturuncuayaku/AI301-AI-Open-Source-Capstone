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

- DOS command interpreter (internal commands and executable utilities)
- DOS filesystem emulation layer
- Drive and path management subsystem
- DOS utility command implementations (similar to existing XCOPY support)
- Compatibility layer for legacy DOS installers

## Reproduction Process
### Environment Setup  
- Visual Studio 2022 BuildTools
- Powershell
- Windows 11 Pro
- Ninja
- Python3 (optional)
  
---
## Debugging the game
- Creating placeholder APPEND.EXE/JOIN.EXE/SUBST.EXE allows installation to proceed further, but execution later fails when the game attempts to use CD-ROM search paths (D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT) that APPEND would normally provide.

- Then this command will start the game, `WC.EXE CD="D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT;" DD="C:\ORIGIN\WINGCMDR"`
- Integrating APPEND is the first step because I have deduced that SUBST is available through the DOSBox command engine and mapped to the "mount" command enabling proper functionality to play most games. This game in particular requires the path to the game data be local to the disk where the game is played. So, "append" enables us to create this relative link so the game on the disk accesses data from the hard disk using relative paths(/this/is/a/relativepath instead of canonical paths(full pathname, C:\ORIGIN\GAMEDATA\).


After bypassing the initial utility check, the installer attempted to launch `WC.EXE` and reported:

```text
Redirected Exec failed --
"WC.EXE"
CD="D:\ORIGIN\WC;D:\ORIGIN\WC\GAMEDAT;"
DD="C:\ORIGIN\WINGCMDR"
```

The presence of multiple semicolon-separated search paths is notable because APPEND was historically used to allow DOS applications to locate data files across multiple directories without requiring explicit path references.

This suggests that the issue may extend beyond simple command recognition and into file lookup behavior.

---

# Proposed minimaly viable product(MVP) implementation for APPEND command

The goal of the MVP is to provide meaningful compatibility improvements while minimizing implementation complexity and risk.

The objective is **not** to fully replicate every APPEND feature available in MS-DOS, but rather to implement the core functionality most likely required by games and installers.

## Supported Commands

```dos
APPEND
APPEND [path[;path...]]
APPEND ;
```

Examples:

```dos
APPEND C:\GAMES\WC
```

```dos
APPEND C:\GAMES\WC;D:\ORIGIN\WC\GAMEDAT
```

```dos
APPEND ;
```

```dos
APPEND
```

---

# Proposed Behavior

## 1. Display Current APPEND State

When executed without arguments:

```dos
APPEND
```

DOSBox Staging should display the currently configured APPEND paths.

Example:

```text
APPEND paths:
C:\GAMES\WC
D:\ORIGIN\WC\GAMEDAT
```

If no paths are configured:

```text
APPEND is not configured.
```

---

## 2. Store APPEND Search Paths

When executed with one or more paths:

```dos
APPEND C:\GAMES\WC;D:\ORIGIN\WC\GAMEDAT
```

DOSBox Staging should:

* Enable APPEND support.
* Parse semicolon-separated directories.
* Store those directories in a global APPEND state.

Example internal representation:

```cpp
struct AppendState {
    bool enabled = false;
    std::vector<std::string> paths;
};
```

---

## 3. Clear APPEND Configuration

When executed as:

```dos
APPEND ;
```

DOSBox Staging should:

* Clear all stored APPEND paths.
* Disable APPEND search behavior.

---

## 4. APPEND File Lookup Behavior

The primary purpose of APPEND was to extend file lookup behavior.

If a DOS application requests:

```text
PILOT.DAT
```

and the file cannot be found through normal lookup rules, DOSBox Staging should attempt to search configured APPEND directories before reporting failure.

Conceptually:

```text
Normal file lookup
        |
        v
File not found
        |
        v
Search APPEND paths
        |
        +--> Found -> Open file
        |
        +--> Not Found -> Return original error
```

This behavior would likely satisfy the majority of DOS applications that historically relied on APPEND.

---

# Architectural Considerations

Based on investigation of the DOSBox Staging codebase, the most promising implementation location appears to be the file resolution layer used during file open operations.

Relevant areas explored:

* `shell_cmds.cpp`
* `CMD_SUBST`
* `DOS_OpenFile()`
* `DOS_MakeName()`
* `localDrive::FileOpen()`

The current understanding is:

* The shell command should only configure APPEND state.
* The actual file search behavior should occur during file resolution.
* Existing file lookup behavior should always take precedence.
* APPEND should only be consulted when normal lookup fails.

This minimizes behavioral changes while preserving compatibility.

---

# Deferred Features

The following APPEND options are intentionally excluded from the MVP:

```dos
APPEND /X
APPEND /E
APPEND /PATH:ON
APPEND /PATH:OFF
```

These options should be considered future enhancements.

At present there is no evidence that Wing Commander Deluxe requires them.

Implementation effort should be focused on functionality that directly improves game compatibility.

---

# Validation Plan

The proposed implementation should be validated against real software rather than synthetic tests alone.

## Primary Validation Target

Wing Commander Deluxe CD-ROM

Success criteria:

* Installer no longer fails due to missing APPEND functionality.
* Installer completes successfully.
* Installed game can locate required data files.
* Runtime behavior matches expectations on a real DOS system.

## Additional Validation

* Verify APPEND path storage.
* Verify APPEND path clearing.
* Verify fallback file lookup behavior.
* Verify existing file lookup behavior remains unchanged when APPEND is disabled.

---

# Expected Benefits

This MVP would:

* Improve compatibility with legacy DOS installers.
* Improve compatibility with CD-ROM based games.
* Provide a foundation for future APPEND enhancements.
* Avoid premature implementation of rarely used MS-DOS features.
* Follow DOSBox Staging's existing compatibility-focused design philosophy.

Most importantly, it targets behavior that appears to be exercised by real software rather than implementing unsupported features speculatively.

