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

This chapter covers the library's support for status line customization,
interactive menus, sound and graphics, file I/O, and undo. These features
span a range of complexity and virtual machine support: some, like status
line customization and undo, work on both the Z-machine and Glulx with
minor differences; others, like file I/O, are available only on Glulx.
Where a feature differs between virtual machines, the relevant sections are
marked with **[Z-machine]** or **[Glulx]** tags.

## 34.1 Status Line Customization

The status line is the top line of the display, conventionally showing the
current location name, score, and move count. The library draws it
automatically every turn, but a game can fully customize its content and
layout by replacing the `DrawStatusLine` routine.

### 34.1.1 The sline1 and sline2 Globals

Two global variables control the values
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

The default `DrawStatusLine` routine is defined in the library. Its behavior depends on the virtual machine and, for
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
reference for the status window (created during initialization by
`GGInitialise`). The default implementation:

1. Returns early if `gg_statuswin` is 0 (the interpreter did not provide
   a status window).
2. Calls `StatusLineHeight(gg_statuswin_size)` to ensure the status
   window is split to the requested number of lines.
3. Uses `MoveCursor(1, 1)` (which internally calls
   `glk_set_window(gg_statuswin)` and
   `glk_window_move_cursor(gg_statuswin, 0, 0)`).
4. Uses `ScreenWidth()` to obtain the window width in characters.
5. Prints `width` spaces to clear the line, then repositions the cursor
   and prints the location name on the left and score/moves on the
   right at column positions computed from the width.
6. Calls `MainWindow()` to return output to the main window (which
   internally calls `glk_set_window(gg_mainwin)`).

The default routine does not call `glk_set_style` explicitly; the
status window is a text-grid window and the interpreter renders it in
its native status-line style.

For details on Glk windows and styles, see §30.8.

### 34.1.3 Screen Utility Routines

The library provides several platform-abstracted screen utility routines
that are used internally by `DrawStatusLine`
and the menu system, and are available for game code:

**`ClearScreen(window)`** — clears the specified
window. On the Z-machine, this expands (via a switch) to
`@erase_window` with operand `-1`, `1`, or `0`. On Glulx, it calls
`glk_window_clear()` on the appropriate Glk window. The `window`
argument is one of the constants `WIN_ALL` (0, clears everything),
`WIN_STATUS` (1, the status window), or `WIN_MAIN` (2, the main
window).

**`MoveCursor(line, column)`** — positions the
cursor to the given line and column within the upper (status) window.
Coordinates are **1-based**: `MoveCursor(1, 1)` is the top-left corner.
On the Z-machine, this compiles to `@set_cursor line column`. On Glulx,
it calls `glk_window_move_cursor(gg_statuswin, column-1, line-1)`,
adjusting for Glk's 0-based coordinates.

**`MainWindow()`** — switches output back to the
main (lower) window. On the Z-machine, this compiles to `@set_window 0`.
On Glulx, it calls `glk_set_window(gg_mainwin)`.

**`ScreenWidth()`** — returns the width of
the screen in characters.

**[Z-machine]** On the Z-machine, `ScreenWidth` reads the screen width
from header byte `$21` (address 33 decimal):

```inform6
[ ScreenWidth;
    return (HDR_SCREENWCHARS->0);
];
```

**[Glulx]** On Glulx, `ScreenWidth` queries the currently active output
window (the status window when the library is drawing the status line,
otherwise the main window):

```inform6
[ ScreenWidth  id;
    id = gg_mainwin;
    if (gg_statuswin && statuswin_current) id = gg_statuswin;
    glk_window_get_size(id, gg_arguments, 0);
    return gg_arguments-->0;
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

- `menu_choices` — either a string or a routine. It supplies the body
  text printed in the menu pane (typically the list of selectable items
  as they appear on screen). If it is a string, the library prints it
  verbatim with `print (string) menu_choices`; if it is a routine, the
  library calls it with no arguments and lets it print whatever it
  likes.
- `EntryR` — a parameterless routine that the library uses to enumerate
  the menu and to label individual items. It communicates with the
  library through the globals `menu_item`, `item_name`, and
  `item_width`.
- `ChoiceR` — a parameterless routine called when the player selects
  an item. It is expected to print the body content for the selected
  item and return a control code (see below).

**Callback protocol.** All three menu globals are declared in
`parser.h`: `menu_item` (the currently active item, 0 while
enumerating), `item_name` (the displayed title, default `"---"`), and
`item_width` (its visible width in characters, default 8).

`EntryR` is called by the library in two situations:

- First, with `menu_item = 0`: the routine must set `item_name` and
  `item_width` to describe the *menu title* and **return the number of
  selectable items** in the menu.
- Later, with `menu_item = N` (1-based): the routine must set
  `item_name` and `item_width` to describe item `N`. The library uses
  these to draw the per-item heading shown while `ChoiceR` is
  printing.

`ChoiceR` is called with `menu_item` set to the selected item number.
It prints the long content for that item and returns one of:

- `1` (or fall through) — wait for a key, then redisplay the menu.
- `2` — redisplay the menu immediately (no "press any key" prompt).
- `3` — exit the menu entirely.

### 34.2.2 Using DoMenu

The following example demonstrates a complete help menu system:

```inform6
[ HelpEntryR;
    switch (menu_item) {
        0: item_name = "Help"; item_width = 4; return 3;
        1: item_name = "About";        item_width = 5;
        2: item_name = "How to play";  item_width = 11;
        3: item_name = "Credits";      item_width = 7;
    }
];

