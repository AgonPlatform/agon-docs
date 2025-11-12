VDP Buffered Commands API
=========================

The VDP Buffered Commands API is a low-level API that allows for the creation of buffers on the VDP.  These buffers can be used for sequences of commands for later execution, storing data, capturing output from the VDP, as well as storing bitmaps, sound samples and fonts.

Through the use of the APIs, it is possible to both send commands to the VDP in a "packetised" form, as well as to have "functions" or "stored procedures" that can be saved on the VDP and executed later.

These commands are collected under `VDU 23, 0, &A0, bufferId; command, [<arguments>]`.

Examples below are given in BBC BASIC.

A common source of errors when sending commands to the VDP from BASIC via VDU statements is to forget to use a `;` after a number to indicate a 16-bit value should be sent.  If you see unexpected behaviour from your BASIC code that is the most likely source of the problem.

All commands must specify a buffer ID as a 16-bit integer.  There are 65534 buffers available for general use, with one buffer ID (number 65535) reserved for special functions, and is generally interpreted as meaning "current buffer".  As with all other VDP commands, these 16-bit values are sent as two bytes in little-endian order, and are documented as per BBC BASIC syntax, such as `bufferId;`.

On a restart all buffers will be empty.  One should not assume however that buffers are empty when your program is run, as other programs may have already used the buffers.  Indeed, it is a valid use case to have a "loader" program that is designed to be run before another program to prepare a set of buffers for that second program to use.  It is therefore advisable to clear out the buffers before use.

A single buffer can contain multiple blocks.  This approach allows for a buffer to be gradually built up over time, and for multiple commands to be sent to the VDP in a single packet.  This can be useful for sending large amounts of data to the VDP, such as a large bitmap or a sound sample, or using smaller blocks to contain for command sequences for more easily referencing an individual command (or indeed even fragments of command sequences).  By breaking up large data into smaller packets it is possible to avoid blocking the screen for long periods of time, allowing for a visual indicator of progress to be made to the user.

Many of the commands accept an offset within a buffer.  An offset is typically a 16-bit value, however as buffers can be larger than 64kb an "advanced" offset mode is provided.  This advanced mode allows for offsets to be specified as 24-bit values, and also provides for a mechanism to refer to individual blocks within a buffer.  When this mode is used, the offset is sent as 3 bytes in little-endian order.  If the top bit of an advanced offset is set, this indicates that _following_ the offset value there will be a 16-bit block number, with the remaining 23-bit offset value to be applied as an offset within the indicated block.  Using block offsets can be useful for modifying commands within buffers, as using block offsets can make identifying where parameters are placed within commands much easier to work out.

At this time the VDP Buffered Commands API does not send any messages back to MOS to indicate the status of a command, or support a mechanism for sending contents of buffers back to MOS.  This will likely change in the future, but that will require changes to agon-mos to support it.

## Command 0: Write block to a buffer {#command-0}

`VDU 23, 0 &A0, bufferId; 0, length; <buffer-data>`

This command is used to store a data block (a sequence of bytes) in a buffer on the VDP.  The exact nature of this data may vary.  It could be a sequence of VDU commands which can be executed later, a bitmap, a sound sample, or just a sequence of bytes.  When used for a sequence of VDU commands, this effectively allows for functions or stored procedures to be created.

This is the most common command to use to send data to the VDP.  Typically you will call command 2 first to ensure that the buffer is empty, and then make a series of calls to this command to send data to the buffer.

The `bufferId` is a 16-bit integer that identifies the buffer to write to.  Writing to the same buffer ID multiple times will add new blocks to that buffer.  This allows a buffer to be built up over time, essentially allowing for a command to be sent across to the VDP in multiple separate packets.

Whilst the length of an individual block added using this command is restricted to 65535 bytes (as the largest value that can be sent in a 16-bit number) the total size of a buffer is _not_ restricted to this size, as multiple blocks can be added to a buffer.  Given how long it takes to send data to the VDP it is advisable to send data across in smaller chunks, such as 1kb of data or less at a time.

As writing to a single buffer ID is cumulative with this command, care should be taken to ensure that the buffer is cleared out before writing to it.

When building up a complex sequence of commands it is often advisable to use multiple blocks within a buffer.  Typically this is easier to code, as otherwise working out exactly how many bytes long a command sequence is can be can be onerously difficult.  It is also easier to modify a command sequences that are broken up into multiple blocks.

As mentioned above it is advisable to send large pieces of data, such as bitmaps or sound samples, in smaller chunks.  In between each packet of data sent to a buffer, the user can then perform other operations, such as updating the screen to indicate progress.  This allows for long-running operations to be performed without blocking the screen, and larger amounts of data to be transferred over to the VDP than may otherwise be practical given the limitations of the eZ80.

If a buffer ID of 65535 is used then this command will be ignored, and the data discarded.  This is because this buffer ID is reserved for special functions.

### Using buffers for bitmaps

Whilst it is advisable to send bitmaps over in multiple blocks, they cannot be _used_ if they are spread over multiple blocks.  To use a bitmap its data must be in a single contiguous block, and this is achieved by using the "consolidate" command `&0E`.

Once you have a block that is ready to be used for a bitmap, the buffer must be selected, and then a bitmap created for that buffer using the bitmap and sprites API.  This is done with the following commands:

```
VDU 23, 27, &20, bufferId;              : REM Select bitmap (using a buffer ID)
VDU 23, 27, &21, width; height; format  : REM Create bitmap from buffer
```

More extensive information on the bitmap and sprites API calls can be found in the [bitmaps and sprites documentation](Bitmaps-API.md).

Until the "create bitmap" call has been made the buffer cannot be used as a bitmap.  That is because the system needs to understand the dimensions of the bitmap, as well as the format of the data.  Usually this only needs to be done once.  The format is given as an 8-bit value, with the following values supported:

| Value | Type     | Description |
| ----- | -------- | ----------- |
| 0     | RGBA8888 | RGBA, 8-bits per channel, with bytes ordered sequentially for red, green, blue and alpha |
| 1     | RGBA2222 | RGBA, 2-bits per channel, with bits ordered from highest bits as alpha, blue, green and red |
| 2     | Mono/Mask | Monochrome, 1-bit per pixel |

The existing bitmap API uses an 8-bit number to select bitmaps, and these are automatically stored in buffers numbered 64000-64255 (`&FA00`-`&FAFF`).  Working out the buffer number for a bitmap is simply a matter of adding 64000.  All bitmaps created with that API will be RGBA8888 format.

There is one other additional call added to the bitmap and sprites API, which allows for bitmaps referenced with a buffer ID to be added to sprites.  This is done with the following command:

```
VDU 23, 27, &26, bufferId;              : REM Add bitmap to the current sprite
```

This command otherwise works identically to `VDU 23, 27, 6`.

It should be noted that it is possible to modify the buffer that a bitmap is stored in using the "adjust buffer contents" and "reverse contents" commands (`5` and `24` respectively).  This can allow you to do things such as changing colours in a bitmap, or flipping an image horizontally or vertically.  This will even work on bitmaps that are being used inside sprites.

Using commands targeting a buffer that create new blocks, such as "consolidate" or "split", will invalidate the bitmap and remove it from use.

### Using buffers for sound samples

Much like with bitmaps, it is advisable to send samples over to the VDP in multiple blocks for the same reasons.

In contrast to bitmaps, the sound system can play back samples that are spread over multiple blocks, so there is no need to consolidate buffers.  As a result of this, the sample playback system is also more tolerant of modifications being made to the buffer after a sample has been created from it, even if the sample is currently playing.  It should be noted that splitting a buffer may result in unexpected behaviour if the sample is currently playing, such as skipping to other parts of the sample.

Full information on the sound system can be found in the [audio API documentation](Enhanced-Audio-API.md).

Once you have a buffer that contains block(s) that are ready to be used for a sound sample, the following command must be used to indicate that a sample should be created from that buffer:

```
VDU 23, 0, &85, 0, 5, 2, bufferId; format
```

The `format` parameter is an 8-bit value that indicates the format of the sample data.  The following values are supported:

| Value | Description |
| ----- | ----------- |
| 0     | 8-bit signed, 16KHz |
| 1     | 8-bit unsigned, 16KHz |

Once a sample has been created in this way, the sample can be selected for use on a channel using the following command:

```
VDU 23, 0, &85, channel, 4, 8, bufferId;
```

Samples uploaded using the existing "load sample" command (`VDU 23, 0, &85, sampleNumber, 5, 0, length; lengthHighByte, <sample data>`) are also stored in buffers automatically.  A sample number using this system is in the range of -1 to -128, but these are stored in the range 64256-64383 (`&FB00`-`&FB7F`).  To map a number to a buffer range, you need to negate it, subtract 1, and then add it to 64256.  This means sample number -1 is stored in buffer 64256, -2 is stored in buffer 64257, and so on.

## Command 1: Call a buffer

`VDU 23, 0 &A0, bufferId; 1`

This command will attempt to execute all of the commands stored in the buffer with the given ID.  If the buffer does not exist, or is empty, then this command will do nothing.

Essentially, this command passes the contents of the buffer to the VDP's VDU command processor system, and executes them as if they were sent directly to the VDP.

As noted against command 0, it is possible to build up a buffer over time by sending across multiple commands to write to the same buffer ID.  When calling a buffer with multiple blocks, the blocks are executed in order.

Care should be taken when using this command within a buffer, as it is possible to create an infinite loop.  For instance, if a buffer contains a command to call itself, then this will result in an infinite loop.  This will cause the VDP to hang, and the only way to recover from this is to reset the VDP.

Using a `bufferId` of -1 (65535) will cause the current buffer to be executed.  This can be useful for creating loops within a buffer.  It will be ignored if used outside of a buffered command sequence.

## Command 2: Clear a buffer {#command-2}

`VDU 23, 0 &A0, bufferId; 2`

This command will clear the buffer with the given ID.  If the buffer does not exist then this command will do nothing.

