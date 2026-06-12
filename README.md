# Contribution 1, Can more DOS commands be added? (APPEND.EXE, SUBST.EXE, and JOIN.EXE)

![installer](https://private-user-images.githubusercontent.com/224465904/587993310-709a1c81-4673-4e49-913e-8211a302f6e1.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3ODEyMzE3NzcsIm5iZiI6MTc4MTIzMTQ3NywicGF0aCI6Ii8yMjQ0NjU5MDQvNTg3OTkzMzEwLTcwOWExYzgxLTQ2NzMtNGU0OS05MTNlLTgyMTFhMzAyZjZlMS5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwNjEyJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDYxMlQwMjMxMTdaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0zYmY1ZDQ3ZDhmNzFkNjZiYjg1N2E2ZmU2YWRjMzUxYmJlMWFlOTRkNTc2YTg3MzBkYjk3ZGU1ZDAyZWEyNDQ3JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCZyZXNwb25zZS1jb250ZW50LXR5cGU9aW1hZ2UlMkZwbmcifQ.6B3x9V8k02batr6a1KqKs0Sy5RAGjzVDnzBXNrnokxI)

**Contribution Number:** 1
**Student:** Andres
**Issue:** https://github.com/dosbox-staging/dosbox-staging/issues/4866#issuecomment-4685769935
**Status:** Phase I [In Progress]

## Why I Chose This Issue

I'm choosing this issue because the maintainers and supporters have given hints to how they would like this solved, and I believe in giving back to communities that promote fun and engaging activities to do on PC's because that was my first exposure to computers. I particularly would like to understand how the emulator works so well with the current command set and help new games work within this repository.

## Understanding the Issue

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
