# VDP System Commands

`VDU 23, 0` is reserved for commands sent to the VDP.  These are largely unique to the Agon, and are used to access some of the more sophisticated features of the VDP.

Some of the commands are used for operating system functionality, and will not usually need to be used by user applications, whilst others allow access to features that applications will want to use.

Please note that not all versions of the VDP support the complete command set.  The following list uses the following symbols to indicate which VDP versions support which commands:

 \* Requires VDP 1.03 or above<br>
 \** Requires VDP 1.04 or above<br>
 § Requires Console8 VDP 2.3.0 or above<br>
 §§ Requires Console8 VDP 2.4.0 or above<br>
 §§§ Requires Console8 VDP 2.6.0 or above<br>
 §§§§ Requires Console8 VDP 2.7.0 or above<br>
 §§§§§ Requires Console8 VDP 2.8.0 or above<br>
 §§§§§§ Requires Console8 VDP 2.9.0 or above<br>
 §§§§§§§ Requires Console8 VDP 2.12.0 or above<br>
 §§§§§§§§ Requires Console8 VDP 2.14.0 or above<br>

Commands between &80 and &89 will return their data back to the eZ80 via the [VDP serial protocol](#vdp-serial-protocol).

NB:

- Prior to MOS 1.03 the subset commands that it supported were indexed from &00, not &80. For example, `VDU 23, 0, &02` to request the cursor position.  Commands in the range &00-&7F were used on Acorn systems to control the VDU hardware, so to avoid conflicts, and to allow for the possibility of supporting some of those commands, Agon commands were re-indexed from &80.


## `VDU 23, 0, &0A, n`: Set cursor start line and appearance §§§§

This command defines the start line (or row) of the cursor and its appearance.  Bits 0-4 define the start line of the cursor, and bits 5-6 define the cursor appearance.  The meaning of the appearance bits are as follows:

| Bit 6 | Bit 5 | Meaning |
| ----- | ----- | ------- |
| 0 | 0 | Steady |
| 0 | 1 | Off |
| 1 | 0 | Fast blink |
| 1 | 1 | Slow blink |

By default, the start line is zero, and the cursor appearance is a slow blink.

This command works in conjunction with `VDU 23, 0, &0B, n`, `VDU 23, 0, &8A, n`, and `VDU 23, 0, &8B, n` to define how the cursor will be drawn on screen.

The cursor start line must be less than the current font height (which is currently 8 pixels, and cannot currently be adjusted).

Support for this command was introduced in the Console8 VDP 2.7.0.  Its behaviour is compatible with the equivalent commands on the Acorn BBC Micro and RISC OS.


## `VDU 23, 0, &0B, n`: Set cursor end line §§§§

This defines the end line of the cursor.  The displayed cursor will be drawn from the start line to the end line.  The end line must be greater than the start line, and less than the current font height.

Together with the start line, this command defines the vertical size of the cursor.

If the start and end lines are equal then the cursor will be drawn as a horizontal line.  If the end line value is less than the start line value then the cursor will not be drawn.  If a value is given that is greater than the current font height then the cursor will be drawn to the end of the font height.


## `VDU 23, 0, &80, n`: General poll

This command will echo back `n` to MOS (see [VDP Serial Protocol](#vdp-serial-protocol))

This command is used by MOS and the VDP to synchronise with each other during the system start-up process.  It is not intended to be used by applications.

## `VDU 23, 0, &81, n`: Set the keyboard locale

Sets the keyboard to a given locale/format.  The following locales are supported:

| Locale | Description |
| ------ | ----------- |
| 0 | UK |
| 1 | US |
| 2 | German |
| 3 | Italian |
| 4 | Spanish |
| 5 | French |
| 6 | Belgian |
| 7 | Norwegian |
| 8 | Japanese ** |
| 9 | US International ** |
| 10 | US International Alternate ** |
| 11 | Swiss German ** |
| 12 | Swiss French ** |
| 13 | Danish ** |
| 14 | Swedish ** |
| 15 | Portuguese ** |
| 16 | Brazilian Portuguese § |
| 17 | Dvorak § |

Any other value will be interpreted as UK.

## `VDU 23, 0, &82`: Request text cursor position

This command will return the current text cursor position to MOS.  Once the cursor position has been returned the text cursor position data inside MOS's [system state variables](../mos/API.md#sysvars) will be updated to reflect the current cursor position.

## `VDU 23, 0, &83, x; y;`: Get ASCII code of character at character position x, y {#vdu-23-0-83}

This command will return the ASCII code of the character at the given character position to MOS.  There is a corresponding command `VDU 23, 0, &93, x; y;` that works in the graphics coordinate system.

This command works by comparing pixels on the screen at the given position to the currently selected font, using the currently selected text colour.  This means that it may not always be accurate, or able to succeed.  Redefining system font characters after they have been drawn, changing the font, modifying the definitions of characters within the selected font, changing the text colour, may cause this command to fail.

This command will not recognise characters that have been mapped to bitmaps using `VDU 23, 0, &92, char, bitmapId;`.

From VDP 2.15 onwards calling this command will set the ["last character read"](./VDP-Variables.md#last-char) system variable to the character that was read, or set it to zero if the character could not be read.

## `VDU 23, 0, &84, x; y;`: Get colour of pixel at graphics coordinate x, y {#vdu-23-0-84}

This command will return the colour of the pixel at the graphics coordinate position to MOS.  The corresponding [MOS sysvars](../mos/API.md#sysvars) will be updated to reflect the read pixel colour.

From VDP 2.15 onwards calling this command will set the various ["last colour"](./VDP-Variables.md#last-colour) VDP variables to the coordinates used for this command and to the colour that was read.  It will set the colour values to zero if the pixel could not be read.  A "read pixel" event will also be triggered allowing [callbacks](./Buffered-Commands-API.md#command-80) to potentially modify the data being sent to MOS.

## `VDU 23, 0, &85, channel, command, <args>`: Audio commands

Sends a command to the [VDP Enhanced Audio API](Enhanced-Audio-API.md) **

Prior to VDP 1.04 this command could only perform what is now audio command zero, which plays a note on a channel.

## `VDU 23, 0, &86`: Fetch the screen dimensions

Returns the screen dimensions to MOS.  Generally applications should not need to call this, as this information will be automatically sent to MOS when the screen mode is changed.

## `VDU 23, 0, &87`: RTC control *

This command controls the Real Time Clock within the Agon VDP.

- `VDU 23, 0, &87, 0`: Read the RTC
    - a data packet will be sent to MOS with the current RTC data, and [MOS sysvars](../mos/API.md#sysvars) updated accordingly

- `VDU 23, 0, &87, 1, y, m, d, h, m, s`: Set the RTC


## `VDU 23, 0, &88, delay; rate; led`: Keyboard Control *

This command controls the keyboard repeat delay, repeat rate, and LED status.

Only delay values between 250 and 1000 are supported, and represents a time in milliseconds before the key will start repeating.

The rate value represents the time between key repeats, and is given in milliseconds.  Values from 33-500 are supported.

The LED value is a bit mask that controls the state of the keyboard LEDs.  The following bits are defined:

| Bit | Name | Meaning |
| --- | ---- | ------- |
| 0 | Scroll Lock | If set then the Scroll Lock LED will be turned on.  If clear then it will be turned off. |
| 1 | Caps Lock | If set then the Caps Lock LED will be turned on.  If clear then it will be turned off. |
| 2 | Num Lock | If set then the Num Lock LED will be turned on.  If clear then it will be turned off. |

## `VDU 23, 0, &89, command, [<args>]`: Mouse control **

Commands beginning with `VDU 23, 0, &89` are reserved for mouse control, and are implemented from VDP 1.04 onwards.

### `VDU 23, 0, &89, 0`: Enable the mouse {#enable-mouse}

Enables the mouse.

If this command is successful then the currently selected mouse cursor will be shown on the screen, and mouse movements will result in mouse update information being sent to MOS.  On first enabling the mouse the default mouse cursor will be shown.

If there is no mouse connected then this command will have no effect.

### `VDU 23, 0, &89, 1`: Disable the mouse

Disables the mouse cursor, and will stop sending mouse data packets to MOS.

### `VDU 23, 0, &89, 2`: Reset the mouse {#reset-mouse}

Resets the mouse system restoring default settings.

NB this command does not reset the selected mouse cursor.  If a reset is successful the mouse will be enabled and the mouse cursor will be shown.

### `VDU 23, 0, &89, 3, cursorId;`: Select a mouse cursor {#select-mouse-cursor}

Changes the mouse cursor to the given cursor ID.

NB This command will only work if the mouse has been successfully [enabled](#enable-mouse).

If you attempt to select an invalid cursor ID (i.e. the cursor not exist) then the mouse cursor will be hidden, but the system will keep track of the previously selected cursor.  The previous cursor be re-shown if you call the [enable](#enable-mouse) or [reset](#reset-mouse) command.  As the system supports [user-defined cursors](./Bitmaps-API.md#mouse-cursor), to explicitly hide the mouse cursor you should use a cursor ID of `65535`.

There are several system mouse cursors available for use.  These have been inherited from FabGL and are numbered from 0-18.  The "Cursor" column in the table below shows the FabGL name for the cursor.

| ID | Cursor | Description |
| -- | ------ | ----------- |
| 0  | PointerAmigaLike | 11x11 Amiga like colored mouse pointer (default) |
| 1  | PointerSimpleReduced | 10x15 mouse pointer |
| 2  | PointerSimple | 11x19 mouse pointer |
| 3  | PointerShadowed | 11x19 shadowed mouse pointer |
| 4  | Pointer | 12x17 mouse pointer |
| 5  | Pen | 16x16 pen |
| 6  | Cross1 | 9x9 cross |
| 7  | Cross2 | 11x11 cross |
| 8  | Point | 5x5 point |
| 9  | LeftArrow | 11x11 left arrow |
| 10 | RightArrow | 11x11 right arrow |
| 11 | DownArrow | 11x11 down arrow |
| 12 | UpArrow | 11x11 up arrow |
| 13 | Move | 19x19 move |
| 14 | Resize1 | 12x12 resize orientation 1 |
| 15 | Resize2 | 12x12 resize orientation 2 |
| 16 | Resize3 | 11x17 resize orientation 3 |
| 17 | Resize4 | 17x11 resize orientation 4 |
| 18 | TextInput | 7x15 text input |

Additional cursors can be added using [`VDU 23, 27, &40, hotX, hotY`](./Bitmaps-API.md#mouse-cursor) which will allow you to create a custom mouse cursor using a bitmap.  Once your new cursor has been defined you will be able to select it using this command.  Please note the inbuilt system mouse cursors cannot be overridden.

Whilst this command will only work if the mouse has been successfully [enabled](#enable-mouse), it is possible to display a mouse cursor without a mouse being connected/enabled by setting the equivalent [VDP Variable](./VDP-Variables.md#mouse-vars) to the desired cursor ID.

### `VDU 23, 0, &89, 4, x; y;`: Set mouse cursor position

Explicitly moves the mouse cursor to a given position on the screen.

This command works whether the mouse is enabled or not, and will even work if the mouse is not visible.

### `VDU 23, 0, &89, 5, x1; y1; x2; y2;`: Reserved

This command is reserved for future use.  It is not yet implemented.

(When implemented it will set the mouse area, which will restrict the mouse cursor to a given area of the screen.  The mouse cursor will not be able to leave this area.)

### `VDU 23, 0, &89, 6, sampleRate`: Set mouse sample rate

Sets the rate at which mouse data will be sampled.  The rate given is a number of samples per second.  The default rate is 60.

Valid sample rates are 10, 20, 40, 60, 80, 100, and 200 (samples/sec).  Any other values will be ignored.

### `VDU 23, 0, &89, 7, resolution`: Set mouse resolution

Sets the mouse resolution.  Values in the range 0-3 are supported, or 255 (-1) to pick the default resolution.  The default resolution is 2.

The following resolutions are supported:

| Value | Resolution |
| ----- | ---------- |
| 0 | 1 count per mm (25 dpi) |
| 1 | 2 counts per mm (50 dpi) |
| 2 | 4 counts per mm (100 dpi) (default) |
| 3 | 8 counts per mm (200 dpi) |

### `VDU 23, 0, &89, 8, scaling`: Set mouse scaling

Sets the mouse scaling factor.  Only values `1` and `2` are supported to indicate 1:1 or 1:2 scaling.  Additionally a value of `0` can be sent to indicate "default" scaling, which is 1:1.

### `VDU 23, 0, &89, 9, acceleration;`: Set mouse acceleration

This sets the mouse acceleration factor to a 16-bit value.  Setting the value to `0` will result in the default acceleration being used, which is 180.  The suggested range for this is 0-2000.

### `VDU 23, 0, &89, 10, wheelAcceleration; wheelAccHighByte`: Set mouse wheel acceleration (accepts a 24-bit value)

Sets the wheel acceleration factor to a 24-bit value.  Setting the value to `0` will result in the default acceleration being used, which is 60000.  The suggested range for this is 0-100000.

### Mouse data packets

Mouse data packets are sent in response to all of the above commands, and if the mouse has been enabled whenever the mouse is moved.  This ensures that mouse data is constantly updated in [MOS sysvars](../mos/API.md#sysvars).


## `VDU 23, 0, &8A, n`: Set the cursor start column §§§§

This command defines the start pixel column of the text cursor.  The displayed cursor will be drawn from the start column to the end column.  The start column can be set to any value, but for the cursor to be visible it must be less than the current font width.  The default value is 0.

Acorn systems did not support directly adjusting the number of columns used by the cursor, and so this command is not supported on those systems and is an Agon-specific extension.

## `VDU 23, 0, &8B, n`: Set the cursor end column §§§§

This command defines the end pixel column of the cursor.  The displayed cursor will be drawn from the start column to the end column.  The end column can be set to any value, but for the cursor to be visible it must be greater than the start column, and less than the current font width.  The default value is 255, which together with the default start column value ensures the cursor is the full width of the currently selected font.

Together with the start column, this command defines the width of the cursor.

If the start and end column values are equal then the cursor will be drawn as a vertical line.  If a value is given that is greater than the current font width then the cursor will be drawn to the end of the font width.

Acorn systems did not support directly adjusting the number of columns used by the cursor, and so this command is not supported on those systems and is an Agon-specific extension.

## `VDU 23, 0, &8C, x; y;`: Relative cursor movement (by pixels) §§§§§

This command will move the text cursor by the given number of pixels in the X and Y directions.  The primary purpose of this command is to allow for slight adjustments of the text cursor position to facilitate text drawing features such as superscript, subscript and manual kerning.

Normal cursor scrolling and wrapping behaviour will be obeyed, depending on the currently set cursor behaviour.

## `VDU 23, 0, &90, n, b1, b2, b3, b4, b5, b6, b7, b8`: Redefine character n (0-255) with 8 bytes of data § {#vdu-23-0-90}

This command works identically to `VDU 23, n, b1, b2, b3, b4, b5, b6, b7, b8`, but allows characters 0-31 to also be redefined.

NB from Console8 VDP 2.8.0 this command will only function if the currently selected font is the system font.

## `VDU 23, 0, &91`: Reset all system font characters to original definition § {#vdu-23-0-91}

This command will reset all system font characters to their original definitions.

## `VDU 23, 0, &92, char, bitmapId;`: Map character char to display bitmapId §§ {#vdu-23-0-92}

This command will map a character to a bitmap.  The bitmap ID given must be a valid bitmap in a 16 bit buffer ID.  The purpose of this command is to allow for fast and efficient drawing of bitmaps to the screen, by allowing them to be drawn as characters.

Any character number can be used, including those in the range of 0-31, or the usual/normal alphabetic range.  This could be used, for instance, to create a custom colour font.  It could alternatively be used with characters in the range of 128-255 to define a custom graphical tile-set, like PETSCII but in colour.

When a character has been mapped to use a bitmap the bitmap will be used in place of that character, and will be drawn at the current text cursor position, placing the bottom left of the bitmap at the cursor position.  The cursor position will then be moved to the right by the width of a character, and _not_ by the width of the bitmap.

That last point is important.  It means that if you use a bitmap that is larger than a character, then the next character will overlap the bitmap.  Similarly bitmaps that are taller than a character will overwrite the line of text above the current cursor.  This can be used to create some interesting effects, but can also be a source of visual bugs if you are not careful.  If you are using oversized bitmaps, then using the `VDU 9` to move forward an additional character may be useful.

Bitmaps mapped to characters in this way are plotted using the current foreground GCOL paint mode.

## `VDU 23, 0, &93, x; y;`: Get ASCII code of character at graphics position x, y §§§§§ {#vdu-23-0-93}

This command will return the ASCII code of the character at the given graphics position to MOS.  This command is similar to `VDU 23, 0, &83, x; y;`, but uses coordinates from the currently selected graphics coordinate system.  The position is for the top left of the character.

This command works by comparing pixels on the screen at the given position to the currently selected font, using the currently selected text colour.  This means that it may not always be accurate, or able to succeed.  Redefining system font characters after they have been drawn, changing the font, modifying the definitions of characters within the selected font, changing the text colour, may cause this command to fail.

This command will not recognise characters that have been mapped to bitmaps using `VDU 23, 0, &92, char, bitmapId;`.

## `VDU 23, 0, &94, n`: Read colour palette entry n (returns a pixel colour data packet) §§ {#vdu-23-0-94}

This command will return the colour of the given palette entry to MOS.  This data is sent using a "screen pixel" data packet.  The corresponding [MOS sysvars](../mos/API.md#sysvars) related to screen pixel colour will be updated to reflect the read palette entry.

The Agon VDP system supports a 64 colour palette, so values in the range of 0-63 will return data on that palette entry.  The following special values are also supported:

| Value | Description |
| ----- | ----------- |
| 128 | The current text foreground colour |
| 129 | The current text background colour |
| 130 | The current graphics foreground colour |
| 131 | The current graphics background colour |

Any other colour value will not be recognise, and no response sent.

It should be noted that before Console8 VDP 2.7.0 when reading palette entries for the current text and graphics colours, the data packet returned would reflect back the colour number `n` sent to this command, rather than responding with the actual palette colour number for that colour.  As of Console8 VDP 2.7.0 the actual palette colour number is returned.

From VDP 2.15 onwards calling this command will set the ["last colour"](./VDP-Variables.md#last-colour) VDP variables to the colour that was read, or set it to zero if the pixel could not be read.  The last colour X and Y coordinate variables will be set to 32768.

## `VDU 23, 0, &95, <command>, [<args>]`: Font management commands §§§§§

This command is used to manage fonts on your system.  For more information please see the [Font API documentation](Font-API.md).

## `VDU 23, 0, &96, <flags>, <bufferId>;`: Set an affine transform matrix §§§§§§

As of the time of writing, this command is experimental and subject to change.  To enable it you must turn on the affine transform feature flag, using the `VDU 23, 0, &F8, 1; 1;` command.

This command tells the graphics system to use an affine transform matrix held within the given buffer for subsequent drawing commands.  This allows for the transformation of graphics when they are drawn in a variety of ways, including scaling, rotation, and translation.

The `flags` field is used to indicate which parts of the graphics system should use the matrix.  The following flags bits are supported:

| Bit | Name | Description |
| --- | ---- | ----------- |
| 0   | Bitmap | Apply the matrix to bitmap drawing commands |
| 1-7 | Reserved | Reserved for future use |

The `bufferId` indicates which buffer holds the matrix data to use.  The matrix is assumed to be a 3x3 matrix of 32-bit floating point values, stored in little-endian byte order.  As creating and manipulating floating point data is not supported in the eZ80 various options are provided as part of the [buffered commands API](Buffered-Commands-API.md) to allow for the creation and manipulation of these matrices.

The `bufferId` set is used as a reference to the matrix.  If a matrix exists in the buffer then a copy of that will be used for all applicable drawing operations when they are performed, otherwise drawing operations will occur as if no matrix was set.  If a matrix is changed after it has been set, then the new matrix will be used for subsequent drawing operations.

Setting the `bufferId` to 65535 (or -1) will clear the matrix, and subsequent drawing operations will not be transformed.

Please note that when an affine transform is set to be applied when drawing bitmaps, the transform is always applied as if the top left of the bitmap is the "origin" point for the transform  So if an affine transform is set to only perform a rotation, then the bitmap will rotate around its top left corner.  This may be slightly contrary to expectations when using OS coordinates, as when plotting a bitmap using the PLOT command you specify the location for the bottom left of the bitmap.  So when using OS coordinates, the origin point of the transformation becomes where the top-left pixel would have been.

## `VDU 23, 0, &98, n`: Turn control keys on and off §§§

Turns control keys on and off, where 1=on (the default) and 0=off.

When control keys are turned on, pressing control and various letters on the keyboard will cause the keyboard to trigger VDU commands on the VDP, where the VDU command corresponds to the number of that letter in the alphabet.  For example, pressing `CTRL`+`N` will perform the VDU command `VDU 14` on the VDP and turn on "paged mode", as N is the fourteenth letter.

Up to and including Console8 VDP 2.4.0, only `CTRL`+`N` and `CTRL`+`O` were supported, and would send `VDU 14` and `VDU 15` respectively, enabling and disabling "paged mode".  Console8 VDP 2.5.0 added support for several more letters, specifically `B`, `C`, `F`, `G`, `L`, `N` and `O`.  `CTRL`+`P` is also supported but rather than performing a VDU 16 it will toggle the printer on and off, compatible with behaviour on BBC BASIC for Windows.  (Owing to how the VDP and the eZ80 interact, it is unlikely that any more control keys will be added in the future.)

When control keys are turned off, the keyboard will not trigger VDU commands on the VDP.

Irrespective of whether the VDP has done anything with a control key, or whether they are enabled or not, key information will always be sent through to MOS.

Until Console8 VDP 2.6.0, this control key behaviour on the VDP was always enabled, and could not be turned off.

From Console8 VDP 2.6.0 onwards, the control keys can be turned off, which may help ensure that unexpected behaviour on the VDP does not occur when using applications running on MOS that may also wish to use these control key combinations.

## `VDU 23, 0, &99, virtualKey`: Request updated keyboard data for a key §§§§§§§

This command will send an updated keycode data packet to MOS with the current state of the given key, using a vdp-gl/FabGL virtual key code.  Usually you should not have to use this command, as the keyboard data packets are sent automatically whenever a key is pressed or released.

(This command is used by MOS 3.0 at boot time to determine if the left shift key is pressed to signals that the boot file should not be run from the SD card.)

## `VDU 23,0, &9A`: Temporarily enable paged mode §§§§§§§§ {#vdu-23-0-9a}

This command temporarily enables the VDP's "paged mode", as if a `VDU 14` command had been sent.  Temporary paged mode remains active until one frame after the VDP no longer has any VDU commands to process, at which point the paged mode is restored to its previous setting.

This command is useful for applications that need to temporarily enable paged mode, but do not want to change the paged mode setting for the VDP permanently.  For example, MOS 3.0 uses this command to ensure that star commands that can output a lot of text to screen will be automatically paged.

## `VDU 23, 0, &9B, bufferId;`: Print the contents of a buffer to the screen §§§§§§

If a buffer is found with the given `bufferId`, then this command will print a buffer out to the screen, using only the raw character values. This bypasses VDU command processing, so no control characters at all are supported and will be printed as-is.

## `VDU 23, 0, &9C`: Set the text viewport using graphics coordinates §§§§§

Sets the text viewport using the rectangle described by the last two coordinates given via PLOT commands.  This command serves a similar purpose to `VDU 28`, but uses graphics coordinates instead of text coordinates, and does not require the coordinates to be sent as part of the command.  This allows for more flexibility in defining the viewport.

Whilst the graphics cursor positions can be anywhere in the coordinate space, the text viewport will be limited to appear within the screen, and will be clipped to the screen.  If the resultant viewport has zero width or height (as the coordinates were off-screen) then the command will have no effect.  This behaviour differs from `VDU 28` which would reject the command if any of the coordinates given were off-screen.

## `VDU 23, 0, &9D`: Set the graphics viewport using graphics coordinates §§§§§

Sets the graphics viewport using the rectangle described by the last two coordinates given via PLOT commands.  This command serves a similar purpose to `VDU 24`, but uses coordinates from the graphics coordinate stack, which allows for more flexibility in defining the viewport.  This can, for instance, be used to define a new viewport in coordinates that are relative to the current graphics origin.

As per `VDU 23, 0, &9C`, the viewport will be clipped to the screen if it is off-screen.  This behaviour differs from `VDU 24` which would reject the command if any of the coordinates given were off-screen.

## `VDU 23, 0, &9E`: Set the graphics origin using graphics coordinates §§§§§

Sets the graphics origin to the last coordinate given via PLOT commands.  This command serves a similar purpose to `VDU 29`, but uses coordinates from the graphics coordinate stack, which allows for more flexibility in defining the origin.  This can, for instance, be used to define a new origin in coordinates that are relative to the current origin.

## `VDU 23, 0, &9F`: Move the graphics origin and viewports §§§§§

Moves the graphics origin to the current graphics cursor position, and also moves the currently defined text and graphics viewports by the same relative amount that the origin has moved.

As with `VDU 23, 0, &9C` and `VDU 23, 0, &9D` the resultant text and graphics viewports will be clipped to the screen area.  This means that this command is not reversible, i.e. if you move the origin to position `100,100` and then move it to `-100,-100` the viewports will not return to their original sizes, as they will have been shrunk in the process.

## `VDU 23, 0, &A0, <bufferId>, <command>, [<args>]`: Buffered command API **

Send a command to the [VDP Buffered Commands API](Buffered-Commands-API.md)

## `VDU 23, 0, &A1`: Update VDP (for exclusive use of the agon-flash tool) **

This command is used by the agon-flash tool to update the VDP firmware.  It should not be used by any other software.

## `VDU 23, 0, &C0, n`: Turn logical screen scaling on and off *

Turns logical screen scaling on and off, where 1=on and 0=off.

When logical scaling is turned off, the graphics system will no longer use the 1280x1024 logical coordinate system and instead use pixel coordinates.  The screen origin point at 0,0 will change to be the top left of the screen, and the Y axis will go down the screen instead of up.

For more information, see the [Screen modes documentation](Screen-Modes.md).

## `VDU 23, 0, &C1, n`: Switch legacy modes on or off **

Turns legacy screen modes on and off, where 1=on and 0=off.

By default, the original screen modes 0-4 are not available and are instead replaced by new modes that are more compatible with modern monitors.  For compatibility with older software, written for Agon systems running earlier versions of the VDP firmware, this command can be used to switch back to those original, legacy, screen modes.

For more information, see the [Screen modes documentation](Screen-Modes.md).

## `VDU 23, 0, &C3`: Swap the screen buffer and/or wait for VSYNC **

Swap the screen buffer (double-buffered modes only) or wait for VSYNC (all modes).

This command will swap the screen buffer, if the current screen mode is double-buffered, doing so at the next VSYNC.  If the current screen mode is not double-buffered then this command will wait for the next VSYNC signal before returning.  This can be used to synchronise the screen with the vertical refresh rate of the monitor.

Waiting for VSYNC can be useful for ensuring smooth graphical animation, as it will prevent tearing of the screen.

(In BASIC performing a `*FX 19` command will perform a similar wait for VSYNC, but on the eZ80 side of the system, but will not swap the screen buffer.)

## `VDU 23, 0, &C4, command, [<args>]`: "Copper" functions §§§§§§§

Send a command to the [VDP Copper API](Copper-API.md).  This allows for the creation of different palettes and for the palettes to be changed on the fly during the scanout of the screen.

These functions were introduced in VDP 2.12.0.  At this time, to access these APIs a feature flag must be set using `VDU 23, 0, &F8, &310; 0;`.  The exact feature set provided may be subject to change in future versions of the VDP firmware.

## `VDU 23, 0, &C8, <command>, [<args>]`: Context management API §§§§§

Send a command to the [Context Management API](Context-Management-API.md).  This allows management of the current graphics context, which includes the current font, text and graphics colours, and the current GCOL paint mode.

The context management API allows for the entire current graphics state to be saved and restored, which can be useful for applications that need to temporarily change the graphics state, but then restore it to its original state.  As part of this, the current state can be saved to a stack, and then restored later.  Additionally completely separate context stacks can be selected.

## `VDU 23, 0, &CA`: Flush current drawing commands §§§§§

For performance reasons, all drawing commands (for both graphics and text) received by the VDP are actually placed in a queue, and are not immediately drawn to the screen.  This command will force all pending drawing commands to be processed and drawn to the screen.

This command is useful for some advanced operations, such as on-the-fly redefining character data in a custom font, ensuring that all characters are drawn before the font is redefined.

Most applications will not need to use this command, as the VDP will automatically flush the drawing queue when it receives a command that requires the screen to be updated, such as reading a screen pixel.

## `VDU 23, 0, &F2, n`: Set dot-dash pattern length §§§§

This command sets the length of the dot-dash pattern used for drawing lines with dotted line plot codes.  The default length is 8, and the length can be set to any value between 1 and 64.  Setting a length of 0 will reset to the default length, and reset the pattern to the default pattern.

The line pattern can be set using `VDU 23, 6, n1, n2, n3, n4, n5, n6, n7, n8`, where `n1` is the first byte of the pattern, `n2` is the second byte, and so on.  Bits are used most-significant-bit first.

Support for this command was added in Console8 VDP 2.7.0.

## `VDU 23, 0, &F8, variableId; value;`: Set a VDP Variable §§§§§§ {#vdp-var-set}

This command is used to set a [VDP variable](VDP-Variables.md).  VDP variables are used to control various features of the VDP, including enabling new functionality that may not quite be ready for general use and/or have an API that may change in the future.  They may also allow for changing various aspects of the VDP's behaviour.

The `variableId` is the ID of the variable to set, and the `value` is the value to set it to.  The meaning of the `value` that any particular variable is set to will be specific to the flag being set.  A value must always be provided, even if the variable does not require a value to be set.

For details on the variables that can be set, see the [VDP Variables documentation](VDP-Variables.md).

## `VDU 23, 0, &F9, variableId;`: Clear a VDP Variable §§§§§§

This command is used to clear a VDP variable, removing them from the variable store, when possible.

Most VDP variables that reflect the system state information cannot be cleared, and this command will ignored attempts to clear them.

## `VDU 23, 0, &FE, n`: Console mode **

Turns "console mode" on and off, where 1=on and 0=off (the default).

This mode is primarily intended for use with debugging tooling.  It will reflect VDU command bytes through to a serial terminal attached to the VDPs USB port, and will attempt to pass keyboard input from the serial terminal back through the VDP to MOS.

This mode is not intended for general use.  If you are looking for a way to output data to the attached serial terminal then `VDU 2` is a better option.


## `VDU 23, 0, &FF`: Switch to or resume "terminal mode"

This command enables "terminal mode", which changes the behaviour of the VDP to be more like a traditional terminal.  This is useful for running CP/M in place of MOS on the eZ80, or other software that expects to be running on a terminal.

When terminal mode is enabled, VDU commands will no longer be recognised and processed, keyboard entry in BBC BASIC/MOS is no longer supported, and the VDP protocol is essentially suspended.  Instead the VDP acts as a dumb terminal, and will send all keyboard input to the eZ80's UART0.  The eZ80 can then process this input as it sees fit.

By default a VT100 terminal is emulated, but this can be changed by sending the appropriate escape sequences to the VDP.  Terminal support is inherited from FabGL, and so its [custom escape sequences](http://www.fabglib.org/special_term_escapes.html) can be used to change the terminal behaviour.

From Console8 VDP version 2.2.0, the following additional escape sequences are supported:

| Sequence | Description |
| -------- | ----------- |
| `ESC + "_#Q!$"` | Quit/end terminal mode |
| `ESC + "_#S!$"` | Suspend terminal mode |

Quitting terminal mode returns the VDP back to normal operation, and will resume the VDP protocol.  The screen mode will be reset to the mode that was active before terminal mode was enabled.

Suspending terminal mode temporarily restores VDU command processing.  (Keyboard handling is unchanged.)  This could be useful to use VDU commands to perform drawing operations.  To resume terminal mode, `VDU 23, 0, &FF` must be sent again.



## VDP Serial Protocol

Data sent from the VDP to the eZ80's UART0 is sent as a packet in the following format:

- `cmd`: The packet type, with bit 7 set
- `len`: Number of data bytes
- `data`: The data byte(s)

Words are 16 bit, and sent in little-endian format

In general, as a programmer using an Agon you should not need to worry about the format and contents of any of these packets, as they are handled by MOS.  On receipt of one of these packets, MOS will set [system state variables (sysvars)](../mos/API.md#sysvars) accordingly.  It may also set a bit in the VDPProtocol status byte sysvar corresponding to the type of data packet received.

If you are performing a command where you need to wait for a response from the VDP, then you will need to wait for the appropriate bit to be set in the VDP protocol byte.  Before you send a command to the VDP you should clear the bit that relates to the command you are about to send.  Once the command has been processed the VDP will set the bit again.  If the bit is not set then there may have been an error processing the command.

Packets:

- `0x00, value`: General Poll
- `0x01, keycode, modifiers, vkey, keydown`: Keyboard state
- `0x02, x, y`: Cursor position
- `0x03, char`: Character read from screen
- `0x04, r, g, b, index`: Pixel colour read from screen
- `0x05, channel, status`: Audio command status (see [VDP Enhanced Audio API](Enhanced-Audio-API.md))
- `0x06, width; height; cols, rows, colours`: Screen dimensions - width and height are words
- `0x07, year, month, day, dayOfYear, dayOfWeek, hour, minute, second`: RTC data *
- `0x08, delay, rate, led`: Keyboard status - delay and rate are words
- `0x09, x; y; buttons, wheelDelta, deltaX; deltaY;`: Mouse status - x, y, deltaX and deltaY are words

\* as of VDP 1.04 the RTC data is sent in a packed format, and is stored in the [MOS sysvars](../mos/API.md#sysvars) in this packed format.  Prior to VDP 1.04 the `dayOfYear` value could be incorrect, as it cannot be guaranteed to fit into a byte.

MOS VDP Protocol flag bits:

| Bit | Description |
| --- | ----------- |
| 0 | Cursor position |
| 1 | Character read from screen |
| 2 | Point data (pixel colour) read from screen |
| 3 | Audio status |
| 4 | Mode info / Screen dimensions |
| 5 | RTC data |
| 6 | Mouse status |

### VDP Protocol events

As of VDP 2.15.0, [events](./Buffered-Commands-API.md#command-80) will be fired in the VDP before and after each packet is sent, and the values sent in VDP protocol packets will reflect the [VDP variables](VDP-Variables.md) that hold the same information.  This allows programs to potentially intercept and modify this data before it is sent to MOS using buffered command sequences, or even to prevent VDP protocol packets from being sent.


### Keyboard packet data

When a key is pressed, a packet is sent with the following data:

- cmd: 0x01
- keycode: The ASCII value of the key pressed
- modifiers: A byte with the following bits set (1 = pressed):

```
0. CTRL
1. SHIFT
2. ALT LEFT
3. ALT RIGHT
4. CAPS LOCK
5. NUM LOCK
6. SCROLL LOCK
7. GUI
```

From VDP 1.03, the following key data is also returned

- vkey: The FabGL/vdp-gl virtual keycode
- keydown: 1 if the key is down, 0 if the key is up
