<!--
  Copyright (C) 2026 Software Freedom Conservancy, Inc.

  This file is part of The Inform 6 Programmer's Guide.

  This document is free documentation: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by the
  Free Software Foundation, either version 3 of the License, or (at your
  option) any later version.

  This document is distributed in the hope that it will be useful, but
  WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
  or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for
  more details.

  You should have received a copy of the GNU General Public License along
  with this document. If not, see <https://www.gnu.org/licenses/>.
-->

# Chapter 34: Menus, Multimedia, and Extended Features

This chapter covers the library's support for status line customisation,
interactive menus, sound and graphics, file I/O, and undo. These features
span a range of complexity and virtual machine support: some, like status
line customisation and undo, work on both the Z-machine and Glulx with
minor differences; others, like file I/O, are available only on Glulx.
Where a feature differs between virtual machines, the relevant sections are
marked with **[Z-machine]** or **[Glulx]** tags.

## 34.1 Status Line Customisation

The status line is the top line of the display, conventionally showing the
current location name, score, and move count. The library draws it
automatically every turn, but a game can fully customise its content and
layout by replacing the `DrawStatusLine` routine.

### 34.1.1 The sline1 and sline2 Globals

Two global variables in `parser.h` (lines 190–191) control the values
displayed on the right side of the default status line:

```inform6
Global sline1;   ! score (or hours, for timed games)
Global sline2;   ! moves (or minutes, for timed games)
```

By default, the library updates these at the end of every turn:

- `sline1` is set to the current `score` (or the hours component of
  `the_time` if the game uses a real-time clock).
- `sline2` is set to the current `turns` (or the minutes component of
  `the_time`).

The `Statusline time;` directive (see §10.9) switches the library from
score/moves mode to time-of-day mode. When this directive is active, the
right side of the status line displays hours and minutes rather than
score and moves. This is independent of the real-time clock mechanism
described in §24.7 — the `Statusline` directive only controls display
formatting, not actual timekeeping.

A game can set `sline1` and `sline2` directly at any point to override the
displayed values. This is most commonly done in an `Initialise` or in
an `each_turn` entry point to show custom information.

### 34.1.2 The Default DrawStatusLine Routine

The default `DrawStatusLine` routine is defined in the library. Its behaviour depends on the virtual machine and, for
the Z-machine, on the Z-machine version number.

**[Z-machine]** On Z-machine **version 3**, the status line is drawn by the
interpreter itself, not by the game. The game communicates values to the
interpreter through header bytes: the interpreter reads the current room
name, score/time values, and the statusline-is-time flag from fixed
locations in the story file header. The `DrawStatusLine` routine in the
library therefore does nothing on V3 beyond setting these header values.
The Z-machine specification requires interpreters to display a status line
in V3, but its precise formatting is interpreter-dependent.

**[Z-machine]** On Z-machine **version 4 and later** (V4/V5/V7/V8), the
game is responsible for drawing its own status line. The default
implementation proceeds as follows:

1. `@split_window 1` — creates a one-line upper window for the status line.
2. `@set_window 1` — switches output to the upper window.
3. `@set_cursor 1 1` — positions the cursor at the top-left corner
   (1-based coordinates).
4. `style reverse` — enables reverse video for the status bar appearance.
5. The routine prints spaces across the full screen width (obtained via
   `ScreenWidth()`) to create a solid reverse-video bar.
6. `@set_cursor 1 1` — repositions to the start of the line.
7. The location name is printed on the left.
8. The score/moves or time display is printed on the right, right-justified
   by computing column positions from the screen width.
9. `style roman` — restores normal text style.
10. `@set_window 0` — switches output back to the main (lower) window.

The opcodes used — `@split_window`, `@set_window`, `@set_cursor`,
`@set_font` — are documented in §29.

**[Glulx]** On Glulx, `DrawStatusLine` uses the Glk windowing API. The
library maintains a global `gg_statuswin` that holds the Glk window
reference for the status window (created during initialisation by
`GGInitialise`). The default implementation:

1. Calls `glk_set_window(gg_statuswin)` to direct output to the status
   window.
2. Uses `glk_window_get_size(gg_statuswin, gg_arguments, gg_arguments+4)`
   to obtain the window dimensions in characters.
3. Calls `glk_window_clear(gg_statuswin)` to erase the previous contents.
4. Uses `glk_set_style(style_Subheader)` for formatting.
5. Positions text with `glk_window_move_cursor(gg_statuswin, col, row)`.
6. Prints the location name on the left and score/moves on the right,
   computing column offsets from the window width.
7. Calls `glk_set_window(gg_mainwin)` to return output to the main window.

For details on Glk windows and styles, see §30.8.

### 34.1.3 Screen Utility Routines

The library provides several platform-abstracted screen utility routines in
`parser.h` (lines 6201–6261) that are used internally by `DrawStatusLine`
and the menu system, and are available for game code:

**`ClearScreen(window)`** (`parser.h` line 6201) — clears the specified
window. On the Z-machine, this compiles to `@erase_window window`. On
Glulx, it calls `glk_window_clear()` on the appropriate Glk window. The
`window` parameter is 0 for the main window and 1 for the status window.

**`MoveCursor(line, column)`** (`parser.h` line 6220) — positions the
cursor to the given line and column within the upper (status) window.
Coordinates are **1-based**: `MoveCursor(1, 1)` is the top-left corner.
On the Z-machine, this compiles to `@set_cursor line column`. On Glulx,
it calls `glk_window_move_cursor(gg_statuswin, column-1, line-1)`,
adjusting for Glk's 0-based coordinates.

**`MainWindow()`** (`parser.h` line 6240) — switches output back to the
main (lower) window. On the Z-machine, this compiles to `@set_window 0`.
On Glulx, it calls `glk_set_window(gg_mainwin)`.

**`ScreenWidth()`** (`parser.h` lines 6253/6261) — returns the width of
the screen in characters.

**[Z-machine]** On the Z-machine, `ScreenWidth` reads the screen width
from header byte `$21` (address 33 decimal):

```inform6
[ ScreenWidth;
    return (HDR_SCREENWCHARS->0);
];
```

**[Glulx]** On Glulx, `ScreenWidth` queries the status window dimensions
via `glk_window_get_size`:

```inform6
[ ScreenWidth width;
    glk_window_get_size(gg_statuswin, gg_arguments, 0);
    width = gg_arguments-->0;
    return width;
];
```

### 34.1.4 Replacing DrawStatusLine

To provide a custom status line, use the `Replace` directive (see §32.1)
to override the library's default implementation. The replacement routine
must handle all windowing and cursor positioning itself.

The following example shows a typical custom status line for V5+ Z-code
games:

```inform6
Replace DrawStatusLine;

[ DrawStatusLine width;
    @split_window 1;
    @set_window 1;
    @set_cursor 1 1;
    style reverse;
    width = ScreenWidth();
    spaces width;
    @set_cursor 1 1;
    print " ", (name) location;
    @set_cursor 1 (width - 20);
    print "Score: ", sline1, " Moves: ", sline2;
    style roman;
    @set_window 0;
];
```

A portable replacement that works on both Z-machine and Glulx should use
conditional compilation:

```inform6
Replace DrawStatusLine;

[ DrawStatusLine width;
    #Ifdef TARGET_ZCODE;
    @split_window 1;
    @set_window 1;
    @set_cursor 1 1;
    style reverse;
    width = ScreenWidth();
    spaces width;
    @set_cursor 1 1;
    print " ", (name) location;
    @set_cursor 1 (width - 20);
    print "Score: ", sline1, "  Moves: ", sline2;
    style roman;
    @set_window 0;
    #Ifnot; ! TARGET_GLULX
    if (gg_statuswin == 0) return;
    glk_set_window(gg_statuswin);
    width = ScreenWidth();
    glk_window_clear(gg_statuswin);
    glk_window_move_cursor(gg_statuswin, 0, 0);
    print " ", (name) location;
    glk_window_move_cursor(gg_statuswin, width - 22, 0);
    print "Score: ", sline1, "  Moves: ", sline2;
    glk_set_window(gg_mainwin);
    #Endif;
];
```

A multi-line status line can be created by splitting the window to two or
more lines. On the Z-machine, change the argument to `@split_window`:

```inform6
Replace DrawStatusLine;

[ DrawStatusLine width;
    @split_window 2;
    @set_window 1;
    width = ScreenWidth();
    style reverse;
    @set_cursor 1 1; spaces width;
    @set_cursor 2 1; spaces width;
    @set_cursor 1 1;
    print " ", (name) location;
    @set_cursor 1 (width - 20);
    print "Score: ", sline1;
    @set_cursor 2 1;
    print " ", (string) narrative_description;
    @set_cursor 2 (width - 20);
    print "Moves: ", sline2;
    style roman;
    @set_window 0;
];
```

Note that multi-line status bars consume screen real estate and reduce the
space available for main text. On small-screen interpreters (particularly
V3 terminals), large status bars may be impractical.

### 34.1.5 Status Line and the Game Loop

The library calls `DrawStatusLine` at the start of every turn, just before
printing the prompt (see §16.3 for the main game loop). This means any
changes to `sline1`, `sline2`, or any other global used by a custom
`DrawStatusLine` will be reflected on the next turn. To force an immediate
redraw mid-turn (for example, after a score change), call
`DrawStatusLine()` explicitly.

## 34.2 Menu Systems

The library includes a built-in menu system that can display hierarchical
menus of information to the player. This is most commonly used for
in-game help systems, "about" pages, and hint menus. The menu system
adapts its interface to the capabilities of the interpreter: on capable
interpreters it uses windowed, cursor-addressed display; on minimal
interpreters it falls back to numbered text menus.

### 34.2.1 The DoMenu Entry Point

The main entry point is `DoMenu(menu_choices, EntryR, ChoiceR)`, which
dispatches to an appropriate menu implementation based on the virtual
machine and interpreter capabilities.