Please note that this clears out all of the blocks sent to a buffer via command 0, not just the last one.  i.e. if you have built up a buffer over time by sending multiple commands to write to the same buffer ID, this command will clear out all of those commands.

Calling this command with a `bufferId` value of -1 (65535) will clear out all buffers.

## Command 3: Create a writeable buffer {#create-buffer}

`VDU 23, 0 &A0, bufferId; 3, length;`

NB it is quite rare that you will want to use this command.  In general, new buffers are best created using the [write buffer](#command-0) command.  You do not need to use this command to create a buffer before using the [write buffer](#command-0) command; doing so will usually lead to errors as you will end up with _two_ blocks in the buffer, the first of which will be empty.  The use of this command to create a buffer in which to store reference data can largely be replaced through the use of [VDP Variables](./VDP-Variables.md).

This command will create a new writeable buffer with the given ID.  If a buffer with the given ID already exists then this command will do nothing; you may therefore wish to [clear the buffer](#command-2) first before using this command.  This command was primarily intended for use to create a buffer that can be used to capture output using the ["set output stream"](#command-4) command, or to store data that can be used for other commands.

If you do wish to use this command to create a buffer to store program data, after using this command you will typically use the "set" operation in the ["buffer adjust"](#command-5) command to change the buffer's contents.  As the [buffer adjust](#command-5) command does not require a "writeable buffer" this is not recommended; it is generally easier to use command 0 to write a data block to a buffer.

This new buffer will be a single empty single block upon creation, containing zeros.

The `length` parameter is a 16-bit integer that specifies the maximum size of the buffer.  This is the maximum number of bytes that can be stored in the buffer.  If the buffer is full then no more data can be written to it, and subsequent writes will be ignored.

After creating a buffer with this command it is possible to use the [write buffer](#command-0) command to write further blocks to the buffer, however this is _probably_ not advisable.

Using this command with a `bufferId` of `65535` (-1) and `0` will be ignored, as these values have special meanings for writable buffers.  See [command 4](#command-4).

## Command 4: Set output stream to a buffer {#command-4}

`VDU 23, 0 &A0, bufferId; 4`

NB: The original intent of this command was to provide a way for buffered command sequences to access to VDP state information.  In general this is no longer necessary, as most of this data can now be accessed via [VDP Variables](./VDP-Variables.md).  As of VDP 2.15, the primary use case for this command is to temporarily prevent the VDP from sending data to MOS when certain commands are being executed.

This command changes the current output stream of the VDP, allowing [VDP protocol](./System-Commands.md#vdp-serial-protocol) data packets to be send to a buffer, or prevented from being sent, instead of being sent to MOS.

The default output stream for the main VDU command processor is the communications channel from the VDP to MOS running on the eZ80.

Changes made with this command apply to the current command stream, and any other buffered command sequences that are called from within that sequence.  Once the buffered command sequence has completed, the output stream will be reset to its original value.  (Please note that from VDP 2.8 until 2.14 a bug meant that this behaviour was not observed, and any changes to the output stream applied globally and permanently.  This has been fixed in VDP 2.15.)

The target buffer must be a writable buffer created with [command 3](#create-buffer), or one of two special buffer IDs.  If the buffer does not exist, or the first block within the target buffer is not writable, then this command will do nothing.

The two special buffer IDs are:

- `65535` (-1): This removes/detaches the output stream.  Any subsequent interactions that cause VDP protocol packets to be sent will have those responses discarded/ignored.
    * This includes both responses from VDU commands and user interactions such as keyboard or mouse update events.
- `0`: This will set the output stream back to its original value for the current command stream.  Typically that will be the communications channel from the VDP to MOS running on the eZ80, but this may not be the case if a nested call has been made.
    * NB From VDP 2.8 until 2.14 setting the output stream to 0 would always set the output stream back to the MOS communication channel.  This was a bug that has been fixed in VDP 2.15.

It should be noted that writable buffers can only be written to until the end of the buffer has been reached; once that happens no more data will be written to the buffer.  It is not currently possible to "rewind" an output stream.  If you are using the command to capture VDP protocol pavkets you should ensure that the buffer is large enough to capture all of the data that is expected to be written to it.  The only current way to "rewind" an output stream would be to clear the buffer and create a new one, and then call set output stream again with the newly created buffer.

This command should be used with care, as it can cause unexpected behaviour if used incorrectly.  It is strongly recommended to only use this command from within a buffered command sequence.  If you do use it outside of a buffered command sequence, it is important to remember to restore the original output channel using `VDU 23, 0, &A0, 0; 4`.  Failing to do this will mean VDP will stop communicating with MOS, and so basic information such as key presses won't be sent across, effectively causing your Agon to hang.

As of VDP 2.15 when this command is used outside of a buffered command sequence the change will be temporary, protecting you from the hanging scenario that has just been described.  As soon as the VDP detects that the current command stream has completed (i.e. there are no pending commands, and a screen refresh has been performed) the output stream will be restored to communicate with MOS.  This can be useful for performing some operations that will set [VDP Variables](./VDP-Variables.md) without having to worry about them also sending data back to MOS.

## Command 5: Adjust buffer contents {#command-5}

`VDU 23, 0, &A0, bufferId; 5, operation, offset; [count;] <operand>, [arguments]`

This command will adjust the contents of a buffer, at a given offset.  The exact nature of the adjustment will depend on the operation used.

Passing a `bufferId` of -1 (65535) to this command will adjust the contents of the current buffer.  This will only work if this command is used within a buffered command sequence, otherwise the command will not do anything.

The basic set of adjustment operations are as follows:

| Operation | Description |
| --------- | ----------- |
| 0 | NOT |
| 1 | Negate |
| 2 | Set value |
| 3 | Add |
| 4 | Add with carry |
| 5 | AND |
| 6 | OR |
| 7 | XOR |

All of these operations will modify a byte found at the given offset in the buffer.  The only exception to that is the "Add with carry" operation, which will also store the "carry" value in the byte at the _next_ offset.  With the exception of NOT and Negate, each command requires an operand value to be specified.

To flip the bits of a byte at offset 12 in buffer 3, you would need to use the NOT operation, and so the following command would be used:
```
VDU 23, 0, &A0, 3; 5, 0, 12;
```

To add 42 to the byte at offset 12 in buffer 3, you would need to use the Add operation, and so the following command would be used:
```
VDU 23, 0, &A0, 3; 5, 3, 12; 42
```

When using add with carry, the carry value is stored in the byte at the next offset.  So to add 42 to the byte at offset 12 in buffer 3, and store the carry value in the byte at offset 13, you would need to use the Add with carry operation, and so the following command would be used:
```
VDU 23, 0, &A0, 3; 5, 4, 12; 42
```

### Advanced operations

Whilst these operations are useful, they are not particularly powerful as they only operate one one byte at a time, with a fixed operand value, and potentially cannot reach all bytes in a buffer.  To address this, the API supports a number of advanced operations.

The operation value used is an 8-bit value that can have bits set to modify the behaviour of the operation.  The following bits are defined:

| Bit | Description |
| --- | ----------- |
| &10 | Use "advanced" offsets |
| &20 | Operand is a buffer-fetched value (buffer ID and an offset) |
| &40 | Multiple target values should be adjusted |
| &80 | Multiple operand values should be used |

These bits can be combined together to modify the behaviour of the operation.

Fundamentally, this command adjusts values of a buffer at a given offset one byte at a time.  When either of the "multiple" variants are used, a 16-bit `count` must be provided to indicate how many bytes should be altered.

Advanced offsets are sent as a 24-bit value in little-endian order, which can allow for buffers that are larger than 64kb to be adjusted.  If the top-bit of this 24-bit value is set, then the 16-bit value immediately following the offset is used as a block index number, and the remaining 23-bits of the offset value are used as an offset within that block.  When the "advanced" offset mode bit has been set then all offsets associated with this command must be sent as advanced offsets.

The "buffer-fetched value" mode allows for the operand value to be fetched from a buffer.  The operand sent as part of the command in this case is a pair of 16-bit values giving the buffer ID and offset to indicate where the actual operand value should be fetched from.  An operand buffer ID of -1 (65535) will be interpreted as meaning "this buffer", and thus can only be used inside a buffered command sequence.  If the advanced offset mode is used, then the operand value is an advanced offset value.

The "multiple target values" mode allows for multiple bytes to be adjusted at once.  When this mode is used, the `count` value must be provided to indicate how many bytes should be adjusted.  Unless the "multiple operand values" mode is also used, the operand value is used for all bytes adjusted.

The "multiple operand values" mode allows for multiple operand values to be used.  When this mode is used, the `count` value must be provided to indicate how many operand values should be used.  This can allow, for instance, to add together several bytes in a buffer.  When this mode is used in conjunction with the "multiple target values" mode, the number of operand values must match the number of target values, and the operation happens one byte at a time.

Some examples of advanced operations are as follows:

Flip the bits of 7 bytes in buffer 3 starting at offset 12:
```
VDU 23, 0, &A0, 3; 5, &40, 12; 7;
```
This uses operation 0 (NOT) with the "multiple target values" modifier (&40).

Add 42 to each of the 7 bytes in buffer 3 starting at offset 12:
```
VDU 23, 0, &A0, 3; 5, &43, 12; 7; 42
```

Set the byte at offset 12 in the fourth block of buffer 3 to 42:
```
VDU 23, 0, &A0, 3; 5, &12, 12; &80, 4; 42
```
This is using operation 2 (Set) with the "advanced offsets" modifier (&10).  As BBC BASIC doesn't natively understand how to send 24-bit values it is sent as the 16-bit value `12;` followed by a byte with its top bit set `&80` to complete the 24-bit offset in little-endian order.  As the top bit of the offset is set, this indicates that the next 16-bit value will be a block index, `4;`.  Finally the value to write is sent, `42`.

An operation like this could be used to set the position as part of a draw command.

Set the value in buffer 3 at offset 12 to the sum of the five values 1, 2, 3, 4, 5:
```
VDU 23, 0, &A0, 3; 5, 2, 12; 0  : REM clear out the value at offset 12 (set it to 0)
VDU 23, 0, &A0, 3; 5, &83, 12; 5; 1, 2, 3, 4, 5
```

AND together 7 bytes in buffer 3 starting at offset 12 with the 7 bytes in buffer 4 starting at offset 42:
```
VDU 23, 0, &A0, 3; 5, &E5, 12; 7; 4; 42;
```

As we are working on a little-endian system, integers longer than one byte are sent with their least significant byte first.  This means that the add with carry operation can be used to add together integers of any size, so long as they are the same size.  To do this, both the "multiple target values" and "multiple operand values" modes must be used.

The following commands will add together a 16-bit, 24-bit, 32-bit, and 40-bit integers, all targeting the value stored in buffer 3 starting at offset 12, and all using the operand value of 42:
```
VDU 23, 0, &A0, 3; 5, &C4, 12; 2; 42;  : REM 2 bytes; a 16-bit integer
VDU 23, 0, &A0, 3; 5, &C4, 12; 3; 42; 0  : REM 3 bytes; a 24-bit integer
VDU 23, 0, &A0, 3; 5, &C4, 12; 4; 42; 0;  : REM 4 bytes; a 32-bit integer
VDU 23, 0, &A0, 3; 5, &C4, 12; 5; 42; 0; 0  : REM 5 bytes; a 40-bit integer
```
Take note of how the operand value is padded out with zeros to match the size of the target value.  `42;` is used as a base to send a 16-bit value, with zeros added of either 8-bit or 16-bits to pad it out to the required size.  The "carry" value will be stored at the next offset in the target buffer after the complete target value.  So for a 16-bit value, the carry will be stored at offset 14, for a 24-bit value it will be stored at offset 15, and so on.


## Command 6: Conditionally call a buffer {#command-6}

`VDU 23, 0, &A0, bufferId; 6, operation, <checkBufferId; checkOffset; | vduVariableId;> [arguments]`

This command will conditionally call a buffer if the condition operation passes.  This command works in a similar manner to the "Adjust buffer contents" command.

With this command a buffer ID of 65535 (-1) is always interpreted as "current buffer", and so can only be used within a buffered command sequence.  If used outside of a buffered command sequence then this command will do nothing.

The basic set of condition operations are as follows:

| Operation | Description |
| --------- | ----------- |
| 0 | Exists (value is non-zero) |
| 1 | Not exists (value is zero) |
| 2 | Equal |
| 3 | Not equal |
| 4 | Less than |
| 5 | Greater than |
| 6 | Less than or equal |
| 7 | Greater than or equal |
| 8 | AND |
| 9 | OR |

The value that is being checked is fetched from the specified check buffer ID and offset, or from a [VDP Variable](VDP-Variables.md).  With the exception of "Exists" and "Not exists", each command requires an operand value argument to be specified to check against.

The operation value used is an 8-bit value that can have bits set to modify the behaviour of the operation.  The following bits are defined:

| Bit value | Description |
| --- | ----------- |
| &10 | Use advanced offsets |
| &20 | Operand value argument is a buffer-fetched value (buffer ID and an offset) |
| &40 | Value to be checked is a [VDP Variable](VDP-Variables.md) * |
| &80 | Compare 16-bit values * |

\* These flag bits are only supported from VDP 2.12.0 onwards.

These modifiers can be combined together to modify the behaviour of the operation.

When bit `&20` is set on the condition operation byte, the value to check against is fetched from a buffer, so the arguments provided must be a buffer ID and an offset (which can be an advanced offset, if the appropriate bit is set).

Up to and including VDP 2.11.0, all comparisons would be conducted against 8-bit values only.  As of VDP 2.12.0 it is possible to specify that the values to compare are 16-bit values by setting bit `&80` in the operation byte, in which case the operand value argument must be a 16-bit value.

Comparisons of values larger than 16-bit are not directly supported.  If you need to compare larger values, you will need to break them down into smaller values and compare them separately, splitting your comparisons across multiple buffers.

If the value to be checked is a VDP Variable, then the check buffer ID argument and offset should be omitted and instead a 16-bit variable ID sent in their place.  The "exists" and "not exists" operations for variables are a true existance check, rather than checking the value against zero.  For all other operations, if the variable does not exist then the operation result will be treated as false.

(When the value to be checked is _not_ a VDP variable, a `checkOffset;` argument must always be provided.  As with other commands, this can be an advanced offset.)

The `AND` and `OR` operations are logical operations, and so the operand value is used as a boolean value.  Any non-zero value is considered to be true, and zero is considered to be false.  These operations therefore are most useful when used with buffer-fetched operand values (operations &28, &29, &38 and &39).

Some examples of condition operations are as follows:

Call buffer 7 if the value in buffer 12 at offset 5 exists (is non-zero):
```
VDU 23, 0, &A0, 7; 6, 0, 12; 5;
```

Call buffer 8 if the value in buffer 12 at offset 5 does not exist (is zero):
```
VDU 23, 0, &A0, 8; 6, 1, 12; 5;
```

Combining the above two examples is effectively equivalent to "if the value exists, call buffer 7, otherwise call buffer 8":
```
VDU 23, 0, &A0, 7; 6, 0, 12; 5;
VDU 23, 0, &A0, 8; 6, 1, 12; 5;
```

Call buffer 3 if the value in buffer 4 at offset 12 is equal to 42:
```
VDU 23, 0, &A0, 3; 6, 2, 4; 12; 42
```

Call buffer 5 if the value in buffer 2 at offset 7 is less than the value in buffer 2 at offset 8:
```
VDU 23, 0, &A0, 5; 6, &24, 2; 7; 2; 8;
```

## Command 7: Jump to a buffer {#command-7}

`VDU 23, 0, &A0, bufferId; 7`

This command will jump to the buffer with the given ID.  If the buffer does not exist, or is empty, then this command will do nothing.

This essentially works the same as the call command (command 1), except that it does not return to the caller.  This command is therefore useful for creating loops.

Using this command to jump to buffer 65535 (buffer ID -1) is treated as a "jump to end of current buffer".  This will return execution to the caller, and can be useful for exiting a loop.

## Command 8: Conditional Jump to a buffer {#command-8}

`VDU 23, 0, &A0, bufferId; 8, operation, checkBufferId; checkOffset; [arguments]`

This command operates in a similar manner to the "Conditionally call a buffer" command (command 6), except that it will jump to the buffer if the condition operation passes.

As with the "Jump to a buffer" command (command 7), a jump to buffer 65535 is treated as a "jump to end of current buffer".

## Command 9: Jump to an offset in a buffer

`VDU 23, 0, &A0, bufferId; 9, offset; offsetHighByte, [blockNumber;]`

This command will jump to the given offset in the buffer with the given ID.  If the buffer does not exist, or is empty, then this command will do nothing.

The offset in this command is always an "advanced" offset, given as a 24-bit value in little-endian order.  As with other uses of advanced offsets, if the top-bit is set in the high byte of the offset value, a block number must also be provided.

When jumping to an offset, using buffer ID 65535 is treated as meaning "jump within current buffer".  This can be useful for creating loops within a buffer, or when building up command sequences that may be copied across multiple buffers.

Jumping to an offset that is beyond the end of the buffer is equivalent to jumping to the end of the buffer.

## Command 10: Conditional jump to an offset in a buffer {#command-10}

`VDU 23, 0, &A0, bufferId; 10, offset; offsetHighByte, [blockNumber;] operation, <checkBufferId; checkOffset; | vduVariableId;> [arguments]`

A conditional jump with an offset works in a similar manner to the "Conditional call a buffer" command (command 6), except that it will jump to the given offset in the buffer if the condition operation passes.

As with the "Jump to an offset in a buffer" command (command 9), the offset in this command is always an "advanced" offset, given as a 24-bit value in little-endian order, and the usual advanced offset rules apply.  And similarly, using buffer ID 65535 is treated as meaning "jump within current buffer".

## Command 11: Call buffer with an offset

`VDU 23, 0, &A0, bufferId; 11, offset; offsetHighByte, [blockNumber;]`

Works just like "Call a buffer" (command 1), except that it also accepts an advanced offset.

## Command 12: Conditional call buffer with an offset

`VDU 23, 0, &A0, bufferId; 12, offset; offsetHighByte, [blockNumber;] operation, <checkBufferId; checkOffset; | vduVariableId;> [arguments]`

Works just like the "Conditional call a buffer" command (command 6), except that it also accepts an advanced offset.

## Command 13: Copy blocks from multiple buffers into a single buffer {#command-13}

`VDU 23, 0, &A0, targetBufferId; 13, sourceBufferId1; sourceBufferId2; ... 65535;`

This command will copy the contents of multiple buffers into a single buffer.  The buffers to copy from are specified as a list of buffer IDs, terminated by a buffer ID of -1 (65535).  The buffers are copied in the order they are specified.

This is a block-wise copy, so the blocks from the source buffers are copied into the target buffer.  The blocks are copied in the order they are found in the source buffers.

The target buffer will be overwritten with the contents of the source buffers.  This will not be done however until after all the data has been gathered and copied.  The target buffer can therefore included in the list of the source buffers.

If a source buffer that does not exist is specified, or a source buffer that is empty is specified, then that buffer will be ignored.  If no source buffers are specified, or all of the source buffers are empty, then the target buffer will be cleared out.

The list of source buffers can contain repeated buffer IDs.  If a buffer ID is repeated, then the blocks from that buffer will be copied multiple times into the target buffer.

If there is insufficient memory available on the VDP to complete this command then it will fail, and the target buffer will be left unchanged.

## Command 14: Consolidate blocks in a buffer

`VDU 23, 0, &A0, bufferId; 14`

Takes all the blocks in a buffer and consolidates them into a single block.  This is useful for bitmaps, as it allows for a bitmap to be built up over time in multiple blocks, and then consolidated into a single block for use as a bitmap.

If there is insufficient memory available on the VDP to complete this command then it will fail, and the buffer will be left unchanged.

## Command 15: Split a buffer into multiple blocks

`VDU 23, 0, &A0, bufferId; 15, blockSize;`

Splits a buffer into multiple blocks.  The `blockSize` parameter is a 16-bit integer that specifies the target size of each block.  If the source data is not a multiple of the block size then the last block will be smaller than the specified block size.

If this command is used on a buffer that is already split into multiple blocks, then the blocks will be consolidated first, and then re-split into the new block size.

If there is insufficient memory available on the VDP to complete this command then it will fail, and the buffer will be left unchanged.

## Command 16: Split a buffer into multiple blocks and spread across multiple buffers

`VDU 23, 0, &A0, bufferId; 16, blockSize; [targetBufferId1;] [targetBufferId2;] ... 65535;`

Splits a buffer into multiple blocks, as per command 15, but then spreads the resultant blocks across the target buffers.  The target buffers are specified as a list of buffer IDs, terminated by a buffer ID of -1 (65535).

The blocks are spread across the target buffers in the order they are specified, and the spread will loop around the buffers until all the blocks have been distributed.  The target buffers will be cleared out before the blocks are spread across them.

What this means is that if the source buffer is, let's say, 100 bytes in size and we split using a block size of 10 bytes then we will end up with 10 blocks.  If we then spread those blocks across 3 target buffers, then the first buffer will contain blocks 1, 4, 7 and 10, the second buffer will contain blocks 2, 5 and 8, and the third buffer will contain blocks 3, 6 and 9.

This command attempts to ensure that, in the event of insufficient memory being available on the VDP to complete the command, it will leave the targets as they were before the command was executed.  However this may not always be possible.  The first step of this command is to consolidate the source buffer into a single block, and this may fail from insufficient memory.  If that happens then all the buffers will be left as they were.  After this however the target buffers will be cleared.  If there is insufficient memory to successfully split the buffer into multiple blocks then the call will exit, and the target buffers will be left empty.

## Command 17: Split a buffer and spread across blocks, starting at target buffer ID

`VDU 23, 0, &A0, bufferId; 17, blockSize; targetBufferId;`

As per the above two commands, this will split a buffer into multiple blocks.  It will then spread the blocks across buffers starting at the target buffer ID, incrementing the target buffer ID until all the blocks have been distributed.

Target blocks will be cleared before a block is stored in them.  Each target will contain a single block.  The exception to this is if the target buffer ID reaches 65534, as it is not possible to store a block in buffer 65535.  In this case, multiple blocks will be placed into buffer 65534.

With this command if there is insufficient memory available on the VDP to complete the command then it will fail, and the target buffers will be left unchanged.

## Command 18: Split a buffer into blocks by width

`VDU 23, 0, &A0, bufferId; 18, width; blockCount;`

This command splits a buffer into a given number of blocks by first of all splitting the buffer into blocks of a given width (number of bytes), and then consolidating those blocks into the given number of blocks.

This is useful for splitting a bitmap into a number of separate columns, which can then be manipulated individually.  This can be useful for dealing with sprite sheets.

## Command 19: Split by width into blocks and spread across target buffers

`VDU 23, 0, &A0, bufferId; 19, width; [targetBufferId1;] [targetBufferId2;] ... 65535;`

This command essentially operates the same as command 18, but the block count is determined by the number of target buffers specified.  The blocks are spread across the target buffers in the order they are specified, with one block placed in each target.

## Command 20: Split by width into blocks and spread across blocks starting at target buffer ID

`VDU 23, 0, &A0, bufferId; 20, width; blockCount; targetBufferId;`

This command essentially operates the same as command 18, but the generated blocks are spread across blocks starting at the target buffer ID, as per command 17.

## Command 21: Spread blocks from a buffer across multiple target buffers

`VDU 23, 0, &A0, bufferId; 21, [targetBufferId1;] [targetBufferId2;] ... 65535;`

Spreads the blocks from a buffer across multiple target buffers.  The target buffers are specified as a list of buffer IDs, terminated by a buffer ID of -1 (65535).  The blocks are spread across the target buffers in the order they are specified, and the spread will loop around the buffers until all the blocks have been distributed.

It should be noted that this command does not copy the blocks, and nor does it move them.  Unless the source buffer has been included in the list of targets, it will remain completely intact.  The blocks distributed across the target buffers will point to the same memory as the blocks in the source buffer.  Operations to modify data in the source buffer will also modify the data in the target buffers.  Clearing the source buffer however will not clear the target buffers.

## Command 22: Spread blocks from a buffer across blocks starting at target buffer ID

`VDU 23, 0, &A0, bufferId; 22, targetBufferId;`

Spreads the blocks from a buffer across blocks starting at the target buffer ID.

This essentially works the same as command 21, and the same notes about copying and moving blocks apply.  Blocks are spread in the same manner as commands 17 and 20.

## Command 23: Reverse the order of blocks in a buffer

`VDU 23, 0, &A0, bufferId; 23`

Reverses the order of the blocks in a buffer.

## Command 24: Reverse the order of data of blocks within a buffer

`VDU 23, 0, &A0, bufferId; 24, options, [valueSize;] [chunkSize;]`

Reverses the order of the data within the blocks of a buffer.  The `options` parameter is an 8-bit value where the bits set in that byte determine the operation's behavior.  The following bits are defined:

| Bit value | Description |
| --- | ----------- |
| 1   | Values are 16-bits in size |
| 2   | Values are 32-bits in size |
| 3 (1+2) | If both value size bits are set, then a `valueSize;` argument must be sent as a 16-bit value |
| 4   | Reverse data of the chosen value size within chunks of data, using a `chunkSize;` sent as a 16-bit value |
| 8   | Reverse blocks |

If no value size is set in the options byte (i.e. the bottom two bits are both zero) then the value size is assumed to be 8-bits.

To be clear, the table above documents how the bits are used inside the `options` byte.  These can be combined, so using a value of `6` for the options would indicate 32-bit values (bit 2) and reversing within chunks (bit 4), and a `chunkSize;` argument must be sent as a 16-bit value.

It is probably easiest to understand what this operation is capable of by going through some examples of how it can be used to manipulate bitmaps.  The VDP supports two different formats of color bitmap, either RGBA8888 which uses 4-bytes per pixel, i.e. 32-bit values, or RGBA2222 which uses a single byte per pixel.

The simplest example is rotating an RGBA2222 bitmap by 180 degrees, which can be done by just reversing the order of bytes in the buffer:
```
VDU 23, 0, &A0, bufferId; 24, 0
```

Rotating an RGBA8888 bitmap by 180 degrees is in principle a little more complex, as each pixel is made up of 4 bytes.  However with this command it is still a simple operation, as we can just reverse the order of the 32-bit values that make up the bitmap by using an options value of `2`:
```
VDU 23, 0, &A0, bufferId; 24, 2
```

Mirroring a bitmap around the x-axis is a matter of reversing the order of rows of pixels.  This can be done by setting a custom value size that corresponds to our bitmap width.  As can be seen from the table above, setting the bottom two bits of the `options` byte, which will be a value of `3`, will indicate that we are using a custom value size, so we will need to follow that with a `valueSize;` argument as a 16-bit value.  Therefore the following command will mirror an RGBA2222 bitmap around the x-axis:
```
VDU 23, 0, &A0, bufferId; 24, 3, width;
```

As an RGBA8888 bitmap uses 4 bytes per pixel we'd need to multiply our width by 4:
```
VDU 23, 0, &A0, bufferId; 24, 3, width * 4;
```

To mirror a bitmap around the y-axis, we need to reverse the order of pixels within each row.  That is achieved by setting the "reverse within chunks" bit in the options byte.  So if we send an options value of `4` we will need to send a `chunkSize;` argument to indicate the width of each row.  Therefore the following command will mirror an RGBA2222 bitmap around the y-axis:
```
VDU 23, 0, &A0, bufferId; 24, 4, width;
```

For an RGBA8888 bitmap we need to set our options to indicate 32-bit values as well as a custom chunk size:
```
VDU 23, 0, &A0, bufferId; 24, 6, width * 4;
```

NB in these last two examples our options indicate that we are only sending a `chunkSize;` argument, so they do not include a `valueSize;` argument.  It is possible to combine both by using an options value of `7` - the uses of which are left as an exercise for the reader.

## Command 25: Copy blocks from multiple buffers by reference

`VDU 23, 0, &A0, targetBufferId; 25, sourceBufferId1; sourceBufferId2; ...; 65535;`

This command is essentially a version of [command 13](#command-13) that copies blocks by reference rather than by value.  The parameters for this command are the same as for command 13, and the same rules apply.

If the target buffer is included in the list of source buffers then it will be skipped to prevent a reference loop.

Copying by reference means that the blocks in the target buffer will point to the same memory as the blocks in the source buffers.  Operations to modify data blocks in the source buffers will therefore also modify those blocks in the target buffer.  Clearing the source buffers will not clear the target buffer - it will still point to the original data blocks.  Data blocks are only freed from memory when no buffers are left with any references to them.

Buffers that get consolidated become new blocks, so will lose their links to the original blocks, thus after a "consolidate" operation modifications to the original blocks will no longer be reflected in the consolidated buffer.

This command is useful to construct a single buffer from multiple sources without the copy overhead, which can be costly.  For example, this can be useful for constructing a bitmap from multiple constituent parts before consolidating it into a single block.  In such an example, using command 13 instead would first make a copy of the contents of the source buffers, and then consolidate them into a single block.  Using this command does not make that first copy, and so would be faster.

This command is also useful for creating multiple buffers that all point to the same data.

## Command 26: Copy blocks from multiple buffers and consolidate

`VDU 23, 0, &A0, targetBufferId; 26, sourceBufferId1; sourceBufferId2; ...; 65535;`

This command is similar to performing a "copy" operation followed by a "consolidate" operation, and thus has similar behaviour to [command 13](#command-13) and/or command 25.  The parameters for this command are the same as for command 25.  As with command 25, you cannot include the target buffer in the list of source buffers.  If you do, then it will be skipped.

This command will replace the target buffer with a new buffer that contains a single block that is the result of consolidating the blocks from the source buffers.  If the target buffer already contains a single block of the same size as the source buffers then it will re-use the memory, and so will be faster than performing a separate "copy by reference" and "consolidate" operation.

It is useful for constructing a single buffer from multiple sources, such as for constructing a bitmap from multiple constituent parts.

## Commands 32 and 33: Create or manipulate a 2D or 3D affine transformation matrix

`VDU 23, 0, &A0, bufferId; 32, operation, [<format>, <arguments...>]`<br>
`VDU 23, 0, &A0, bufferId; 33, operation, [<format>, <arguments...>]`

As of the time of writing, this command is experimental and subject to change.  It features in the VDP 2.9.0 release, but to use this command you need to enable the feature by setting the affine transforms test flag.  This is done using the command `VDU 23, 0, &F8, 1; 1;`.  The exact operations and arguments supported by this command may change in the future.  Command 33 was added in the VDP 2.10.0 release.

The purpose of these commands is to create or manipulate an affine transform matrix stored in a buffer.  Affine transforms are used to manipulate 2D or 3D points, and can be used to perform operations such as translation, rotation, scaling, and shearing.  Command 32 creates or manipulates a 2D affine transform matrix, and command 33 creates or manipulates a 3D affine transform matrix.  Affine transforms are used to manipulate 2D or 3D points, and can be used to perform operations such as translation, rotation, scaling, and shearing.

To use this API you do not need to understand the mathematics of affine transformations.  The API is designed to be simple to use, and to allow for complex transformations to be built up from simple operations.

Technically, a 2D affine transformation matrix is a 3x3 matrix that can be used to perform transformations on 2D points, and can be applied to drawing operations.  A 3D affine transform uses a 4x4 matrix to account for the extra dimension.  The matrix is stored in row-major order, and is stored as 32-bit single-precision IEEE-754 floating point values.  The matrix is stored in a single block in the buffer.  The drawing system may store a second block in the buffer to cache an inverse of the matrix, but you should not rely on that being there.

A challenge with this API is that inherently neither the VDU command system nor the eZ80 CPU directly support floating-point arithmetic.  The API therefore supports sending numbers across in a variety of different formats to help facilitate this, and will convert the values sent to floating-point values as required.  The API supports sending fixed-point values, 16-bit and 32-bit integers, and 16-bit and 32-bit floating-point values.  Values must just be sent across as bytes in the VDU command stream in little-endian order, much like any other value byte must be send.

Several different operations are supported by this command.  When an operation is performed, the result is stored back in the buffer, replacing any existing data that may have been there.  Most operations will combine their result with the existing matrix in the buffer.  This means that you can combine for instance rotation and scaling into one transform matrix.  The following operations are supported:

| Operation | Arguments | Description |
| --------- | -------------- | ----------- |
| 0 | 0 | Set an "identity matrix" (effectively a "reset" operation) |
| 1 | 0 | Invert the matrix (this will only succeed if the buffer already contains a valid transform matrix) |
| 2 | 1 (3 for 3D) | Rotate anticlockwise by angle in degrees |
| 3 | 1 (3 for 3D) | Rotate anticlockwise by angle in radians |
| 4 | 1 | Multiply all values in the first 8 matrix positions by an amount |
| 5 | 2 (3 for 3D) | Scale X and Y (and Z for 3D) by given scaling factors |
| 6 | 2 (3 for 3D) | Translate X and Y (and Z for 3D) by a number of pixels |
| 7 | 2 (3 for 3D) | Translate X and Y by using currently selected graphics coordinate system units, (Z transform in pixels for 3D version) |
| 8 | 2 (3 for 3D) | Shear X and Y by given amounts |
| 9 | 2 (3 for 3D) | Skew X and Y by angle in degrees |
| 10 | 2 (3 for 3D) | Skew X and Y by angle in radians |
| 11 | 6 (12 for 3D) | Transform (combine with another transform matrix - requires 6 values to be sent, or 12 values for 3D - the final matrix row will be set to `0, 0, 1`, or `0, 0, 0, 1` for a 3D matrix) |
| 12 | `bitmapId;`, 2 | Translate X and Y by a proportion of the size (width/height) of a given bitmap.  This operation requires a 16-bit bitmap ID to immediately follow the operation byte, before the format byte and X, Y values.  NB the 3D version of this command will also only perform an X, Y translation, and thus also only accepts 2 arguments |

Repeatedly calling this command with different operations will build up a transform matrix in the buffer.  It should be noted that if the buffer is not cleared out before starting to build up a new matrix (or set to an identity matrix) then the results may not be as expected.

Every operation that requires arguments to be provided must then send a byte to indicate the format of the arguments that follow.  The format byte is as follows:

| Bit value | Description |
| --- | ----------- |
| 0-&1F | Shift (used for fixed-point values, ignored for floating-point) |
| &20 | Unused, must be zero (Reserved for future use) |
| &40 | When set, data is provided in fixed-point format, otherwise it is IEEE-754 floating point |
| &80 | When set, data is presented as 16-bit values, otherwise they are 32-bit values |

The fixed-point format supported by this command essentially means that values sent will be interpreted as a binary number with a "binary point".  The binary point starts out to the right of the right-most bit of the value (the least significant bit), meaning that when a shift value of zero is used the number being sent is an integer.  A shift of 1 will move the binary point one place to the left, effectively dividing the number by 2.  A shift of 2 will divide the number sent by 4, and so on.

The shift value is a 5-bit value, and its use varies depending on whether a 16-bit or a 32-bit value is being sent (i.e. if bit 7 has been set).  When a 32-bit value is sent (bit 7 is clear), the shift is interpreted as a 5-bit unsigned integer, i.e. it has the range of 0-31.  For 16-bit values, the 5-bit shift is a signed integer, giving a range of -16 to +15.  This allows for a negative shift to be applied to 16-bit values, meaning the number sent will be multiplied by 2 when a shift of -1 is given, by 4 for a shift of -2, and so on.

As can be seen, the API also supports sending IEEE-754 floating point values.  These can be sent either as single-precision values (using 32-bits), or as half-precision values (in 16-bits).  The "shift" bits in the format byte will be ignored when sending floating-point values.  A format value of `0` therefore indicates that values will be sent as 32-bit single-precision IEEE-754 floating point values, and a format value of `&80` indicates values will be sent as 16-bit half-precision IEEE-754 values.

In all cases data should be sent in little-endian byte order.

### Advanced operations

Similar to the Adjust command, it is possible to perform some advanced operations with this command by setting some of the upper bits of the operation byte.  The following bits are defined:

| Bit value | Description |
| --- | ----------- |
| &10 | Use advanced offsets (when using buffer-fetched values) |
| &20 | Fetch values from a buffer (and offset), rather than the command stream |
| &40 | Separate arguments will have individual format bytes |

The most important of these is the "fetch values from a buffer" bit.  When this bit is set, a format byte is still read from the VDU command stream, but then the next two bytes in the stream are interpreted as a buffer ID, which should then be followed by an offset (2 further bytes, unless the "use advanced offsets" bit is set).  The value to be used in the operation will then be fetched from the buffer at the given offset, and interpreted using the format described in the format byte.  Using this allows for transform matrices to be built up over time in multiple buffers, and then combined into a single buffer.  This can be useful for building up complex transformations in a modular way.

When this flag is set, all arguments are fetched from the given buffer, at the given offset.  If the operation requires multiple bytes they will be read consecutively from the buffer.  The format byte is still used to interpret the values fetched from the buffer.

When the "Separate arguments have individual format bits" flag is set then each argument will be prefaced with its own format byte, rather than a single byte being used to dictate the format of all arguments.  This can be useful when sending multiple arguments of different types.  This can be combined with the other flags.


## Command 34: Create or combine a matrix of arbitrary dimensions

`VDU 23, 0, &A0, bufferId; 34, operation, rows, columns, [<arguments>]`

As of the time of writing, this command is experimental and subject to change.  It features in the VDP 2.10.0 release, but to use this command you need to enable the feature by setting the affine transforms test flag.  This is done using the command `VDU 23, 0, &F8, 1; 1;`.  The exact operations and arguments supported by this command may change in the future.

This command expands on commands 32 and 33 to allow for the creation of a matrix of arbitrary dimensions.  The matrix is stored in a buffer, and is stored in row-major order.  The matrix is stored as 32-bit single-precision IEEE-754 floating point values.  The matrix is stored in a single block in the buffer.

This command works in a similar manner to commands 32 and 33, and where applicable supports the same floating point data formats.  The operations supported by this command are as follows:

| Operation | Arguments | Description |
| --------- | -------------- | ----------- |
| 0 | `format, <arguments...>` | Set the matrix to the values provided |
| 1 | `sourceBufferId; row, column, format, <arguments...>` | Copy source matrix and set an individual value |
| 2 | `format, <value>` | Create a matrix with all entries filled with the given value |
| 3 | `format, <value>` | Create a matrix filled with zeros and set the diagonal to the given value |
| 4 | `sourceBufferId1; sourceBufferId2;` | Add two matrices together |
| 5 | `sourceBufferId1; sourceBufferId2;` | Subtract matrices (target = source1 - source2) |
| 6 | `sourceBufferId1; sourceBufferId2;` | Multiply two matrices together |
| 7 | `sourceBufferId; format, <value>` | Multiply matrix by a scalar value |
| 8 | `sourceBufferId; row, column` | Extract a sub-matrix from a source matrix at given row and column. Target matrix will be filled from top-left, truncating or padding with zeros as necessary |
| 9 | `sourceBufferId; row` | Create a new copy of the source matrix, inserting a row at given offset |
| 10 | `sourceBufferId; column` | Create a new copy of the source matrix, inserting a column at given offset |
| 11 | `sourceBufferId; row` | Create a new copy of the source matrix, removing a row at given offset |
| 12 | `sourceBufferId; column` | Create a new copy of the source matrix, removing a column at given offset |

The target matrix will always be created with the number of rows and columns given in the command.  If the source matrix is not the same size as the target matrix then the source matrix will be truncated or padded with zeros as necessary.

When multiplying two different matrixes together, usually it is required that the second matrix will have the same number of rows as there are columns in the first matrix.  This command however will allow different matrixes of different dimensions to be multiplied together.  It does this by making the matrixes square and padding with zeros where necessary, where the square size is the maximum of the dimensions of the sources and target matrixes.


### Advanced operations

Similar to commands 32 and 33, it is possible to perform some advanced operations with this command by setting some of the upper bits of the operation byte.  The following bits are defined:

| Bit value | Description |
| --- | ----------- |
| &10 | Use advanced offsets (when using buffer-fetched values) |
| &20 | Fetch values from a buffer (and offset), rather than the command stream |


## Command 40: Create a transformed bitmap

`VDU 23, 0, &A0, bufferId; 40, options, transformBufferId; sourceBitmapId; [width; height;]`

As of the time of writing, this command is experimental and subject to change.  It features in the VDP 2.10.0 release, but to use this command you need to enable the feature by setting the affine transforms test flag.  This is done using the command `VDU 23, 0, &F8, 1; 1;`.  The exact options and arguments supported by this command may change in the future.

This command applies an affine transformation to a bitmap, creating a new RGBA2222 format bitmap.  It will replace the target buffer with the new bitmap, and creates a corresponding bitmap.

The `options` parameter is an 8-bit value that can have bits set to modify the behaviour of the operation.  The following bits are defined, and can be combined together:

| Bit value | Arguments | Description |
| --- | --- | ----------- |
| 1 |  | Target bitmap should be resized.  When _not_ set, target will be same dimensions as the original bitmap. |
| 2 | `width; height;`| Target bitmap will be resized to explicitly given dimensions
| 4 |  | Automatically translate target bitmap position.  When set the calculated transformed minimum x,y coordinates will be placed at the top left of the target |

Usually the target bitmap will be the same size as the source bitmap, but it is possible to resize the target bitmap to a different size.  When no explicit size is given, but the "resize" bit has been set, then the target bitmap size will depend on the transformation being applied.  This could mean, for example, that applying a progressive series of rotations to a bitmap can result in several different sizes of target bitmap.


## Command 41: Apply a transform matrix to data in a buffer

`VDU 23, 0, &A0, bufferId; 41, options, format, transformBufferId; sourceBufferId; [<arguments>]`

As of the time of writing, this command is experimental and subject to change.  It features in the VDP 2.10.0 release, but to use this command you need to enable the feature by setting the affine transforms test flag.  This is done using the command `VDU 23, 0, &F8, 1; 1;`.  The exact options and arguments supported by this command may change in the future.

This command will copy the source buffer and transform sets of values held within it using the given format and transform matrix.  The transformed values will be stored in the target buffer, replacing any existing data that may have been there.  The options byte, and optional arguments (depending on which bits of the options byte are set), control how the transformation is applied.

The `options` parameter is an 8-bit value that can have bits set to modify the behaviour of this command, including which arguments should be sent.  The following bits are defined, and can be combined together:

| Bit value | Arguments | Description |
| --- | --- | ----------- |
| &01 | `size` | Explicit data set size (otherwise data size will be one less than the number of rows in the transform matrix) |
| &02 | `offset;` | Has an offset into the buffer for where to find the first data set (NB this may be an "advanced offset" if that option is set) |
| &04 | `stride;` | Has an explicit stride (in bytes) between value sets |
| &08 | `limit;` | Has an explicit limit to the number of data items to be transformed |
| &10 |  | "Advanced offsets" should be used |
| &20 |  | Optional argument values should be fetched from buffers (and thus be a bufferId and offset in the command stream) |
| &40 |  | Data transforms should be applied on a block-by-block basis on data in the source buffer |

The `format` argument indicates the format of data in the source buffer, using the same format as for the affine transform commands.  This means that values to be transformed will be interpretted either as fixed-point or floating-point values in either 16 or 32 bits.  Using fixed-point format with a shift value of zero will interpret the values as integers (e.g. a format of `&C0` means source data is in 16-bit integers).  Data in the target buffer will be stored in the same format as the source buffer.

This command works with sets of data.  Typically if you are using this command to transform 2D points the size of the data set will be two, to indicate that you're transforming two values, the X and Y coordinates.  These sets of data values must appear contiguously in the source buffer.  The `stride` dictates how far apart (in bytes) the start of each set of data lies in the source buffer.  The `limit` parameter can be used to limit the number of data sets that are transformed.

As an example, this command can be used to transform sets of coordinates in a buffer that contains a series of PLOT commands.  A complete PLOT VDU command is a sequence of 6 bytes, where the first byte is `25` for PLOT, the second byte is the PLOT operation code, and then there are two 16-bit integer values for the X and Y coordinates.  The `offset` therefore would be set to `2`, and the `stride` would be set to `6`.  Our `format` needs to indicate we are using 16-bit fixed-point values with no shift, so that equates to `&C0`.  As we wish to set an explicit (start) offset and stride we need an `options` value of `6` (which is 2 + 4).  A command to use a 2D transform matrix stored in buffer `10` on a command sequence the data stored in buffer 20, writing the transformed data to buffer 30, would look like this:

```
VDU 23, 0, &A0, 30; 41, 6, &C0, 10; 20; 2; 6;
```

This example makes two significant assumptions.  The first, as mentioned above, is that the source buffer only contains a series of PLOT commands.  The second assumption is that we are using a 2D affine transform matrix created by command 32, which will have created a 3x3 matrix, and therefore an explicit `size` argument is not needed - it will be automatically derived as a data set size of 2, i.e. X and Y coordinates.

Once this example command has been executed, buffer 30 would contain the same sequence of PLOT commands as buffer 20, but with the X and Y coordinates transformed by the matrix stored in buffer 10.  Buffer 30 could then be called, and a transformed version of the PLOT commands would be drawn on the screen.

It should be noted that since it is only the coordinates that are transformed, the nature of the PLOT commands themselves will not be changed.  If the transform matrix was created with only "translate" or "scale" operations then the effect will work as expected for all PLOT commands (except for bitmap plots, which would not be drawn scaled as only the target coordinates woulld have changed), but if the transform included "rotate", "shear" or "skew" then results may differ.  PLOT commands that only draw lines, or fill triangles, will draw properly transformed versions of those shapes.  The effect on some other PLOT commands, such as those to fill a rectangle, or plot a circle/arc/sector will differ, as it is just the coordinates that are being transformed.  A rectangle may be drawn with a different size, but its sides will still be drawn aligned to the X and Y axis, and a circle will still be round.


## Command 48: Read a VDP variable into a buffer {#command-48}

`VDU 23, 0, &A0, bufferId; 48, options, offset; variableId; [default[;]]`

This command will copy the current value read from a [VDP variable](VDP-Variables.md) with the given `variableId;` into a buffer at the given offset.  Support for this command was added in VDP 2.12.0, and extended in VDP 2.15.0.

The `options` argument is an 8-bit value that can have bits set to modify the behaviour of this command.  The following bits are defined:

| Bit value | Description |
| --- | ----------- |
| &01 | Read value as big-endian (added in VDP 2.15.0) |
| &10 | Use advanced offsets |
| &40 | Use provided default value if no variable of the given ID is set |
| &80 | Use 16-bit values |

The value size for this command will, by default, be a single byte.  VDP Variables are however stored as 16-bit values, and so bit `&80` in the `options` byte can be set to indicate that the value should be read as a 16-bit value.  Such values are stored in little-endian order.

If you wish to read only the upper byte of a 16-bit value then you can set bit `&01` in the `options` byte.  This will swap the order of the bytes in the read value around, so therefore if you are reading an 8-bit value it will return the upper byte from the variable's 16-bit value.  This can be useful for reading a VDP variable that stores two 8-bit values in a single 16-bit variable, such as the keyboard event [keycode variable](./VDP-Variables.md#key-event).

If the variable does not exist, then the buffer will not be changed unless the `&40` bit has been set in the options byte, and a default value is provided.  The size of the default value sent must match the size of the value being read from the variable (as determined by bit `&80`).

If the `bufferId` does not exist, or the offset is out of bounds, then the command will fail.


## Command 64: Compress a buffer

`VDU 23, 0, &A0, targetBufferId; 64, sourceBufferId;`

This command will compress the contents of a buffer, replacing the target buffer with the compressed data.  Unless the target buffer is the same as the source, the source buffer will be left unchanged.

## Command 65: Decompress a buffer

`VDU 23, 0, &A0, targetBufferId; 65, sourceBufferId;`

This command will decompress the contents of a buffer, replacing the target buffer with the decompressed data.  Unless the target buffer is the same as the source, the source buffer will be left unchanged.

The source buffer must contain a complete set of compressed data, but need not be in a single block.  The decompressed data will be stored in a single block in the target buffer.

Using this command, data can be sent from MOS in a compressed form, and then decompressed on the VDP.  This can be useful for sending large amounts of data over to the VDP, as it can reduce the amount of data that needs to be sent.

The compression algorithm supported by this command and the corresponding "compress" command is "TurboVega-style" compression.  Source code for the compression and decompression routines and tools to use them on other systems can be found in the [TurboVega agon_compression repository](https://github.com/TurboVega/agon_compression).


## Command 72: Expand a bitmap

`VDU 23, 0, &A0, bufferId; 72, options, sourceBufferId; [width;] <mappingDataBufferId; | mapping-data...>`

This command will expend a bitmap stored in the source buffer indicated by `sourceBufferId` that uses an arbitrary number of bits per pixel into a new buffer (indicated by `bufferId`) that uses 8-bits per pixel.

The primary intent of this command is to allow the VDP to support other formats of bitmap than the natively supported formats.  The bitmap should be mapped to valid RGBA2222 colour values, allowing the destination buffer to then be set as a bitmap in RGBA2222 format.  It should be noted that the destination buffer is not automatically marked as being a bitmap.

It should also be noted that this command could be used for purposes other than expanding bitmaps, although the language used to describe the function of this command is based around bitmaps.  This is left to the user to explore.

The `options` parameter is an 8-bit value that can have bits set to modify the behaviour of the operation.  The following bits are defined:

| Bits | Description |
| ---- | ----------- |
| 0-2  | Number of bits per pixel in the source bitmap |
| 3    | When set, the source bitmap is aligns to the next byte at a given width (in pixels) |
| 4    | When set, mapping data is in a buffer |
| 5-7  | Reserved for future use (set to zero) |

The number of bits per pixel in the source bitmap is specified by the bottom 3 bits of the `options` parameter.  This can be any value from 1 to 8 where a 0 is interpreted as 8.

It is assumed that pixels are stored in the source buffer in a continuous manner, one byte at a time, starting from the top-most bits of the first byte.

If bit 3 has been set of the options byte, then following the `sourceBufferId` should be a 16-bit `width` parameter.  This value is used as a pixel count, after which the system will align to the next byte.  This is useful when dealing with bitmaps that have widths that do not align to a byte boundary.

The various different values that pixels will be mapped to should immediately follow in the command stream, with the number of bits per pixel given dictating how many mapping value bytes are sent (so 1 bits per pixel will have 2 values, 2 bits per pixel will have 4 values, and so on).  If bit 4 has been set of the options byte, then following the `width` parameter should be a 16-bit `mappingDataBufferId` parameter.  This buffer should contain the mapping data which will be used instead of values sent as part of the command stream..

When a buffer is used for mapping data, that buffer must exist, and must contain a single block of at least the number of values required for the given number of bits per pixel.


## Command 80: Set a buffer to be used for a callback {#command-80}

`VDU 23, 0, &A0, bufferId; 80, eventType;`

Sets a buffer to be used as a callback when a certain event is triggered in the VDP.

Support for callbacks was added in VDP 2.12.0, and expanded in VDP 2.15.0.

The `eventType;` argument is a 16-bit value that specifies which event type a callback buffer is to be used for.  The following types are supported:

| Type | Event | Caused by |
| ---- | ----------- | ----------------- |
| 0 | VSYNC | A new video frame has been displayed |
| 1 | Mode change | The screen mode has been changed |
| 2 | Keyboard update * | A key has been pressed or released |
| 3 | Mouse update * | A mouse change has been detected (movement or button press) |
| 4 | Palette change * | A palette entry has been changed using [`VDU 19`](./VDU-Commands.md#vdu-19) |
| 5 | Read pixel * | A pixel has been read from the screen using [`VDU 23, 0, &84, x; y;`](./System-Commands.md#vdu-23-0-84) |
| &100-&17F | VDP Protocol - about to send packet * | Indicates that a VDP Protocol packet is about to be sent to MOS |
| &180-&1FF | VDP Protocol - packet sent * | A VDP Protocol packet has just been sent to MOS |

\* Support for keyboard, mouse, palette change, and VDP protocol events were added in VDP 2.15.0.

When an event occurs that has one or more buffers set as callbacks, those buffers will be executed in the order they were added as callbacks for that event type.  Adding the same buffer as a callback for the same event type multiple times will have no effect; it will only be called once when the event happens.

A buffer will remain as a callback until it is removed or the buffer deleted.  If you wish to have a "one-shot" callback then you should remove the buffer from the callback after it has been triggered.

Additional callback event types will be added in later versions of the VDP.  These are likely to include callbacks for audio system events.

Callbacks registered for VSYNC events will be cleared after any mode change.  To ensure that your VSYNC callbacks remain active after a mode change you will need to re-register them after the mode change has occurred.  You can do this with a buffer registered to respond to mode change events.

Keyboard, mouse, and read pixel callbacks are all triggered when the VDP processes those events, before it attempts to send data to MOS.  This allows the callback to potentially modify the data before it is sent.

### Palette change event callbacks

If you are using a palette change event callback in order to derive other palette entries you should note that calling [`VDU 19`](./VDU-Commands.md#vdu-19) from within your callback routine will also trigger your callback, creating an infinite loop.  There are several ways this can be avoided.  The simplest way is to set your derived palette using their corresponding [palette entry variables](./VDP-Variables.md#palette-entries).  This will not trigger the callback, or update the ["last colour" VDP variables](./VDP-Variables.md#last-colour).  There are other methods that can be used which would allow the use of `VDU 19` without causing an infinite loop; this is left as an exercise for the reader.

### VDP Protocol event callbacks

Events in the range &100-&1FF relate to the [VDP Protocol](./System-Commands.md#vdp-serial-protocol).  The lower half of this range (&100-&17F) will be triggered before a VDP protocol packet is sent to MOS, and the upper half &180-&1FF are triggered after a VDP protocol packet has been received from MOS.  The bottom 7 bits of the event type correspond to the VDP protocol packet type.

These callbacks can be used to modify the state of VDP variables before and after a VDP protocol packet is sent.

NB these VDP protocol event callbacks will occur no matter what the [current output stream is set to](#command-4).  Even if the output stream has been disabled (set to `-1`), the VDP protocol event callbacks will still be triggered when a VDP protocol packet is sent.  It should be noted though that as the command to [set the output stream](#command-4) only applies within the context of a buffer call this command cannot be used by callbacks to prevent that VDP protocol packets from being sent as the output stream will be restored when the callback buffer call ends.

One can however suppress sending a VDP protocol packet from within a "before send" VDP protocol event callback by setting VDP variable &103 to any value, for example:

`VDU 23, 0, &F8, &103; 0;`

If a packet is suppressed in this way the corresponding "packet sent" event callback will not be triggered.

## Command 81: Remove buffer from a callback

`VDU 23, 0, &A0, bufferId; 81, type;`

Removes a buffer from being used as a callback for a certain event type.

Support for callbacks was added in VDP 2.12.0.

The types are as described in [command 80](#command-80), with the addition that sending a type of `65535` will remove the buffer from all callback types.

Calling this command with a bufferId value of `65535` will clear all callbacks for the given type.  Therefore if you wish to clear all callbacks for all types then you can call this command with a bufferId of `65535` and a type of `65535`.


## Command 128: Debug info command

`VDU 23, 0, &A0, bufferId; 128`

This command is a debugging command that will print out info on a buffer to the USB serial console.  This is useful for debugging purposes, and can be used to check the contents of a buffer after a series of operations have been performed on it.

The info printed to the console will tell you how many blocks/streams are stored against the `bufferId`.  If the buffer contains a transform matrix, then the matrix will be printed out in a human-readable format.  For other buffers the whole of the first block/stream will be output to the console in hexadecimal format.  NB the whole block/stream will be output, so be aware that if the buffer is large then a lot of data will be sent to the console.

Before VDP 2.10.0 this command would only work if you were using a VDP compiled with the `DEBUG` flag set.  As of VDP 2.10.0 this command will now work without the flag being set.


## Examples

What follows are some examples of how the VDP Buffered Commands API can be used to perform various tasks.

### Loading a sample

Sound sample files can be large, and so it is not practical to send them over to the VDP in a single packet.  Even with optimised machine code, it could take several seconds to send a single sample over to the VDP.  This would block the screen, and make it impossible to show progress to the user.  Using the VDP Buffered Commands API we can send a sample over to the VDP in multiple packets.

The following example will load a sound sample from a file called `sound.bin` and send it over to the VDP.  Lines 10-50 prepare things, opening up the file and getting its length.  Line 70 clears out buffer 42 so it is ready to store the sample.  The loop from lines 90 to 170 sends the sample one block at a time, adding the sample data to buffer 42.  Finally, lines 200-220 creates the sample, sets channel 1 to use it, and then plays it.

```
 10 blockSize% = 1000
 20 infile% = OPENIN "sound.bin"
 30 length% = EXT#infile%
 40 PRINT "Sound sample length: "; length%; "bytes"
 50 remaining% = length%
 60 REM Load sample data into buffer 42
 70 VDU 23, 0, &A0, 42; 2       : REM Clear out buffer 42
 80 PRINT "Loading sample";
 90 REPEAT
100   IF remaining% < blockSize% THEN blockSize% = remaining%
110   remaining% = remaining% - blockSize%
120   PRINT ".";       : REM Show progress
130   VDU 23, 0, &A0, 42; 0, blockSize%; : REM Send next blockSize% bytes to buffer 42
140   FOR i% = 1 TO blockSize%
150     VDU BGET#infile%
160   NEXT
170 UNTIL remaining% = 0
180 CLOSE #infile%
190 REM Set buffer 42 to be an 8-bit unsigned sample
200 VDU 23, 0, &85, 1, 5, 2, 42; 1     : REM Channel is ignored in this command
210 VDU 23, 0, &85, 1, 4, 8, 42;       : REM Set sample for channel 1 to buffer 42
220 VDU 23, 0, &85, 1, 0, 100, 750; length% DIV 16;  : REM Play sample on channel 1
```

Please note that the BASIC code here is not fast, owing to the fact that it has to read the sample file one byte at a time.  This is because BBC BASIC does not provide a way to read a chunk of a file at once.  This is not a limitation of the VDP Buffered Commands API, but rather of BBC BASIC.

This can be optimised by writing a small machine code routine to read a chunk of a file at once, and then calling that from BASIC.  This is left as an exercise for the reader.

Whilst this example illustrates loading a sample, it is easily adaptable to loading in a bitmap.

### Repeating a command

This example will print out "Hello " 20 times.

This is admittedly a contrived example, as there is an obvious way to achieve what this code does in plain BASIC, but it is intended to illustrate the API.  The technique used here can be fairly easily adapted to more complex scenarios.

This example uses three buffers.  The first buffer is used to print out a string.  The second buffer is used to store a value that will be used to control how many times the string printing buffer is called.  The third buffer is used to call the string printing buffer the required number of times, and is gradually built up.

```
 10 REM Clear the buffers we're going to use (1-3)
 20 VDU 23, 0, &A0, 1; 2       : REM Clear out buffer 1
 30 VDU 23, 0, &A0, 2; 2       : REM Clear out buffer 2
 40 VDU 23, 0, &A0, 3; 2       : REM Clear out buffer 3
 50 VDU 23, 0, &A0, 1; 0, 6;   : REM Send the next 6 bytes to buffer 1
 60 PRINT "Hello ";            : REM The print will be captured into buffer 1
 70 REM Create a writable buffer with ID 2, 1 byte long for our iteration counter
 80 VDU 23, 0, &A0, 2; 3, 1;
 90 REM set our iteration counter to 20 - adjust (5), set value (2), offset 0, value 20
100 VDU 23, 0, &A0, 2; 5, 2, 0; 20
110 REM gradually build up our command buffer in buffer 3
120 VDU 23, 0, &A0, 3; 0, 6;   : REM 6 bytes for the following "call" command
130 VDU 23, 0, &A0, 1; 1       : REM Call buffer 1 to print "Hello "
140 VDU 23, 0, &A0, 3; 0, 10;  : REM 10 bytes for the following "adjust" command
150 REM Decrement the iteration counter in buffer 2
160 VDU 23, 0, &A0, 2; 5, 3, 0; -1
170 VDU 23, 0, &A0, 3; 0, 11;  : REM 11 bytes for the following "conditional call" command
180 REM If the iteration counter is not zero, then call buffer 3 again
190 VDU 23, 0, &A0, 3; 6, 0, 2; 0;
200 REM That's all the commands for buffer 3
210 REM Now call buffer 3 to execute those commands
220 VDU 23, 0, &A0, 3; 1
```

It should be noted that after this code has been run the iteration counter in buffer ID 2 will have been reduced to zero.  Calling buffer 3 again at that point will result in the counter looping around to 255 on its first decrement, and then counting down from there, so you will see the loop run 256 times.  To avoid this, the iteration counter in buffer 2 should be reset to the desired value before calling buffer 3 again.

Another thing to note is that if there were any additional commands added to buffer 3 beyond the final conditional call then it is likely that the VDP would crash, which is obviously not ideal.  This would happen because the call stack depth (i.e. number of "calls within a call") will have become too deep, and the command interpreter inside the VDP will have run out of memory.  This code works as-is because the conditional call is the last command in buffer 3 the VDP uses a method called "tail call optimisation" to avoid having to return to the caller.  The call is automatically turned into a "jump".  This is a technique that is used in many programming languages, and is a useful technique to be aware of.

A safer way to write this code would be to use a conditional jump (command 8) rather than a conditional call.  This would avoid the call stack depth issue, and allow additional commands to be placed in buffer 3 after that jump.

As a very simple example, you can imagine replacing buffer 1 with a buffer that draws something to the screen.  Those drawing calls could use relative positioning, allowing for repeated patterns to be drawn.  It could indeed do anything.  The point here is just to illustrate the technique.


### A simplistic "reset all audio channels" example

Sometimes you will just want the ability to very quickly call a routine to perform a bulk action.  One potential example of this is to reset all audio channels to their default state (default waveform, and remove any envelopes that may have been applied).  The audio API provides a call to reset individual channels, but there is no call to reset them all.

```
 10 REM Clear the buffer we're going to use
 20 resetAllChannels% = 7
 30 VDU 23, 0, &A0, resetAllChannels%; 2       : REM Clear out buffer
 40 FOR channel = 0 TO 31
 50   VDU 23, 0, &A0, resetAllChannels%; 0, 5; : REM 5 bytes for the following "reset channel" command
 60   VDU 23, 0, &85, channel, 10
 70 NEXT
 80 REM Call the clear command
 90 VDU 23, 0, &A0, resetAllChannels%; 1
```

In this example we take a simplistic approach to building up a command that will reset all the audio channels.  The nature of the Audio API is that one can ask any channel to be reset, even if it has not been enabled, so we can just loop through all 32 potential channels.  An alternative approach could have been to disable all the channels and then enable a default number of channels.

Once a command buffer has been sent it will remain on the VDP until that buffer is cleared.  This means that the `VDU 23, 0, &A0, clearCommand%; 1` call can be made many times.  This can be useful if you wanted to reset all the audio channels at the start of a game loop, for instance.

It is possible to write a more sophisticated version of this example that would use a loop on the VDP, rather than relying on sending multiple "reset channel" commands.  That would require the use of a few more buffers.  The reality of this approach however is that it is significantly more complex to accomplish and require quite a lot more BASIC code.  Since there is a lot of available free memory on the VDP for storing commands, it is not necessary to be overly concerned about the number of commands sent, so often a simpler approach such as the one in this example is on balance the better option.


### Other ideas, techniques, and principles

The examples above are intended to illustrate some of the principles of how the VDP Buffered Commands API can be used.  They are not intended to be complete solutions or illustrations of what is possible.

#### Stack depth

It should be noted that the VDP does not have a very deep call stack, and so it is possible to run out of stack space if you have a large number of nested calls.  (At the time of writing, the depth limit appears to be in the region of 20 calls.  For those that don't understand what "stack depth" means, an example of this would be calling buffer 1, which in turn calls buffer 2, which in turn calls buffer 3 and so on, up to calling buffer 20.)  If the depth limit is exceeded, the VDP will crash, and you will need to press the reset button.

As noted above in the "repeat" example, the VDP will use a technique called "tail call optimisation" to help avoid/mitigate this issue.  This is where a call is automatically turned into a "jump" if it is the last command in a buffer.  This avoids the need to return to the caller, removing the need to start a new VDU command interpreter, and so avoids the call stack depth issue.

Often the stack depth issue can be avoided by using a "jump" command rather than a "call" command (as this also does not need to start a new VDU command interpreter).  A jump differs from a call in that it just changes the command sequence being executed, and does not keep track of where it was being called from.

The down-side of using a jump is that over-use of jumps can result in "spaghetti code", which can be difficult to follow.  It is therefore recommended to use jumps sparingly, and to use calls where possible.

#### Using many buffers

The API makes use of 16-bit buffer ID to allow a great deal of freedom.  Buffer IDs do not have to be used sequentially, so you are free to use whatever buffer IDs you like.  It is suggested that you can plan out different ranges of buffer IDs for your own uses.  For example, you could decide to use buffer IDs &100-&1FF for sprite management, &200-&2FF for sound management, &400-&4FF for data manipulation.  This is _entirely_ up to you.

Command buffers can be as short, or as long, as you like.  Often it will be easier to have many short buffers in order to allow for sophisticated behaviour.

The VDP also has significantly more free memory available for storing commands and data than the eZ80 does, so it is not really necessary to be overly concerned about the number of commands sent.  (Currently there is 4 megabytes of free memory available on the VDP for storing commands, sound samples, and bitmaps.  The memory attached to the VDP is actually an 8 megabyte chip, and a later version of the VDP software may allow for even more of that to be used.)

#### Self-modifying code

A technique that was fairly common in the era of 8-bit home computers was to use self-modifying code.  This is where a program would modify its own code in order to achieve some effect.  The VDP Buffered Commands API allows for this technique to be used via the "adjust buffer contents" command.  For example this could be used to adjust the coordinates that are part of a command sequence to draw a bitmap, allowing for a bitmap to be drawn at different locations on the screen.

#### Jump tables

There are a number of ways to implement jump tables using the VDP Buffered Commands API.

One such example would be to allocate a range of buffer IDs for use as jump table entries, and to use the "adjust buffer" command to change the lower byte of the buffer ID on a "jump" or "call" command (or a "conditional" version) to point to the buffer ID of the jump table entry.

An alternative way could be to use "jump with offset" command with an advanced offset, specifying a block within a buffer to jump to, and adjusting that block as needed.  This would allow for a jump table to be built up within a single buffer.

#### Using the output stream

Some VDU commands will send response packets back to MOS running on the eZ80.  These packets can be captured by using the "set output stream" command.  This can be used to capture the response packets from a command.

When you have a captured response packet, the contents of the buffer can be examined to determine what the response was.  Values can be extracted from the buffer using the "adjust buffer contents" command and used to modify other commands.

For example, you may wish to find out where the text cursor position is on screen and then use that information to work out whether the text cursor should be moved before printing new output.

Care needs to be taken when using "set output stream" to ensure that the sequence of commands you're using doesn't create more response packets than you are expecting.  It is usually best to use this command within a buffered command sequence, as that will ensure that the output stream is reset to its original value once the buffered command sequence has completed.

If you are using this command as part of a longer sequence it is recommended to use the "set output stream" command to reset the output stream to its original value (by using a buffer ID of 0) once you have captured the response packet you are interested in.

Please note that at present the number of commands that send response packets is currently very limited, and so this technique is not as useful as it could be.  This will likely change in the future.

Also to note here is that response packets will be written sequentially to the output stream.  There is no mechanism to control _where_ in the output stream a response packet is written.  This means that if you are capturing response packets, you will need to be careful to ensure that the response packets you are interested in are not interleaved with other response packets that you are not interested in.  Clearing and re-creating a buffer before capturing response packets is recommended.

#### Use your imagination!

As can be seen, by having command sequences that can adjust the contents of buffers, and conditionally call other command sequences, it is possible to build up quite sophisticated behaviour.  You effectively have a programmable computer within the VDP.

It is up to your imagination as to how exactly the API can be used.

