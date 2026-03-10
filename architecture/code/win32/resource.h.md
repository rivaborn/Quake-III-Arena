# code/win32/resource.h

## File Purpose
Auto-generated Windows resource identifier header for the Quake III Arena Win32 build. It defines numeric IDs for embedded Win32 resources (icons, bitmaps, cursors, strings) referenced by `winquake.rc`.

## Core Responsibilities
- Define symbolic integer constants for Win32 resource IDs (icons, bitmaps, cursors, string tables)
- Provide APSTUDIO bookkeeping macros so Visual Studio's resource editor knows the next available ID values for each resource category
- Act as the bridge between the `.rc` resource script and C/C++ source code that references resources by name

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header of `#define` constants.

### Notes
- All symbols are simple preprocessor integer constants; no functions or data structures are present.
- The `//{{NO_DEPENDENCIES}}` comment is a Visual Studio resource compiler marker suppressing dependency scanning on this file.
- The `APSTUDIO_INVOKED` / `APSTUDIO_READONLY_SYMBOLS` guard block is only active when the file is opened inside the Visual Studio resource editor; it is invisible to the C compiler.

## Control Flow Notes
Not applicable. This file participates in no runtime flow; it is consumed at compile/link time by the resource compiler (`rc.exe`) and any C/C++ translation unit that needs to reference a resource by ID.

## External Dependencies
- **Consumed by:** `code/win32/winquake.rc` (resource script referencing these IDs)
- **Potentially referenced by:** Win32 platform code in `code/win32/` that loads icons, cursors, or bitmaps via `LoadIcon`, `LoadCursor`, `LoadBitmap`, etc.
- No standard library includes; no external symbols are used or defined here.

| Resource Constant | Value | Kind |
|---|---|---|
| `IDS_STRING1` | 1 | String table entry |
| `IDI_ICON1` | 1 | Icon resource |
| `IDB_BITMAP1` | 1 | Bitmap resource |
| `IDB_BITMAP2` | 128 | Bitmap resource |
| `IDC_CURSOR1` | 129 | Cursor resource |
| `IDC_CURSOR2` | 130 | Cursor resource |
| `IDC_CURSOR3` | 131 | Cursor resource |