**Parameters:**

- `menu_choices` — a string containing the menu title and item labels
  separated by `^` (newline) characters. The first entry is the menu
  title; subsequent entries are the item labels.
- `EntryR` — a routine called when a menu item is highlighted or
  displayed. It receives the item number (1-based) and should print the
  item's text.
- `ChoiceR` — a routine called when the player selects a menu item. It
  receives the item number and should print the full content for that
  choice.

Both callback routines are called with a single argument: the 1-based
item number. `EntryR` is called to display each item in the menu list,
while `ChoiceR` is called when the player selects an item to read its
content.

### 34.2.2 Using DoMenu

The following example demonstrates a complete help menu system:

```inform6
[ HelpMenuR choice;
    switch (choice) {
        1: print "About this game...^";
        2: print "How to play...^";
        3: print "Credits...^";
    }
];

[ HelpChoiceR choice;
    switch (choice) {
        1: "This is a demonstration game.";
        2: "Type commands to interact with the world.";
        3: "Written by the author.";
    }
];

[ HelpSub;
    DoMenu("Help^About^How to play^Credits", HelpMenuR, HelpChoiceR);
];
```

To make this accessible, define a verb:

```inform6
Verb meta 'help' 'info'
    *                          -> Help;
```

The menu title (the first `^`-delimited segment, "Help" in this example)
appears at the top of the menu display. The remaining segments become the
selectable items. The number of items is determined by counting `^`
separators in the string.

### 34.2.3 The LowKey_Menu Fallback

`LowKey_Menu(menu_choices, EntryR, ChoiceR)` (`verblib.h` lines 766–810)
provides a text-only menu fallback. It is used automatically on Z-machine
V3 (which has no upper window support) and on interpreters that lack
cursor-addressing capabilities. It can also be called directly by game
code that wants a simple numbered-choice interface.

The routine works as follows:

1. Counts the number of menu items by scanning for `^` characters.
2. Prints the menu title.
3. Displays each item with a number prefix (1, 2, 3, ...).
4. Prompts the player to type a number or Q to quit.
5. Reads single-character input using `KeyCharPrimitive()`.
6. If the input is Q or q, the menu exits.
7. If the input is a valid item number, `ChoiceR` is called with that
   number.
8. The menu loops, redisplaying after each selection until the player
   quits.

This implementation requires no special interpreter capabilities — it
works with plain sequential text output and basic keyboard input.

### 34.2.4 Z-machine Windowed Menu

**[Z-machine]** On V5 and later (`verblib.h` lines 816–926), `DoMenu`
uses a graphical windowed menu that takes over the full screen. The
implementation proceeds through several phases:

**Screen setup:**
- `@erase_window $ffff` clears all windows, resetting to a blank screen.
- `@split_window height` creates an upper window large enough to display
  the menu items plus a title bar and navigation instructions.
- `@set_window 1` directs output to the upper window.

**Menu rendering:**
- The title is printed centred at the top of the upper window using
  `@set_cursor` for positioning.
- Each menu item is printed on a separate line. The currently highlighted
  item is displayed in reverse video (`style reverse`), while other items
  are printed in normal style.
- If there are more items than fit on one page, page indicators
  (such as "[MORE]" or arrows) are shown.

**Input handling:**
- `@read_char 1` reads a single keypress without echoing it.
- Navigation keys:
  - **N** or **Down arrow** — move highlight to the next item.
  - **P** or **Up arrow** — move highlight to the previous item.
  - **Return/Enter** — select the highlighted item, calling `ChoiceR`.
  - **Q** or **Escape** — exit the menu and return to the game.
  - **N** (when at bottom) — advance to the next page (if paginated).
  - **P** (when at top) — go back to the previous page (if paginated).

**Selection display:**
When the player selects an item, the lower window is used to display the
full content returned by `ChoiceR`. The player presses a key to return to
the menu listing.

**Cleanup:**
When the player exits the menu (Q or Escape), the routine restores the
normal screen layout:
- `@erase_window $ffff` clears both windows.
- `@split_window 1` restores the standard one-line status bar.
- `DrawStatusLine()` is called to redraw the status line.
- A `<Look>` action is generated to redisplay the current room.

### 34.2.5 Glulx Menu

**[Glulx]** The Glulx menu implementation (`verblib.h` lines 932–1036)
uses the Glk API for all display operations. The structure mirrors the
Z-machine version but uses Glk calls:

**Window management:**
- `glk_window_clear(gg_mainwin)` clears the main window.
- `glk_set_window(gg_statuswin)` selects the status window for menu
  drawing.
- `glk_window_get_size(gg_statuswin, gg_arguments, gg_arguments+4)`
  obtains the window dimensions for layout calculations.
- `glk_window_move_cursor(gg_statuswin, x, y)` positions text within
  the status window for each menu item.

**Styling:**
- `glk_set_style(style_Subheader)` is used for the menu title and
  highlighted items.
- `glk_set_style(style_Normal)` is used for non-highlighted items and
  body text.

