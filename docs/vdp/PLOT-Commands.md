# VDU 25: PLOT commands

`VDU 25, code, x; y;`

The Agon VDP system supports a number of `PLOT` commands via `VDU 25`, which are used to draw lines, circles, and other shapes on the screen.  The command set is inherited from Acorn's BBC Micro, plus Acorn's Graphics eXtension ROM (GXR).

PLOT commands sent to the VDP all require a command byte, followed by two 16-bit values for X and Y coordinates.  The command byte is used to specify the drawing operation, and the mode in which the drawing operation is performed.

Most PLOT commands require additional coordinates than the single pair provided with the command.  The graphics system works using a concept of a "graphics cursor", and will keep track of the last few positions of the graphics cursor.  Every PLOT command "pushes" a copy of the current graphics cursor position onto a stack, and then moves the graphics cursor to the new position.  Commands that require additional coordinates will look at the last few positions on the stack to determine the additional coordinates.

Support for PLOT commands has grown over time, and as of VDP 2.16 the Agon now supports all operations available on a BBC Micro with a GXR ROM, or on RISC OS.  The complete basic set of GCOL painting modes are also supported, but at this time pattern fills are not available.

PLOT commands are essentially split into a drawing operation, and a mode that controls how the drawing operation is performed.  There are 8 different modes for each operation.  The command byte is effectively split into two parts where the lower 3 bits define the mode, and the upper 5 bits define the drawing operation.

The complete set of PLOT codes supported by the Agon VDP follows.

| PLOT code | (Decimal) | Effect |
| --------- | --------- | ------ |
| &00-&07 | 0-7 | Solid line, includes both ends |
| &08-&0F | 8-15 | Solid line, final point omitted |
| &10-&17 | 16-23 | Dot-dash line, includes both ends, pattern restarted §§§§ |
| &18-&1F | 24-31 | Dot-dash line, final point omitted, pattern restarted §§§§ |
| &20-&27 | 32-39 | Solid line, first point omitted |
| &28-&2F | 40-47 | Solid line, both points omitted |
| &30-&37 | 48-55 | Dot-dash line, first point omitted, pattern continued §§§§ |
| &38-&3F | 56-63 | Dot-dash line, both points omitted, pattern continued §§§§ |
| &40-&47 | 64-71 | Point plot |
| &48-&4F | 72-79 | Line fill left and right to non-background §§ |
| &50-&57 | 80-87 | Triangle fill |
| &58-&5F | 88-95 | Line fill right to background §§ |
| &60-&67 | 96-103 | Rectangle fill |
| &68-&6F | 104-111 | Line fill left and right to foreground §§ |
| &70-&77 | 112-119 | Parallelogram fill |
| &78-&7F | 120-127 | Line fill right to non-foreground §§ |
| &80-&87 | 128-135 | Flood until non-background §§§§§ |
| &88-&8F | 136-143 | Flood until foreground §§§§§ |
| &90-&97 | 144-151 | Circle outline |
| &98-&9F | 152-159 | Circle fill |
| &A0-&A7 | 160-167 | Circular arc §§§§ |
| &A8-&AF | 168-175 | Circular segment §§§§ |
| &B0-&B7 | 176-183 | Circular sector §§§§ |
| &B8-&BF | 184-191 | Rectangle copy/move |
| &C0-&C7 | 192-199 | Ellipse outline §§§§§ |
| &C8-&CF | 200-207 | Ellipse fill §§§§§ |
| &D0-&D7 | 208-215 | Not defined |
| &D8-&DF | 216-223 | Fill path (Experimental - Not defined on Acorn systems) §§§§ |
| &E0-&E7 | 224-231 | Not defined |
| &E8-&EF | 232-239 | Bitmap plot § |
| &F0-&F7 | 240-247 | Not defined |
| &F8-&FF | 248-255 | Not defined |

Within each group of eight plot codes, the effects are as follows:

| Plot code | Effect |
| --------- | ------ |
| 0  (8) | Move relative |
| 1  (9) | Plot relative in current foreground colour |
| 2  (A) | Plot relative in logical inverse colour §§§ |
| 3  (B) | Plot relative in current background colour |
| 4  (C) | Move absolute |
| 5  (D) | Plot absolute in current foreground colour |
| 6  (E) | Plot absolute in logical inverse colour §§§ |
| 7  (F) | Plot absolute in current background colour |

Codes 0-3 use the position data provided as part of the command as a relative position, adding the position given to the current graphical cursor position.  Codes 4-7 use the position data provided as part of the command as an absolute position, setting the current graphical cursor position to the position given.

The various "Line fill" plot commands have an additional effect, which is to adjust the graphics cursor position to the right-hand edge of the filled line, if one was drawn.

§ Support added in Agon Console8 VDP 2.1.0<br>
§§ Support added in Agon Console8 VDP 2.2.0<br>
§§§ Support added in Agon Console8 VDP 2.6.0<br>
§§§§ Support added in Agon Console8 VDP 2.7.0<br>
§§§§§ Support added in Agon Platform VDP 2.16.0<br>


## Interaction with GCOL paint modes

The GCOL command (`VDU 18, mode, colour`) is used to set the paint mode for the PLOT command.  The paint mode is used to control how the PLOT command interacts with the existing pixels on the screen.

Up until Console8 VDP 2.6.0 the only fully supported GCOL paint mode was mode 0, which sets the pixel to the target colour value.  This is the default mode, and is used if no GCOL command has been issued.  There was limited support for mode 4, which inverts the pixel, but this was only supported for straight line drawing operations.

As of Console8 VDP 2.6.0, the following modes are now available for all currently supported plot operations:

| Mode | Effect |
| ---- | ------ |
| 0 | Set on-screen pixel to target colour value |
| 1 | OR value with the on-screen pixel |
| 2 | AND value with the on-screen pixel |
| 3 | EOR value with the on-screen pixel |
| 4 | Invert the on-screen pixel |
| 5 | No operation |
| 6 | AND the inverse of the specified colour with the on-screen pixel |
| 7 | OR the inverse of the specified colour with the on-screen pixel

PLOT commands using an "inverse" plot code are essentially identical to setting a GCOL paint mode of 4, and will temporarily override the current GCOL paint mode if a different GCOL paint mode is set.

NB the Agon does not currently support GCOL painting mode numbers outside of the range 0-7; any command sent with a greater number will be ignored.  On Acorn systems these could be used for a number of other actions, such as pattern fills.


## Line drawing (PLOT codes &00-&3F)

For all of the line drawing plot codes, the position given with the PLOT command is the end point of the line.  The start point of the line is the last position on the graphics cursor stack.

Support for plotting dotted lines was added in Agon Console8 VDP 2.7.0.  The default pattern is 1 pixel on, 1 pixel off, but this can be changed using the `VDU 23, 6, n1-n8` command which takes 8 bytes of data.  The length of pattern can be set using `VDU 23, 0, 242, n` where n is the number of pixels in the pattern.  Setting a length of zero will reset to the default pattern with a length of 8.  When drawing a line, the pattern is repeated as many times as necessary to draw the line and will be used from the top-most bit of the pattern data, repeating as necessary to draw the whole line.


## Line fill codes (PLOT codes &48-&4F, &58-5F, &68-6F, &78-&7F)

These various PLOT codes will fill horizontal lines on the screen.  When executing a line fill, the graphics system will scan the line to find appropriate start and end positions, depending on the PLOT code used and the screen contents.  The final calculated end position will be pushed to the graphics cursor stack.


## Flood fill codes (PLOT codes &80-&87, &87-8F)

These PLOT codes will perform a flood fill from the given seed location.

NB owing to the nature of how the graphics system works on the Agon, the various line fill and flood fill PLOT codes will be executed immediately, causing any other queued drawing operations to also be performed.  This is necessary in order to ensure that the graphics cursor stack is kept properly up to date.  In the future we may provide an option to allow for the graphics cursor stack update to be skipped and allow these PLOTs to be queued.


## Filled triangles (PLOT codes &50-&57)

