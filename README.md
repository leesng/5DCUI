5DCUI
==================

5DCUI is a graphical user interface for 5D Chess, built using SFML 3.0.2. It is currently integrated with the 5dchess_engine 0.3.4 or later as the core chess logic engine. The project also utilizes two open-source components: minifiledialog for file dialogs and nanosvg for SVG image parsing, providing a smooth cross-platform 5D chess playing and game management experience.

### Features

1. Button Controls
-  Real-time game status display;
-  Backward/Forward navigation through game history;
-  Legal move hinting for the current position;
-  Load external game files (5DPGN);
-  Save current game state;

2. Scrollable Text Boxes
-  Display and selection of the complete game move list;
-  Display and selection of variation branches for the current position;

3. Mouse Interactions
-  Left Click: Select a piece, select a target move;
-  Right Click: Deselect, undo the last move;
-  Left Double Click: Confirm move and smoothly pan camera to follow;
-  Scroll Wheel: Zoom board view; Middle Click (Wheel Press): Recenter camera;
-  Left Drag: Freely pan the board canvas position;

### Dependencies

-  SFML: Graphics, window, and system library , v3.0.2 or later
-  5dchess_engine: Core 5D chess logic engine , v0.3.4 or later
-  minifiledialog: Lightweight file dialog component
-  nanosvg: Minimalist SVG image parser
-  CMake 3.20+: Build tool
-  C++20/23 Compiler: e.g., GCC 13+, Clang 16+, or MSVC 2022+

### Build Guide

Before building, ensure the project structure is as follows:

5DCUI/
+-- CMakeLists.txt
+-- src/                    # UI Source code
+-- extern/
    +-- SFML3/         # SFML Library
    +-- 5dchess_engine/ # Engine Source code

1. Windows (example MSVC 2022) Build

-Create Build Directory

mkdir build
cd build

-Debug Version:
cmake ..
cmake --build .

-Release Version:
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --config Release

After compilation, the executable is located in the build/bin/ directory.

2. other OS (example macOS / Linux ) Build

mkdir build
cd build
cmake ..
cmake --build .