**Input:**
- `KeyCharPrimitive()` handles input through the Glk event system,
  calling `glk_request_char_event()` and waiting for a `evtype_CharInput`
  event. This provides the same single-keypress interface as `@read_char`
  on the Z-machine.

**Cleanup:**
On exit, the routine clears both windows, redraws the status line via
`DrawStatusLine()`, and generates a `<Look>` action to restore the room
description.

### 34.2.6 Menu Design Considerations

The menu string passed to `DoMenu` encodes all item labels in a single
string with `^` separators. This imposes some practical constraints:

- The total string length must fit in the compiler's string space.
- Item labels should be short enough to fit on one line; the menu
  system does not wrap long labels.
- The number of items per page depends on the screen height. On small
  screens, pagination activates automatically.

For complex multi-level menus, the `ChoiceR` routine can itself call
`DoMenu` recursively, creating nested submenus. Each level of nesting
pushes a new menu onto the screen, and exiting returns to the parent:

```inform6
[ TopMenuChoiceR choice;
    switch (choice) {
        1: DoMenu("Commands^Movement^Objects^Talking",
                   SubMenuR, SubMenuChoiceR);
        2: "This game was created as a demonstration.";
    }
];
```

## 34.3 Sound and Graphics

Both the Z-machine and Glulx support sound effects and graphical images,
but through entirely different mechanisms. Z-machine multimedia is
accessed through dedicated opcodes, while Glulx uses the Glk API.
In both cases, the game must be packaged with resource files containing
the actual sound and image data (typically in a Blorb archive).

### 34.3.1 Z-machine Sound and Graphics

**[Z-machine]** Sound effects are available from V3 onward. Graphics are
restricted to V6 (the graphical version of the Z-machine, rarely used in
practice).

**Sound effects:**

The `@sound_effect` opcode plays, stops, or manages sound resources:

```
@sound_effect number effect volume routine
```

| Parameter  | Description |
|------------|-------------|
| `number`   | Sound resource number (indexes into the Blorb sound resources). |
| `effect`   | Operation: 1 = prepare, 2 = start, 3 = stop, 4 = finish all. |
| `volume`   | Volume level: 1–8 (1 = quietest, 8 = loudest), or -1 for maximum. On V5+ the operand is a single word: the low byte holds the volume (1–8) and the high byte holds the number of repeats (0 = once, 255 = loop forever). When only volume is needed, pass the value directly (1–8). |
| `routine`  | Callback routine address (V5+ only), called when the sound finishes playing. Pass 0 for no callback. |

Examples:

```inform6
! Play sound resource 3 at volume 8, no callback
@sound_effect 3 2 8 0;

! Stop sound resource 3
@sound_effect 3 3;

! Play with a completion callback (V5+)
@sound_effect 5 2 6 SoundDone;
```

Not all interpreters support sound. Games should degrade gracefully when
sound is unavailable: the Z-machine specification does not provide a
standard capability query for sound in versions before V5. In V5+, the
header flags byte at address `$01` indicates interpreter capabilities,
but sound support flags are not universally reliable.

**Graphics (V6 only):**

V6 provides opcodes for drawing and managing pictures:

- `@draw_picture pic_num y x` — draws picture `pic_num` at position
  (`y`, `x`) in the current window. Coordinates are in screen units
  (pixels in most V6 interpreters).
- `@erase_picture pic_num y x` — erases the area occupied by picture
  `pic_num` at the given position, filling it with the background colour.
- `@picture_data pic_num array -> result` — stores the height and width
  of picture `pic_num` into `array-->0` and `array-->1` respectively.
  If `pic_num` is 0, stores the number of available pictures and the
  release number of the picture file. Returns true if the picture exists.
- `@picture_table table` — hints to the interpreter that the pictures
  listed in `table` (a word array of picture numbers, terminated by 0)
  should be preloaded into memory for fast display.

```inform6
! Draw picture 1 at the top-left corner
@draw_picture 1 1 1;

! Query picture dimensions
Array pic_info --> 2;
@picture_data 1 pic_info -> i;
if (i) {
    print "Height: ", pic_info-->0,
          " Width: ", pic_info-->1, "^";
}

! Preload pictures 1-3
Array preload_pics --> 1 2 3 0;
@picture_table preload_pics;
```

V6 games are uncommon because few interpreters fully support the V6
graphical model. Most modern Inform 6 games targeting graphical output
use Glulx instead.

### 34.3.2 Glulx Sound and Graphics

**[Glulx]** Glulx provides sound and graphics through the Glk API. Unlike the Z-machine's dedicated opcodes, Glk
uses an object-oriented model with windows, streams, and channels.

**Capability checking:**

Before using multimedia features, a game should query the interpreter's
capabilities using `glk_gestalt()`. The relevant gestalt constants are:

| Constant             | Value | Meaning |
|----------------------|-------|---------|
| `gestalt_Graphics`   | 6     | Can the interpreter display graphics at all? |
| `gestalt_DrawImage`  | 7     | Can images be drawn in the specified window type? |
| `gestalt_Sound`      | 8     | Can the interpreter play sounds? |
| `gestalt_SoundVolume` | 9    | Does the interpreter support volume control? |
| `gestalt_SoundNotify` | 10   | Does the interpreter send sound completion events? |
| `gestalt_Sound2`     | 21    | Does the interpreter support the Sound2 extension (MOD, OGG)? |

Usage:

```inform6
if (glk_gestalt(gestalt_Graphics, 0) == 0) {
    print "This interpreter cannot display graphics.^";
}
if (glk_gestalt(gestalt_Sound, 0) == 0) {
    print "This interpreter cannot play sounds.^";
}
```

The second argument to `glk_gestalt()` is a subquery parameter. For
`gestalt_DrawImage`, pass the window type constant (e.g.,
`wintype_TextBuffer` or `wintype_Graphics`) to check whether images can
be drawn in that specific window type.

**Graphics functions:**

- `glk_image_draw(win, image, val1, val2)` — draws image
  resource `image` in window `win`. For text buffer windows, `val1` and
  `val2` control alignment. For graphics windows, they specify the x,y
  position.
- `glk_image_draw_scaled(win, image, val1, val2, width, height)`
  — draws the image scaled to the specified `width` and
  `height` in pixels.
- `glk_image_get_info(image, width_ptr, height_ptr)` —
  retrieves the dimensions of image `image`, storing the width and
  height at the given addresses. Returns true if the image exists.

Example of drawing an image:

```inform6
[ ShowImage img win width height;
    if (glk_gestalt(gestalt_DrawImage, wintype_TextBuffer) == 0) return;
    if (glk_image_get_info(img, gg_arguments, gg_arguments+4)) {
        width = gg_arguments-->0;
        height = gg_arguments-->1;
        glk_image_draw(win, img, imagealign_MarginLeft, 0);
    }
];
```

**Sound channel functions:**

Sound in Glk is played through **sound channels**. A game creates one or
more channels, then plays sounds through them:

- `glk_schannel_create(rock)` — creates a new sound channel.
  The `rock` is an arbitrary identifier for the channel. Returns the
  channel reference, or 0 on failure.
- `glk_schannel_play(chan, sound)` — plays sound resource
  `sound` on channel `chan`. Returns true on success.
- `glk_schannel_play_ext(chan, sound, repeats, notify)` —
  extended play: `repeats` is the number of times to play (1 = once,
  -1 = loop forever), and `notify` is a non-zero value that will be
  included in the sound notification event when playback completes.
- `glk_schannel_stop(chan)` — stops any sound currently playing on
  `chan`.
- `glk_schannel_set_volume(chan, vol)` — sets the volume for `chan`.
  Volume is a linear scale where `$10000` (65536) is full volume and
  0 is silence.
- `glk_sound_load_hint(sound, flag)` — hints to the
  interpreter that sound resource `sound` should be preloaded (`flag`
  = 1) or may be unloaded (`flag` = 0). The interpreter may ignore
  this hint.

Complete sound example:

```inform6
Global background_channel;

[ InitSound;
    if (glk_gestalt(gestalt_Sound, 0) == 0) return;
    background_channel = glk_schannel_create(210);
];

[ PlaySound snd chan;
    if (glk_gestalt(gestalt_Sound, 0) == 0) return;
    chan = glk_schannel_create(0);
    if (chan) glk_schannel_play(chan, snd);
];

[ PlayBackgroundMusic snd;
    if (background_channel == 0) return;
    ! Loop the music indefinitely
    glk_schannel_play_ext(background_channel, snd, -1, 0);
    glk_schannel_set_volume(background_channel, $8000); ! 50% volume
];

[ StopBackgroundMusic;
    if (background_channel) glk_schannel_stop(background_channel);
];
```

**Sound notification events:**

If `glk_gestalt(gestalt_SoundNotify, 0)` returns true, the interpreter
sends an `evtype_SoundNotify` event when a sound finishes playing (only
if a non-zero `notify` value was passed to `glk_schannel_play_ext`). The
event's `val2` field contains the notification value. Games can handle
this in their `HandleGlkEvent` entry point (see §32.3) to trigger
actions when sounds complete.

### 34.3.3 Blorb Resource Packaging

Both Z-machine and Glulx games reference sounds and images by numeric
resource ID. The actual media data is typically stored in a **Blorb**
archive — a container format that bundles the game file with its
resources. The Blorb file maps resource numbers to embedded data chunks
(AIFF or OGG for sounds, PNG or JPEG for images).

The compiler does not create Blorb archives directly. A separate tool
such as `bres` or `iblorb` is used to combine the compiled game file
with resource files into a single `.zblorb` (Z-machine) or `.gblorb`
(Glulx) package.

Resource numbering must be consistent between the Blorb manifest and
the resource IDs used in game code. By convention, picture resources
start at 1 and sound resources also start at 1; the Blorb format
distinguishes them by chunk type.

## 34.4 File I/O (Glulx)