Filled triangles are drawn using the last two positions on the graphics cursor stack and the position given with the PLOT command.

The behaviour of the triangle fill command has changed slightly in Agon Console8 VDP 2.6.0, and you may observe some slight differences in the exact pixels drawn when using this command.  Unfortunately the old behaviour was not compatible with the new GCOL paint modes, and so the behaviour had to be changed.  The behaviour is consistent with general "best practice" for triangle fill algorithms, and so should be more predictable and reliable.  As an example, with the revised triangle plotting it is possible to draw a fan of triangles using an EOR paint mode and you will not see the edges of the triangles "cancel each other out" as you would have done with the old behaviour.

This new behaviour is not entirely consistent with the BBC Micro or GXR ROM, but it is more consistent with modern graphics systems and should be more predictable and reliable.  In practice, the differences are minimal and should not affect most programs.


## Filled rectangles (PLOT codes &60-&67)

Filled rectangles are drawn using the last position on the graphics cursor stack and the position given with the PLOT command to define two corners of a rectangle.


## Filled parallelograms (PLOT codes &70-&77)

Parallelograms require three points to be defined.  They will therefore use the last two points pushed to the graphics cursor stack, coupled with the position given with the PLOT command.

The three given points represent any three corners of the parallelogram.  The first and last (third) points are used to define opposing corners of the parallelogram.  The graphics system will calculate the fourth corner of the parallelogram as a point opposite to the second corner, and then fill the shape.


## Circle drawing (PLOT codes &90-&9F)

Circle drawing requires two points to be defined.  They will therefore use the last point pushed to the graphics cursor stack, coupled with the position given with the PLOT command.  The first point is the centre of the circle, and the second point is a point on the circumference of the circle.


## Arcs, segments, and sectors (PLOT codes &A0-&B7) §§§§

Arcs, segments, and sectors require three points to be defined.  They will therefore use the last two points pushed to the graphics cursor stack, coupled with the position given with the PLOT command.

Arcs, segments, and sectors are all drawn anticlockwise from a start point to an end.

The first point defines the start of the arc, segment, or sector, which is a point on the circumference of the circle.  The second point defines the centre of the circle that the arc, segment, or sector is part of.  The distance from the first point to the second point defines the radius of the circle.  The final point (provided with the PLOT command) defines the end of the arc, segment, or sector.  This point does not have to be on the circumference of the circle, and the graphics system will calculate the point on the circumference that the arc, segment, or sector ends at.  Instead the final point is effectively used to define the angle of the arc.


## Ellipses (PLOT codes &C0-&C7, &C8-&CF)

Ellipse drawing also require three points to be defined.  They use the last two points pushed to the graphics cursor stack, coupled with the position given with the PLOT command.

The first point will be used to mark the centre of the ellipse.  The second point is an "outermost" point at the same height as the centre point of the ellipse you wish to draw.  The final point provided with the PLOT command is either the highest or lowest point of the ellipse.

This allows you to draw ellipses that are not axis-aligned, i.e. it supports drawing a "sheared" ellipse.


## Rectangle copy/move (PLOT codes &B8-&BF)

These PLOT codes are interpreted slightly differently from others.  The purpose of these commands is to copy or move a rectangle of pixels from one location to another.  The rectangle is defined by the last two positions on the graphics cursor stack, and the position provided with the PLOT command is the destination to copy the rectangle to.

A "move" operation will copy the rectangle to the new location, and then clear the original rectangle using the currently defined background colour.  (NB the background GCOL paint mode is _not_ used when filling the original rectangle - the pixels will be set with the background colour).  A "copy" operation will copy the rectangle to the new location, but leave the original rectangle in place.

| PLOT code | Effect |
| --------- | ------ |
| &B8 | Move cursor relative to last position |
| &B9 | Relative rectangle move |
| &BA | Relative rectangle copy |
| &BB | Relative rectangle copy |
| &BC | Move cursor to absolute position |
| &BD | Absolute rectangle move |
| &BE | Absolute rectangle copy |
| &BF | Absolute rectangle copy |