[ HelpChoiceR;
    switch (menu_item) {
        1: print "This is a demonstration game.^";
        2: print "Type commands to interact with the world.^";
        3: print "Written by the author.^";
    }
];

[ HelpSub;
    DoMenu("About^How to play^Credits^", HelpEntryR, HelpChoiceR);
];
```

The string `menu_choices` is what the player sees as the list of items
on the menu screen; the `^` characters in it are simply newlines in the
printed text, not delimiters parsed by the library. The number of items
is determined solely by the value returned from `EntryR` when
`menu_item == 0`.

To make this accessible, define a verb:

```inform6
Verb meta 'help' 'info'
    *                          -> Help;
```

The text drawn at the top of the menu is whatever `EntryR` writes into
`item_name` when called with `menu_item == 0`. The selectable items
themselves are whatever `menu_choices` prints (or, if `menu_choices` is
a routine, whatever that routine prints). The library does *not* parse
`^`-delimited segments of `menu_choices` to obtain the title or count
the items.

### 34.2.3 The LowKey_Menu Fallback

`LowKey_Menu(menu_choices, EntryR, ChoiceR)`
provides a text-only menu fallback. It is used automatically on
Z-machine V3 (which has no upper window support) and on any platform
where the global `pretty_flag` is zero (set by the screen-capability
check when the interpreter lacks cursor-addressing, or by the author).
On Glulx it is also used when `gg_statuswin` is 0. It can be called
directly by game code that wants a simple numbered-choice interface.

The routine works as follows:

1. Calls `EntryR()` once (with `menu_item = 0`) to obtain the title in
   `item_name` and the line count as the routine's return value.
2. Prints `--- title ---` followed by the body, which is the
   `menu_choices` parameter (printed as a string, or invoked as a
   routine if it is one).
3. Prints library message `Miscellany` 52 (the "Type a number..."
   prompt) and reads a line of input using `read buffer parse` on the
   Z-machine or `KeyboardPrimitive(buffer, parse)` on Glulx — *not*
   single-character `KeyCharPrimitive` input.
4. If the input is empty or its first word is `QUIT1__WD` /
   `QUIT2__WD`, the menu exits (decrementing `menu_nesting` and, at
   the outermost level, performing a `<Look>`).
5. If `TryNumber(1)` yields a valid item number in 1..lines,
   `menu_item` is set to that value and `ChoiceR()` is called; the
   loop continues based on `ChoiceR`'s return value (2 = redisplay
   title, 3 = exit).

This implementation requires no special interpreter capabilities — it
works with plain sequential text output and basic keyboard input.

### 34.2.4 Z-machine Windowed Menu

**[Z-machine]** On V5 and later, `DoMenu`
uses a graphical windowed menu that takes over the full screen. The
implementation proceeds through several phases:

**Screen setup:**
- `@erase_window $ffff` clears all windows, resetting to a blank screen.
- `@split_window height` creates an upper window large enough to display
  the menu items plus a title bar and navigation instructions.
- `@set_window 1` directs output to the upper window.

**Menu rendering:**
- The title (`item_name`, set by `EntryR` when called with
  `menu_item == 0`) is printed in reverse video, centred across the
  upper window using `@set_cursor` for positioning, with `item_width`
  used to compute the centre offset.
- Below the title, the library prints two reverse-video lines listing
  the navigation keys (`NKEY__TX`, `PKEY__TX`, `RKEY__TX`, and
  `QKEY1__TX` / `QKEY2__TX` from `english.h`).
- The body (`menu_choices` parameter) is then printed once, in normal
  style, with no reformatting by the library.
- A `>` cursor in the left margin marks the currently highlighted
  item; navigation moves the cursor up and down.

**Input handling:**
- `@read_char 1 -> pkey` reads a single keypress without echoing it.
- Navigation keys (compared against the constants in `english.h` plus
  the Z-machine function-key codes 129/130/131/132):
  - **N**, **Down arrow** (key code 130) — move highlight to the next
    item (wrapping from the last to the first).
  - **P**, **Up arrow** (key code 129) — move highlight to the previous
    item (wrapping from the first to the last).
  - **Return** (10 or 13) or **Select** (key code 132) — select the
    highlighted item, calling `EntryR` (to print the per-item heading)
    and then `ChoiceR`.
  - **Q**, **Escape** (key code 27), or **Function key** 131 — exit
    the menu and return to the game.

**Selection display:**
When the player selects an item, `EntryR` is called with `menu_item`
set to the chosen line, the screen is redrawn with `item_name` shown
as a one-line title, and `ChoiceR` is called in the lower window to
print the full content. If `ChoiceR` returns 2 the menu is
redisplayed immediately; if it returns 3 the menu exits; otherwise
the library prints `Miscellany` 53 (the "Press any key" prompt),
reads a key, and redisplays the menu.

**Cleanup:**
When the player exits the menu (Q, Escape, or function key 131), the
routine clears both windows with `@erase_window $ffff`, switches back
to window 0 with `@set_window 0`, and (at the outermost level of
`menu_nesting`) generates a `<<Look>>` to redisplay the current room.
It does *not* explicitly call `DrawStatusLine`; the status line is
redrawn automatically by the main game loop on the next turn.

### 34.2.5 Glulx Menu

**[Glulx]** The Glulx menu implementation
uses the Glk API for all display operations. The structure mirrors the
Z-machine version but uses Glk calls:

**Window management:**
- `glk_window_clear` is called on both `gg_statuswin` and `gg_mainwin`.
- `glk_set_window(gg_statuswin)` selects the status window for menu
  drawing.
- `StatusLineHeight(lines+7)` enlarges the status window to hold the
  title, the two-line key legend, a blank line, and the body items.
- `glk_window_get_size(gg_statuswin, gg_arguments, gg_arguments+4)`
  obtains the window dimensions for layout calculations.
- `glk_window_move_cursor(gg_statuswin, x, y)` positions text within
  the status window for each menu item.

**Styling:**
- `glk_set_style(style_Subheader)` is used for the menu title and the
  per-item heading shown during the selection display.
- `glk_set_style(style_Normal)` is used for the body and key legend.

**Input:**
- Input is read with `KeyCharPrimitive(gg_statuswin, true)`, which
  internally uses Glk's `glk_request_char_event()` and waits for an
  `evtype_CharInput` event. The navigation key constants from
  `english.h` are accepted alongside the Glk special-key codes
  (e.g. `$fffffffb` for `keycode_Down`, `$fffffffc` for `keycode_Up`,
  `$fffffffa` for `keycode_Return`, `$fffffff8` for `keycode_Escape`).

**Cleanup:**
On exit the routine clears `gg_mainwin` and (at the outermost level of
`menu_nesting`) generates a `<<Look>>` to restore the room description.
It does not explicitly call `DrawStatusLine`; the status line is
redrawn on the next turn by the main game loop.

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
pushes a new menu onto the screen, and exiting returns to the parent
(the library tracks the depth in the global `menu_nesting`):

```inform6
[ TopMenuChoiceR;
    switch (menu_item) {
        1: DoMenu("Movement^Objects^Talking^",
                   SubEntryR, SubChoiceR);
        2: print "This game was created as a demonstration.^";
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
  `pic_num` at the given position, filling it with the background color.
- `@picture_data pic_num array ?label` — stores the height and width
  of picture `pic_num` into `array-->0` and `array-->1` respectively,
  and branches to `label` if the picture exists. If `pic_num` is 0,
  it stores the number of available pictures and the release number of
  the picture file instead. (This is a *branch* instruction, not a
  store instruction: results are returned through the array and the
  branch, not through a `-> result` operand.)
- `@picture_table table` — hints to the interpreter that the pictures
  listed in `table` (a word array of picture numbers, terminated by 0)
  should be preloaded into memory for fast display.

```inform6
! Draw picture 1 at the top-left corner
@draw_picture 1 1 1;

! Query picture dimensions
Array pic_info --> 2;
@picture_data 1 pic_info ?HavePic;
jump NoPic;
.HavePic;
print "Height: ", pic_info-->0,
      " Width: ", pic_info-->1, "^";
.NoPic;

! Preload pictures 1-3
Array preload_pics --> 1 2 3 0;
@picture_table preload_pics;
```

V6 games are uncommon because few interpreters fully support the V6
graphical model. Most modern games targeting graphical output
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

The veneer includes `Glk__Wrap`, a wrapper routine around the `@glk`
opcode. It accepts the Glk call ID as its first argument and any
number of additional Glk arguments through the variadic `_vararg_count`
mechanism, dispatches them to the Glk layer via `@glk`, and returns
the Glk call's result value. This is how the compiler implements
variadic Glk function calls expressed in Inform 6 source (the
`glk_*` veneer stubs in `infglk.h` each use `@glk` directly, but
indirect or runtime-dispatched Glk calls go through `Glk__Wrap`).
On Glulx, `print` output is routed through the current Glk stream;
the `@glk` opcode (directly or through `Glk__Wrap`) is the underlying
mechanism by which the runtime talks to the Glk dispatch layer.

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

The library maintains two global variables to track undo state:

```inform6
Global undo_flag;
Global just_undone;
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
loop. This code runs at the **start** of
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
   state restore, then normalizes the result to 2.
3. **Normalize Glulx results**: the `~~i` is Inform 6's **logical NOT**
   operator (`~` is bitwise NOT; `~~` is logical NOT). It converts the
   Glulx convention (0 = success at save time) to the library's
   convention (non-zero = success) by mapping 0 to 1 and any non-zero
   value to 0.
4. **Reset `just_undone`** to 0, allowing undo on this turn.
5. **Set `undo_flag`** based on the result:
   - Result of -1 (Z-machine) → `undo_flag = 0` (no undo support).
   - Result of 0 (Z-machine) → `undo_flag = 1` (save failed this turn).
   - Otherwise → `undo_flag = 2` (undo available).

### 34.5.4 The PerformUndo Routine

When the player types UNDO, the parser recognizes it as a special
meta-command and calls `PerformUndo`.
This routine handles validation and error reporting. In the library
its checks are performed in the following order:

1. **Check for game start**: if `turns == START_MOVE`, the undo is
   rejected with library message `Miscellany` 11:
   *"You can't ~undo~ what hasn't been done!"*

2. **Check undo availability**: if `undo_flag` is 0, the undo is
   rejected with library message `Miscellany` 6:
   *"Your interpreter does not provide ~undo~. Sorry!"*
   If `undo_flag` is 1, the undo is rejected with library message
   `Miscellany` 7:
   *"You cannot ~undo~ any further."*

3. **Attempt restore**: the appropriate restore opcode is executed:

   ```inform6
   #Ifdef TARGET_ZCODE;
   @restore_undo i;
   #Ifnot; ! TARGET_GLULX
   @restoreundo i;
   i = (~~i);
   #Endif;
   ```

4. **Handle failure**: if the restore opcode returns (which means it
   failed — a successful restore never returns to this point), and
   the (Glulx-normalised) result is 0, the routine prints `Miscellany`
   message 7 and returns 0.

5. **Handle success**: on a successful restore, execution does not
   resume in `PerformUndo` at all: it resumes at the
   `@save_undo` / `@saveundo` call in the game loop (§34.5.3) with a
   result indicating restoration. The game loop detects this, sets
   `just_undone = 1`, and prints `Miscellany` message 13:
   *"Previous turn undone."*

The library's `just_undone` global is set after a successful undo and
cleared at the top of each turn. The comment in `parser.h` describes
its intent as "Can't have two successive UNDOs", but in this version
of the library `PerformUndo` does not itself test `just_undone` — the
practical block on consecutive undos comes from the underlying
opcode, since each `@restore_undo` consumes the snapshot saved by
the previous `@save_undo`.

### 34.5.5 Undo-Related Library Messages

The undo system communicates with the player through the `Miscellany`
library message set. These can be customized
through the standard library message interception mechanism
(see §32.7):

| Message Number | Default Text | When Used |
|----------------|-------------|-----------|
| 6  | "[Your interpreter does not provide ~undo~. Sorry!]" | `undo_flag == 0`: interpreter lacks undo support entirely. |
| 7 (Z-machine)  | "~Undo~ failed. [Not all interpreters provide it.]" | `undo_flag == 1`, or `@restore_undo` returned failure. |
| 7 (Glulx)  | "[You cannot ~undo~ any further.]" | `undo_flag == 1`, or `@restoreundo` returned failure. |
| 11 | "[You can't ~undo~ what hasn't been done!]" | Player tries to undo before the first move (`turns == START_MOVE`). |
| 12 | "[Can't ~undo~ twice in succession. Sorry!]" | Defined in the library but not currently emitted from the undo code path; reserved for language overrides that wish to add such a check. |
| 13 | "[Previous turn undone.]" | Successful undo restoration. |

The tilde characters (`~`) in these messages are the language's escape
sequence for double-quote characters in strings (see §1.4).

### 34.5.6 Dictionary Words for UNDO

The words that the parser recognizes as the UNDO command are defined
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
maintaining a stack of serialized game states using the file I/O
facilities described in §34.4 (Glulx only).