**[Glulx]** The Glk API provides a complete file I/O system, allowing
Glulx games to read and write external files. This capability has no
equivalent on the Z-machine, which supports only save/restore of game
state. File I/O can be used for persistent data storage, inter-game
communication, transcript output, and command recording.

### 34.4.1 File Mode and Usage Constants

File operations use mode and usage constants:

**File mode constants** control how a file is opened:

| Constant             | Value | Description |
|----------------------|-------|-------------|
| `filemode_Write`     | 1     | Open for writing; truncates existing file. |
| `filemode_Read`      | 2     | Open for reading only. |
| `filemode_ReadWrite`  | 3    | Open for both reading and writing. |
| `filemode_WriteAppend` | 5   | Open for writing; appends to existing file. |

**File usage constants** describe the type of file:

| Constant                | Value | Description |
|-------------------------|-------|-------------|
| `fileusage_Data`        | 0     | General-purpose data file. |
| `fileusage_SavedGame`   | 1     | Saved game state (used by save/restore). |
| `fileusage_Transcript`  | 2     | Transcript output file. |
| `fileusage_InputRecord`  | 3    | Recorded command input file. |
| `fileusage_TextMode`    | 256   | Text mode (line-ending conversion). Combine with `|`. |
| `fileusage_BinaryMode`  | 0    | Binary mode (no conversion). This is the default. |

The usage and mode values are combined with the bitwise OR operator. For
example, `fileusage_Data | fileusage_TextMode` opens a general-purpose
text file with platform-appropriate line-ending conversion.

### 34.4.2 File References

Before opening a file, a **file reference** (fileref) must be created.
A fileref represents the identity of a file on the host system without
actually opening it. There are three ways to create a fileref:

**`glk_fileref_create_by_prompt(usage, mode, rock)`** — prompts the
player to specify a filename through the interpreter's native file
dialog. This is the recommended method for any file the player should
be aware of, as it respects the interpreter's sandboxing and file
access policies. Returns 0 if the player cancels.

**`glk_fileref_create_by_name(usage, name, rock)`** — creates a fileref
with a specific filename. The interpreter may modify the filename for
security reasons (adding extensions, restricting to a safe directory).
Not all interpreters support this; those that do may impose restrictions
on the path.

**`glk_fileref_create_temp(usage, rock)`** — creates a fileref for a
temporary file. The interpreter chooses the filename and location. The
file is deleted when the fileref is destroyed or the game exits.

In all cases, the `rock` parameter is an arbitrary identifier that can
be used to find the fileref later through iteration.

After use, filerefs should be destroyed with `glk_fileref_destroy(fref)`
to release the associated system resource. Destroying a fileref does not
delete the underlying file or close any stream opened from it.

To check whether a file exists before attempting to read it, use
`glk_fileref_does_file_exist(fref)`, which returns true if the file
referenced by `fref` exists on the host system.

### 34.4.3 Stream Operations

Files are accessed through **streams**, which provide a uniform I/O
interface. The stream operations are:

**Opening streams:**

- `glk_stream_open_file(fileref, mode, rock)` — opens the
  file identified by `fileref` in the given mode. Returns a stream
  reference, or 0 on failure.
- `glk_stream_open_memory(buf, buflen, mode, rock)` — opens
  a memory buffer as a stream. Useful for in-memory string construction
  and formatting. `buf` is a byte array, `buflen` is its length.

**Closing streams:**

- `glk_stream_close(stream, result_ptr)` — closes the
  stream. If `result_ptr` is non-zero, the read and write counts are
  stored at that address.

**Positioning:**

- `glk_stream_set_position(stream, pos, seekmode)` — sets
  the read/write position. `seekmode` is 0 for absolute, 1 for relative
  to current position, 2 for relative to end.
- `glk_stream_get_position(stream)` — returns the current
  position in the stream.

**Reading and writing:**

- `glk_put_char_stream(stream, ch)` — writes a single character.
- `glk_put_string_stream(stream, str)` — writes a string.
- `glk_put_buffer_stream(stream, buf, len)` — writes `len` bytes from
  `buf`.
- `glk_get_char_stream(stream)` — reads a single character. Returns -1
  at end of file.
- `glk_get_buffer_stream(stream, buf, len)` — reads up to `len` bytes
  into `buf`. Returns the number of bytes actually read.
- `glk_get_line_stream(stream, buf, len)` — reads a line (up to `len`
  bytes) into `buf`. Returns the number of bytes read.

**Current stream:**

- `glk_stream_set_current(stream)` — sets the current output stream.
  All subsequent `print` statements will be directed to this stream.
  This is the Glulx equivalent of the Z-machine's `@output_stream`
  opcode. To restore normal output, set the current stream back to the
  main window's stream.

### 34.4.4 Complete File I/O Example

The following example demonstrates writing data to a file and reading it
back:

```inform6
[ SaveData fref str;
    fref = glk_fileref_create_by_prompt(
        fileusage_Data | fileusage_TextMode,
        filemode_Write, 0);
    if (fref == 0) rfalse;
    str = glk_stream_open_file(fref, filemode_Write, 0);
    glk_fileref_destroy(fref);
    if (str == 0) rfalse;
    glk_put_string_stream(str, "saved data here");
    glk_stream_close(str, 0);
    rtrue;
];
```