## Fill path - Experimental (PLOT codes &D8-&DF) §§§§

The ability to draw a filled path was added in the Agon Console8 VDP 2.7.0 release.  This command was not part of Acorn's original PLOT command set, either on the BBC Micro, in their Graphics Extension ROM, or in the later Acorn Archimedes operating systems.  For now, it should therefore be considered to be experimental.  Whilst at the time of writing this documentation it is thought that this command is not likely to change, it _is_ possible that the command may be removed or altered in future releases.

This command will fill a path defined by an arbitrary number of points.  The path must be at least three points long.

As the path is is of an arbitrary length the nature of this commands operation differs from other PLOT commands.

All new paths will be started with the last two points pushed to the graphics cursor stack, plus the position given with the PLOT command.  The path thus starts with three points and therefore, if no subsequent commands are received to extend the path, a triangle will be drawn.

If the next VDU command received immediately after a fill path plotting command is also a matching "Fill path" PLOT command then the path will be extended with the new position given with the PLOT command.  If the next VDU command is not a matching "Fill path" PLOT command then the path will be considered to be complete, i.e. the path will be closed by connecting the last point to the first one and the graphics system will then fill the area defined by the path.

Please note that any "move" PLOT command will be interpreted as closing the current path.  Whilst absolute and relative positioning PLOT commands can be combined when building up a path, changing between different codes (such as from "foreground" to "background" PLOTs) will be interpreted as closing the current path and starting a new path.

If any other VDU commands are received after a "Fill path" PLOT command, then the path will be considered to be complete and the graphics system will fill the area defined by the path.  Similarly if there is a significant delay between the VDP receiving "fill path" PLOT commands, even if the next command uses a matching PLOT code, then the path will be considered to be complete and the graphics system will fill the area defined by the path.

As a result of this mode of operation, it is not possible to interactively enter multiple "fill path" PLOT commands one after another at the BASIC command prompt using separate commands over multiple lines, as the system will interpret the character output for command entry as closing the current path.  Instead filled paths must be defined in a single command sequence, uninterrupted by other VDU commands, usually as part of a program.


## Bitmap plots (PLOT codes &E8-&EF) {#bitmap-plot}

Bitmap plots will draw the currently selected bitmap to the screen.

Before a bitmap plot can be used a valid bitmap must be selected using either `VDU 23, 27, 0, id` or `VDU 23, 27, &20, bufferId;`.

Bitmap plots will only draw non-transparent pixels to the screen.  When using "foreground" PLOTs, the bitmap is drawn to screen using the current foreground GCOL painting mode using the colour from the bitmap.  When using "background" PLOTs, the bitmap is drawn to screen using the current background GCOL painting mode using the currently selected background colour.  Inverse plot modes invert on-screen pixels that correspond to non-transparent pixels in the bitmap.

As of Console8 VDP version 2.9.0 it is possible to set an affine transform matrix to be applied to bitmap plots.  This allows for bitmaps to be scaled, rotated, sheared, and translated when being drawn onto the screen.  The transform matrix is set using `VDU 23, 0, &96, 1, bufferId;` which is documented in the [System Commands](System-Commands.md) documentation.  The buffered commands API provides commands to set and manipulate the transform matrix, and is documented in the [Buffered Commands](Buffered-Commands-API.md) documentation.


## PLOT support prior to VDP 1.04

Prior to VDP 1.04 the PLOT command support was very limited, did not support relative positioning, and had buggy and incorrect interpretations of some of the PLOT codes.  As of VDP 1.04 the PLOT command support has been greatly improved, and is now much more compatible with the BBC Micro and GXR ROM.

The most significant bug was the mis-interpretation of several "move" commands to instead be "draw".  As a result, some programs written for earlier versions of the VDP firmware may no longer work correctly.  Fixing these programs however is usually very straightforward and usually just involves changing the PLOT code used.


## Compatibility

If the VDP does not recognise a plot operation, it will be ignored.  This can allow you to write programs that "feature detect" whether the VDP supports a particular plot operation, and then use it if it does, or fall back to an alternative if it does not.
