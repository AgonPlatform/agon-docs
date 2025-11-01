# VDP Variables

VDP variables provide a way to both read and change the state of the VDP.  They can be used to enable an experimental feature, read or change the current state of the VDP, or store information for use in a [buffered command](./Buffered-Commands-API.md) sequence.

VDU variables are 16-bit values, although some variables may only use the lower 8 bits.  Whilst natively variables are stored as unsigned values, the VDP will interpret them as signed values where appropriate; for instance graphics coordinates can be negative, and the VDP will interpret them as such.

Variables are currently split into three general categories: test flags, system settings, and VDU variables.  In the future we will add more variables to expose the state of the audio system.  These categorites are given some broad ranges of variable IDs to allow for future expansion, and to allow for easy identification of the type of variable.

Test flags are used to enable a feature that may either be experimental, not yet fully implemented, have an API that might change, or are not fully tested.

System settings variables provide access to VDP system information, such as memory usage, real-time clock data, and keyboard settings.

VDU variables provide access to the graphics system state, including context-specific information that may change when switching [context](Context-Management-API.md).

The VDP variables system was added in Console8 VDP 2.9.0, and was initially used only for test flags.

The Console8 VDP 2.12.0 release has added many new variables that expose the state of the VDP, and allow for changes to be made to that state.  It is also now possible to use VDP variables in buffered commands.  This allows for conditional commands to be used to check against a variable, and the contents of a variable to be read into a buffer.