Reading data back:

```inform6
Array read_buffer -> 256;

[ LoadData fref str len;
    fref = glk_fileref_create_by_prompt(
        fileusage_Data | fileusage_TextMode,
        filemode_Read, 0);
    if (fref == 0) rfalse;
    if (glk_fileref_does_file_exist(fref) == 0) {
        glk_fileref_destroy(fref);
        rfalse;
    }
    str = glk_stream_open_file(fref, filemode_Read, 0);
    glk_fileref_destroy(fref);
    if (str == 0) rfalse;
    len = glk_get_buffer_stream(str, read_buffer, 256);
    glk_stream_close(str, 0);
    ! Process read_buffer contents (len bytes read)
    rtrue;
];
```

### 34.4.5 The Glk Wrapper and Output Streams

The veneer includes `Glk__Wrap`,
a wrapper routine that mediates between Inform's `print` statement and
the Glk I/O system. When the compiler targets Glulx, all `print`
output is routed through the current Glk stream. The `@glk` opcode
provides direct access to the Glk API; the veneer function translates
between the compiler's calling convention and the Glk dispatch layer.

On the Z-machine, `@output_stream` directs output to different
destinations (screen, transcript, table, or command file). On Glulx,
this is accomplished entirely through Glk stream management:

| Z-machine `@output_stream` | Glulx equivalent |
|-----------------------------|-------------------|
| Stream 1 (screen)          | `glk_set_window(gg_mainwin)` — output to main window. |
| Stream 2 (transcript)      | Open a file stream and set it as the current stream. |
| Stream 3 (table)           | `glk_stream_open_memory()` — capture output to a buffer. |
| Stream 4 (command file)    | Open a file stream with `fileusage_InputRecord`. |

## 34.5 Undo Support

The undo mechanism allows the player to reverse the most recent turn,
restoring the complete game state to what it was before the last command.
Both the Z-machine and Glulx provide hardware-level undo through
save/restore-state opcodes, and the library integrates undo into the
standard game loop.

### 34.5.1 Undo Opcodes

**[Z-machine]** The Z-machine provides two opcodes for undo (available
from V5 onward):

- `@save_undo -> result` (EXT opcode 0x09) — saves a snapshot of the
  entire game state (memory, stack, program counter) into an
  interpreter-managed buffer. The result is:
  - 1 if the save succeeded.
  - 0 if the save failed (interpreter does not support undo or ran out
    of memory).
  - 2 if the interpreter has just **restored** from this undo point
    (i.e., control has returned here after a `@restore_undo`).

- `@restore_undo -> result` (EXT opcode 0x0A) — restores the game state
  from the most recently saved undo snapshot. If successful, execution
  resumes at the point where `@save_undo` was called, with the result
  variable set to 2. If unsuccessful, execution continues normally and
  the result is 0.

**[Glulx]** Glulx provides equivalent opcodes with slightly different
result conventions:

- `@saveundo result` — saves a state snapshot. Returns 0 on success,
  1 on failure, and -1 when control returns after a restore.

- `@restoreundo result` — restores from the most recent snapshot. On
  success, execution resumes at the `@saveundo` point. On failure,
  returns 1.

The result conventions differ between the two virtual machines. The
library code accounts for this with conditional compilation, translating
the Glulx return values to match the library's internal convention.

### 34.5.2 Library Undo Globals

The library maintains two global variables in `parser.h` (lines 209–210)
to track undo state:

```inform6
Global undo_flag;     ! parser.h line 209
Global just_undone;   ! parser.h line 210
```

**`undo_flag`** records the current availability of undo:

| Value | Meaning |
|-------|---------|
| 0     | The interpreter does not support undo at all. |
| 1     | Undo is temporarily unavailable (the last save attempt failed, possibly due to memory pressure). |
| 2     | Undo is available — a valid undo snapshot exists. |

**`just_undone`** is a boolean flag that prevents the player from issuing
consecutive undo commands. It is set to 1 immediately after a successful
undo and reset to 0 at the start of the next turn. This prevents
multiple undos in a row, since each undo restores the state to the same
point (only one level of undo is maintained).

### 34.5.3 Saving Undo State in the Game Loop

The library saves an undo snapshot every turn as part of the main game
loop (`parser.h` lines 1384–1396). This code runs at the **start** of
each turn, before the player's command is read, so that the undo point
captures the state as it was at the beginning of the turn:

```inform6
#Ifdef TARGET_ZCODE;
@save_undo i;
#Ifnot; ! TARGET_GLULX
@saveundo i;
if (i == -1) { GGRecoverObjects(); i = 2; }
else i = (~~i);
#Endif;
just_undone = 0;
undo_flag = 2;
if (i == -1) undo_flag = 0;
if (i == 0) undo_flag = 1;
```

This code performs the following steps:

1. **Save the undo snapshot** using the appropriate opcode for the target
   platform.
2. **Handle Glulx restore return**: on Glulx, when `@saveundo` returns
   -1, it means the interpreter has just restored to this point (the
   equivalent of result 2 on the Z-machine). The library calls
   `GGRecoverObjects()` to re-establish Glk object references (window
   IDs, stream handles, etc.) that may have been invalidated by the
   state restore, then normalises the result to 2.
3. **Normalise Glulx results**: the `~~i` (bitwise NOT) converts the
   Glulx convention (0 = success) to the library's convention
   (non-zero = success).
4. **Reset `just_undone`** to 0, allowing undo on this turn.
5. **Set `undo_flag`** based on the result:
   - Result of -1 (Z-machine) → `undo_flag = 0` (no undo support).
   - Result of 0 (Z-machine) → `undo_flag = 1` (save failed this turn).
   - Otherwise → `undo_flag = 2` (undo available).

### 34.5.4 The PerformUndo Routine

When the player types UNDO, the parser recognises it as a special
meta-command and calls `PerformUndo` (`parser.h` lines 1499–1512).
This routine handles all the validation and error reporting:

1. **Check for game start**: if the turn count is at its initial value
   (comparing `turns` against the starting turn), the undo is rejected
   with library message `Miscellany` 11:
   *"You can't ~undo~ what hasn't been done!"*

2. **Check for consecutive undos**: if `just_undone` is true, the undo
   is rejected with library message `Miscellany` 12:
   *"Can't ~undo~ twice in succession. Sorry!"*

3. **Check undo availability**: if `undo_flag` is 0, the undo is
   rejected with library message `Miscellany` 6:
   *"Your interpreter does not provide ~undo~. Sorry!"*
   If `undo_flag` is 1, the undo is rejected with library message
   `Miscellany` 7:
   *"You cannot ~undo~ any further."*

4. **Attempt restore**: the appropriate restore opcode is executed:

   ```inform6
   #Ifdef TARGET_ZCODE;
   @restore_undo i;
   #Ifnot; ! TARGET_GLULX
   @restoreundo i;
   #Endif;
   ```

5. **Handle failure**: if the restore opcode returns (which means it
   failed — a successful restore never returns to this point), the
   routine prints `Miscellany` message 7.

6. **Handle success**: on a successful restore, execution resumes at the
   `@save_undo`/`@saveundo` call in the game loop (§34.5.3) with a
   result indicating restoration. The library detects this, sets
   `just_undone = 1`, and prints `Miscellany` message 13:
   *"Previous turn undone."*

### 34.5.5 Undo-Related Library Messages

The undo system communicates with the player through the `Miscellany`
library message set. These can be customised
through the standard library message interception mechanism
(see §32.4):

| Message Number | Default Text | When Used |
|----------------|-------------|-----------|
| 6  | "Your interpreter does not provide ~undo~. Sorry!" | `undo_flag == 0`: interpreter lacks undo support entirely. |
| 7  | "You cannot ~undo~ any further." | `undo_flag == 1`: undo save failed, or restore failed. |
| 11 | "You can't ~undo~ what hasn't been done!" | Player tries to undo before the first move. |
| 12 | "Can't ~undo~ twice in succession. Sorry!" | `just_undone` is set — consecutive undo attempt. |
| 13 | "Previous turn undone." | Successful undo restoration. |

The tilde characters (`~`) in these messages are Inform's escape
sequence for double-quote characters in strings (see §1.4).

### 34.5.6 Dictionary Words for UNDO

The words that the parser recognises as the UNDO command are defined
in `english.h` as dictionary word constants:

```inform6
Constant UNDO1__WD  = 'undo';
Constant UNDO2__WD  = 'undo';
Constant UNDO3__WD  = 'undo';
```

All three constants point to the same word by default. A language
definition file for another language can redefine these to provide
up to three synonyms for the undo command in the target language
(see §33.1 for the language definition file interface).

The parser checks for these words early in its main loop, before
full grammar parsing begins, treating UNDO as a special command
that bypasses the normal action processing pipeline. This is
necessary because undo must restore game state before any turn
processing occurs.

### 34.5.7 Programmatic Undo

A game can invoke undo programmatically by calling `PerformUndo()`
directly. This is occasionally useful for implementing a "restart
last puzzle" mechanic or an auto-undo triggered by certain events.
However, since only one level of undo state is maintained, calling
`PerformUndo()` more than once between saves will fail on the second
attempt.

A game can also prevent undo on specific turns by setting
`undo_flag = 0` in a `before` or `after` routine. This causes the
next undo attempt to report that the interpreter does not support
undo. To prevent undo globally, set `undo_flag = 0` in the
`Initialise` entry point and again at the start of each turn (in a
`TimePasses` or `each_turn` handler) to override the library's
automatic save.

For games that require multi-level undo (more than one turn back),
this must be implemented at the application level — for example, by
maintaining a stack of serialised game states using the file I/O
facilities described in §34.4 (Glulx only).