Further extensions to the VDP variables system were made in VDP 2.15.0, exposing more information, and functionality related to [VDP events](./Buffered-Commands-API.md#command-80).


## Variable APIs

There are just two API calls to directly work with VDP variables, one to set a variable, and another to clear it.  As of VDP 2.12.0 the buffered command API also allows for the reading of variables into a buffer, and for conditional commands to be used to perform their checks against a variable.

The commands to set and clear variables are documented in the [system commands](./System-Commands.md#vdp-var-set) documentation.  Briefly they are:

* `VDU 23, 0, &F8, variableId; value;`: Set a VDP Variable
* `VDU 23, 0, &F9, variableId;`: Clear a VDP Variable

The [buffered commands API](./Buffered-Commands-API.md) has also been updated to support the use of variables.  All conditional commands, such as the [conditional call](./Buffered-Commands-API.md#command-6) and [conditional jump to an offset](./Buffered-Commands-API.md#command-10), can support conditions to be checked against VDP variables.  A new command to [read the value of a VDP variable and store it in a buffer](./Buffered-Commands-API.md#command-48) has also been added which can be used to insert variable values into command sequences.


## Variable ID ranges

All variables listed are available from VDP 2.12.0 onwards.  Variables that were introduced in 2.9.0 and 2.11.0 are marked as such.

The broad ranges of variable IDs are as follows:

| Variable ID range | Description |
| ----------- | ----------- |
| 0x0001-0x00FF | [Test flags](#test-flags) |
| 0x0100-0x0FFF | [System variables](#system-variables) |
| 0x1000-0x1FFF | [VDU variables](#vdu-variables) (graphics/text system) |
| 0x2000-0x2FFF | Reserved for audio system variables |
| 0x3000-0x3FFF | Reserved for future use |
| 0x4000-0xFFFF | Available for general use |

The system will not prevent you from using variables in reserved ranges, although inside the "VDU variables" range values that the system does not directly support will not be stored, and cannot be read.


## Test Flags

In general, test flags are used to enable features that are not yet fully implemented, or may be considered experimental with an API that may change.  A new feature will typically require a test flag to be set to enable it until the feature is fully implemented and considered stable, at which point a program will not be required to set the flag to use the feature.  As a feature develops over time, programs may need to change the value they set the test flag to in order to opt in to new API changes.  It is possible that a new feature may continue to recognise its test flag after it has been fully implemented to allow for programs that used older versions of their API to continue to work, but this is not guaranteed.

Test flags are essentially just simple variables.  Setting a test flag for a feature that has already been fully implemented and no longer requires the flag to be set therefore will have no harmful effect.  The test flag will be ignored, and the feature will work as normal.

As of Console8 VDP 2.9.0 the following test flags are supported:

| Variable ID | Value | Description |
| ------- | ----- | ----------- |
| 1 | 0 (any) | Enable the Affine Transforms feature (available from 2.9.0) |
| 2 | 0 (any) | Enable hardware sprites |

If a flag is set that is not recognised then it will have no effect.  This means that if a feature graduates from being a test feature and no longer requires a flag to be set for use then, so long as the API for the feature remains the same, software that sets the flag to enable the feature will still work.

For the current test flags that the VDP supports, any value can be set to enable the feature, but you are strongly advised to use set their flags with a value of zero.  Future flags may require specific values to be set to control how the feature works.

(Some features that may be considered experimental use variables outside of this range to enable them.  In general this is because the feature may use a variable to control how it works, which will still be required after the feature is fully implemented.  In general though this practice will be avoided.)


## System Variables

Broadly speaking, system variables are used to read and change the state and configuration of the VDP that are not _directly_ related to the current graphics context state or screen mode.  Within the system variables block there are sub-blocks for different types of system information.

The sub-ranges within system variables are broadly as follows:

| Block | Description |
| ----------- | ----------- |
| 0x0100-0x01FF | [Communications system settings](#comms-vars) |
| 0x0200-0x0208 | [Real-time clock data](#rtc-vars) |
| 0x0209-0x020F | Reserved for future use |
| 0x0210-0x021F | [VDP memory information](#vdp-mem-vars) |
| 0x0220-0x022F | [Keyboard settings](#keyboard-vars) |
| 0x0230-0x023F | [Context management](#context-vars) |
| 0x0240-0x024F | [Mouse settings](#mouse-vars) * |
| 0x0250-0x025F | [Keyboard event info](#key-event) |
| 0x0250-0x02FF | Reserved for future use |
| 0x0300-0x03FF | [Graphical system enhancement settings/flags](#graphics-settings) |
| 0x0400-0x04FF | [Bitmap/Sprite system control settings](#bitmap-sprite-settings) |
| 0x0500-0x05FF | [Last value variables](#response-vars) |
| 0x0600-0x07FF | Reserved for future use |
| 0x0800-0x08FF | [Keyboard map](#key-map) |
| 0x0900-0x0FFF | Reserved for future use |

\* The mouse settings variables were added to this range in VDP 2.15.0, as these represent global settings for the mouse system.

Many system variables can be set and adjusted by other VDU commands

As of Console8 VDP 2.12.0 the following system variables are supported:

### Communications system settings {#comms-vars}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0101 | 0/1 | | X | Full duplex UART hardware flow control flag. This is intended for internal use by MOS. NB setting this flag will break communications with MOS unless a suitable version of MOS that supports full duplex flow control.  The first version of MOS to support this is MOS 3.0 alpha 3 |
| 0x0102 | n/a | | X | Reserved for future use (Buffer size on MOS for VDP protocol packets) |
| 0x0103 | | | X | Suppress the next VDP protocol packet. This variable is automatically cleared after the next attempt to send a VDP protocol packet (added in VDP 2.15.0) |
| 0x0110 | 0/1 | | X | Reserved for future use (Echo back received data, for redirect/spool, with a suitable version of MOS that supports this feature) |

Setting variable 0x0103 to any value will cause the next VDP protocol packet that would be sent to MOS to be suppressed and discarded.  This can be used to prevent specific events from sending data to MOS, which could be useful in some circumstances.  For example a VDP-resident "mouse-keys" implementation that intercepts keyboard events to move the mouse cursor could use this to prevent applicable keyboard events from being sent to MOS.

### Real-time clock data {#rtc-vars}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0200 | 0-999 | | | Real-time clock year |
| 0x0201 | 1-12 | | | Real-time clock month |
| 0x0202 | 1-31 | | | Real-time clock day |
| 0x0203 | 0-23 | | | Real-time clock hour |
| 0x0204 | 0-59 | | | Real-time clock minute |
| 0x0205 | 0-59 | | | Real-time clock second |
| 0x0206 | 0-999 | X | | Real-time clock millisecond |
| 0x0207 | 0-6 | X | | Real-time clock weekday |
| 0x0208 | 0-366 | X | | Real-time clock day of year |

### VDP memory information {#vdp-mem-vars}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0210 | | X | | Free PSRAM low bytes |
| 0x0211 | | X | | Free PSRAM high bytes |
| 0x0212 | | X | | Number of buffers used |

### Keyboard settings {#keyboard-vars}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0220 | 0-17 | | | Keyboard layout (setting to an invalid number will set to zero) |
| 0x0221 | 0/1 | | | Control keys on/off (setting to any non-zero value sets to 1) |
| 0x0222 | 240-1000 | | | Keyboard repeat delay (milliseconds, rounded to nearest 250) |
| 0x0223 | 33-500 | | | Keyboard repeat rate (characters per second) |
| 0x0224 | 0-7 | | | Keyboard LED status (bitmask, combined values of NumLock, CapsLock and ScrollLock) |
| 0x0225 | 0/1 | | | Keyboard NumLock LED status (1=on, 0=off) |
| 0x0226 | 0/1 | | | Keyboard CapsLock LED status (1=on, 0=off) |
| 0x0227 | 0/1 | | | Keyboard ScrollLock LED status (1=on, 0=off) |

### Context management {#context-vars}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0230 | 0-255 | | | Current active context ID |

### Mouse settings {#mouse-vars}

Support for mouse settings in this range was added in VDP 2.15.

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0240 | | | X | Mouse cursor ID * |
| 0x0241 | 0/1 | | X | Mouse enabled.  The mouse can only be enabled if a mouse is physically connected to your Agon's PS/2 mouse port |
| 0x0242 | | | | Mouse cursor X position in screen coordinates |
| 0x0243 | | | | Mouse cursor Y position in screen coordinates |
| 0x0244 | 0-7 | X | | Mouse cursor button status.  Bit 0 indicates left button pressed, bit 1 the right button, and bit 2 the middle button |
| 0x0245 | | X | | Mouse wheel delta |
| 0x0246 | | | | Mouse sample rate |
| 0x0247 | | | | Mouse resolution |
| 0x0248 | | | | Mouse scaling |
| 0x0249 | | | | Mouse acceleration |
| 0x024A | | | | Mouse wheel acceleration |
| 0x024B | 0/1 | | X | Mouse cursor visible |

\* Setting the mouse cursor ID to a valid cursor ID will (as of VDP 2.15) always show the mouse cursor, and setting to an invalid ID will hide it, but not change the stored value.  Clearing the value will reset the mouse cursor ID to the default (0) and hide the cursor.  This will also affect the "mouse cursor visible" variable accordingly.

### Keyboard event info {#key-event}

Support for keyboard event/state info in this range was added in VDP 2.15.

Values in this range provide information on the last keyboard event that occurred.  This includes all the data sent to MOS when a key is pressed, plus some more information.

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0250 | | X | | Lower byte last ASCII keycode value sent to MOS *<br>Upper byte raw ASCII value from latest key event |
| 0x0251 | | X | | FabGL Virtual keycode |
| 0x0252 | | X | | Key down flag (1) or up (0) |
| 0x0253 | | X | | Key modifiers byte (Shift, Ctrl, Alt, etc) |
| 0x0254 | | X | | CTRL key state (1=pressed, 0=not pressed) |
| 0x0255 | | X | | SHIFT key state (1=pressed, 0=not pressed) |
| 0x0256 | | X | | LEFT ALT key state (1=pressed, 0=not pressed) |
| 0x0257 | | X | | RIGHT ALT key state (1=pressed, 0=not pressed) |
| 0x0258 | | X | | CAPSLOCK key state (1=pressed, 0=not pressed) |
| 0x0259 | | X | | NUMLOCK key state (1=pressed, 0=not pressed) |
| 0x025A | | X | | SCROLLLOCK key state (1=pressed, 0=not pressed) |
| 0x025B | | X | | GUI key state (1=pressed, 0=not pressed) |
| 0x025C | | X | | Raw PS/2 key scancode value, bytes 1 and 2 |
| 0x025D | | X | | Scancode bytes 3 and 4 |
| 0x025E | | X | | Scancode bytes 5 and 6 |
| 0x025F | | X | | Scancode bytes 7 and 8 |

\* The Agon VDP makes some minor adjustments to the keycode value it sends to MOS; it sets specific values for all four arrow keys, plus the tab and backspace keys.  The raw value before these adjustments were made is available in the upper byte of this variable.  Reading the two bytes stored in this variable separately is possible using the [read VDP variable](./Buffered-Commands-API.md#command-48) command.  If you request the variable as an 8-bit value in big-endian order, it will read the upper byte containing the raw value.  Reading as an 8-bit value in little-endian order then you will return the lower byte containing the adjusted value.

### General Poll byte

Support for the general poll byte variable was added in VDP 2.15.

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0280 | 0-255 | | X | General poll byte. This will be the last byte value received with the [general poll command](./System-Commands.md#vdu-23-0-80), and/or as returned back to MOS in the corresponding VDP protocol response packet |

In most circumstances setting this variable will have no effect.  However, a [callback event handler](./Buffered-Commands-API.md#command-80) listening for event `0x100` will see this variable as the value that was received as part of the general poll command, and can modify the value before it is sent back to MOS.  This potentially means that an inventive programmer could use the "general poll" command as a way to send data from the VDP to MOS, one byte at a time...

### Graphical system enhancement settings/flags {#graphics-settings}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0300 | 0 (any) | | X | Tile engine flag (enables layers commands, available from VDP 2.11.0) |
| 0x0310 | 0 (any) | | X | Enables copper features flag |

### Bitmap/Sprite system control settings {#bitmap-sprite-settings}

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0400 | 0 (any) | | X | Prefer hardware sprites flag. When set, all sprites will be set to be hardware sprites after calling the "Reset sprites" API, if the "Enable hardware sprites" test flag has also been set |

### Last value variables {#response-vars}

Support for these variables were added in VDP 2.15.

Variables in this range are used to store the recent values related to a command or event that has occurred on the VDP.  This provides a way to see responses or results of VDU commands that may not otherwise be available as [VDU variables](#vdu-variables).  At this time, the following commands will cause these variables to be updated:

* [Get the value of a character on screen using text coordinates](./System-Commands.md#vdu-23-0-83)
* [Get the value of a character on screen using graphics coordinate](./System-Commands.md#vdu-23-0-93)
* [Get colour of pixel at a graphics coordinate](./System-Commands.md#vdu-23-0-84)
* [Read colour palette entry](./System-Commands.md#vdu-23-0-94)
* [Change a colour palette entry](./VDU-Commands.md#vdu-19)
* [Resetting the palette](./VDU-Commands.md#vdu-20)

When any of these commands are successfully executed the corresponding variables described below will be updated.  With the exception of the commands to [change a colour palette entry](./VDU-Commands.md#vdu-19) and [reset the palette](./VDU-Commands.md#vdu-20), they will also send a VDP Protocol data packet back to MOS with response data.  You can prevent the VDP Protocol data packet from being sent by temporarily [changing the output stream](./Buffered-Commands-API.md#command-4) to `65535`, and then restoring it back to the original stream by setting it back to `0` (the default stream).  This will prevent the VDP from sending the data packet, but the variables will still be set.

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x0500 | 0-255 | | X | Last character value read {#last-char} |
| 0x0510 | 0-255 | | X | Last colour red value {#last-colour} |
| 0x0511 | 0-255 | | X | Last colour green value |
| 0x0512 | 0-255 | | X | Last colour blue value |
| 0x0513 | 0-63 | | X | Last colour logical colour value (it's palette index) |
| 0x0514 | 0-63 | | X | Last colour physical colour value (RGB222 equivalent of variables 0x0510-0x0512) |
| 0x0515 | | | X | X coordinate of the last colour (graphics coordinates) |
| 0x0516 | | | X | Y coordinate of the last colour (graphics coordinates, ) |

In the case of the command to [read a colour palette entry](./System-Commands.md#vdu-23-0-94), the palette can already be read from [VDU variables](#palette-entries), but that will only provide you with the physical colour value for a palette entry.  The command can also read the currently selected text and graphics foreground and background colours, which are otherwise only available as their [logical colour values](#logical-col).

Palette changes from [`VDU 19`](./VDU-Commands.md#vdu-19) or [`VDU 20`](./VDU-Commands.md#vdu-20) will update the last colour variables and also perform any [callbacks](./Buffered-Commands-API.md#command-80) looking for a "palette change" event.  In the case of a palette reset, the last colour variable values for red, green, and blue will all be set to 0, and the logical and physical colour values will be set to 255.  As those are not valid values for logical or physical colours this can be used to detect a palette reset.

Command that read or set palette entries will set the last colour X and Y coordinate variables to 32768.

Setting a palette entry by directly changing the corresponding [VDU variable](#palette-entries) will cause the palette to be updated correctly, but will not update the last colour variables, or cause a palette change event.

### Keyboard map {#key-map}

Variables from ID `0x0800` onwards for a total of 259 variables contain a map of values that relate to the keyboard.  The index into this range is the FabGL virtual keycode value.  It should be noted that this is _not_ the ASCII value of a key.

Within this range, the variable's lower-byte is a boolean that will indicate whether that virtual key is currently pressed/down (1) or not (0).  The upper byte provides the the current ASCII equivalent to the virtual keycode, if applicable.  This value may be adjusted by a modifier key being pressed such as the Ctrl key.  Keycodes that do not have an ASCII equivalent, such as modifier keys, function keys, or keys that are not mapped in your currently selected keyboard layout, will have the upper byte set to 0.

All of the values in this range are read-only.


## VDU Variables

VDU variables are numbered in the range &1000-&1FFF and are used to expose information on the current graphics system state.  This includes both information about the current screen mode, and the current context state.

The set of variables are loosely based on the VDU variables available in Acorn's RISC OS operating system, and where appropriate they are numbered the same.  The Agon VDP includes many extensions to include additional information that is specific to the Agon platform.

In general, most of the variables within this range are specific to the currently selected [graphics context](./Context-Management-API.md).  This means that when the context is changed, the values of these variables may change.  Within this range are some variables specific to the current screen mode which will not change when the context changes, plus variables that are derived from both the current context and the screen mode, such as the number of text columns and rows; in general these variables are read-only.

Some information is specific to the current screen mode and will not change if the context is changed.  This includes the screen width and height, the number of screen banks, and the maximum logical colour number for the current screen mode, and the graphics palette definition.  Some information is derived from both the current context and the screen mode, such as the number of text columns and rows.

All variables within this range are reserved for use by the VDU system.  Any values that are not recognised will not be stored, and cannot be read.

Flag variables use values 0 to indicate disabled, and 1 to indicate enabled.  Setting a flag to any non-zero value is counted as if setting to 1.

Some variables provide coordinates.  For those variables that are marked as "screen coordinates", the origin is at the top-left of the screen, and the location is measured in pixels.  For those variables that are marked as "character coordinates", the origin is at the top-left of the screen, and the location is measured in characters.  Variables shown as "OS coordinates" will reflect the currently selected coordinate system, as defined in variable &1057.  When the default coordinate system is selected, the origin is at the bottom-left of the screen, and the location is measured in OS units, where the screen is defined as 0-1279 for X and 0-1023 for Y.

| Variable ID | Value | Read-only | Clearable | Description |
| ------- | ----- | --- | --- | ----------- |
| 0x1001 | | X | | Screen text columns - (characters) |
| 0x1002 | | X | | Screen text rows - (characters) |
| 0x1003 | 1/3/15/63 | X | | Max logical colour number for current screen mode |
| 0x100B | | X | | Screen width in pixels - 1 |
| 0x100C | | X | | Screen height in pixels - 1 |
| 0x100D | 1/2 | X | | Number of screen banks (1 for single-buffered modes, 2 for double-buffered) |
| 0x1017 | 0-255 | | | Current line thickness (pixels) |
| 0x1018 | | | | Text cursor, absolute X position (chars) |
| 0x1019 | | | | Text cursor, absolute Y position (chars) |
| 0x1020 | | | | Frame counter low word (the frame counter is a 32-bit value) |
| 0x1021 | | | | Frame counter high word |
| 0x1022 | | | | Number of frames to pause on newline when the "Ctrl" key is held * |
| 0x1023 | | | | Current number of frames being waited for * |
| 0x1055 | | X | | Current screen mode number |
| 0x1056 | 0/1 | | | Legacy modes flag |
| 0x1057 | 0/1 | | | Coordinate system (0 = screen/pixel coordinates, 1 = logical/OS (default)) |
| 0x1058 | 0/1/2*/3* | | | Paged mode flag (0 = disabled, 1 = enabled, 2 = disabled, but temporary paged mode is on *, 3 = enabled, and temporary paged mode is on *) |
| 0x1059 | | | | Paged mode context row count * |
| 0x1066 | 0-255 | | | Cursor behaviour flags byte, as set via [VDU 23,16,setting,mask](./VDU-Commands.md#cursor-behaviour) |
| 0x1067 | 0/1 | | | Text cursor visibility |
| 0x1068 | | | | Text cursor block horizontal start column |
| 0x1069 | | | | Text cursor block horizontal end column |
| 0x106A | 0-31 | | | Text cursor block vertical start row |
| 0x106B | | | | Text cursor block vertical end row |
| 0x106C | 0-3 | | | Text cursor appearance, write only (0 = steady, 1 = off, 2 = fast blink, 3 = slow blink) |
| 0x1070 | | X | | Active cursor type (0 = Text cursor, 1 = Graphics cursor) |
| 0x1080 | | | | Graphics window, LH column, screen coordinates |
| 0x1081 | | | | Graphics window, Bottom row, screen coordinates |
| 0x1082 | | | | Graphics window, RH column, screen coordinates |
| 0x1083 | | | | Graphics window, Top row, screen coordinates |
| 0x1084 | | | | Text window, LH column, character coordinates |
| 0x1085 | | | | Text window, Bottom row, character coordinates |
| 0x1086 | | | | Text window, RH column, character coordinates |
| 0x1087 | | | | Text window, Top row, character coordinates |
| 0x1088 | | | | Graphics origin, X, OS coordinates |
| 0x1089 | | | | Graphics origin, Y, OS coordinates |
| 0x108A | | | | Graphics cursor, X, OS coordinates |
| 0x108B | | | | Graphics cursor, Y, OS coordinates |
| 0x108C | | | | Oldest Graphics cursor, X, screen coordinates |
| 0x108D | | | | Oldest Graphics cursor, Y, screen coordinates |
| 0x108E | | | | Previous Graphics cursor, X, screen coordinates |
| 0x108F | | | | Previous Graphics cursor, Y, screen coordinates |
| 0x1090 | | | | Graphics cursor, X, screen coordinates |
| 0x1091 | | | | Graphics cursor, Y, screen coordinates |
| 0x1097 | 0-7 | | | GCOL action for foreground colour |
| 0x1098 | 0-7 | | | GCOL action for background colour |
| 0x1099 | 0-63 | | | Graphics foreground (logical) colour {#logical-col} |
| 0x109A | 0-63 | | | Graphics background (logical) colour |
| 0x109B | 0-63 | | | Text foreground (logical) colour |
| 0x109C | 0-63 | | | Text background (logical) colour |
| 0x10A1 | | X | | Max mode number (not double-buffered) |
| 0x10A2 | | X | | X font size, graphics cursor |
| 0x10A3 | | X | | Y font size, graphics cursor |
| 0x10A4 | | X | | X font spacing, graphics cursor |
| 0x10A5 | | X | | Y font spacing, graphics cursor |
| 0x10A7 | | X | | X font size, text cursor |
| 0x10A8 | | X | | Y font size, text cursor |
| 0x10A9 | | X | | X font spacing, text cursor |
| 0x10AA | | X | | Y font spacing, text cursor |
| 0x10F2 | | | | Dotted line pattern length |
| 0x10F3 | | | | Line pattern, bytes 0-1 |
| 0x10F4 | | | | Line pattern, bytes 2-3 |
| 0x10F5 | | | | Line pattern, bytes 4-5 |
| 0x10F6 | | | | Line pattern, bytes 6-7 |
| 0x1100 | | | | Width of text window in chars |
| 0x1101 | | | | Height of text window in chars |
| 0x1118 | | | | X position of text cursor within text window |
| 0x1119 | | | | Y position of text cursor within text window |
| 0x111A | | | | X position of text cursor in screen coordinates |
| 0x111B | | | | Y position of text cursor in screen coordinates |
| 0x1200-0x123F | 0-63 | | | Palette entries.  Maps logical colours to physical screen colours. The entries used will depend on the number of colours in the current screen mode {#palette-entries} |
| 0x1300-0x13FF | | | | Character to bitmap mapping.  Value is a 16-bit bitmap ID, or 65535 if character is not mapped.  See [`VDU 23, 0, &92, char, bitmapId;`](./System-Commands.md#vdu-23-0-92) |
| 0x1400 | | | | Currently selected bitmap ID (16-bit bitmap ID) |
| 0x1401 | | X | | Count of bitmaps used |
| 0x1402 | | | | Current bitmap transform ID. Must be set to a buffer ID containing a valid affine transform.  The affine transforms flag must be set to change this value |
| 0x1410 | 0-255 | | | Current sprite ID |
| 0x1411 | | X | | Number of sprites active (not necessarily visible) |
| 0x1440 | | | | Mouse cursor ID ** |
| 0x1441 | 0/1 | | | Mouse enabled.  The mouse can only be enabled if a mouse is physically connected to your Agon's PS/2 mouse port ** |
| 0x1442 | | | | Mouse cursor X position in screen coordinates ** |
| 0x1443 | | | | Mouse cursor Y position in screen coordinates ** |
| 0x1444 | 0-7 | X | | Mouse cursor button status.  Bit 0 indicates left button pressed, bit 1 the right button, and bit 2 the middle button ** |
| 0x1445 | | X | | Mouse wheel delta ** |
| 0x1446 | | | | Mouse sample rate ** |
| 0x1447 | | | | Mouse resolution ** |
| 0x1448 | | | | Mouse scaling ** |
| 0x1449 | | | | Mouse acceleration ** |
| 0x144A | | | | Mouse wheel acceleration ** |

\* Support for these variables was added in VDP 2.14.0<br>
\** As of VDP 2.15.0 the mouse cursor variables in this range are deprecated; you are advised to use the equivalent [mouse system variables](#mouse-vars).  On earlier versions of the VDP setting the mouse cursor ID would only work if the mouse was enabled<br>
