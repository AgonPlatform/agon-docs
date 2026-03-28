# What is the MOS API

The MOS API can be used by external applications to access MOS functionality.

Please note that this documentation uses assembler in the examples in a format that is compatible with the Zilog ZDS II assembler.  The assembler syntax used in BBC BASIC is similar, but not identical.  Additionally you will need to use the new ADL version of BBC BASIC to use the new eZ80 ADL mode and extended instruction set.

This documentation is not intended as a tutorial on eZ80 assembler, but as a reference for those who are already familiar with the eZ80 or Z80 instruction set and wish to use MOS APIs in their programs.

## Usage from Z80 assembler {#rst}

There are four RST instructions for accessing MOS functionality from Z80.

- `RST 00h`: Reset the eZ80
- `RST 08h`: Execute a MOS command
- `RST 10h`: Output a single character to the VDP
- `RST 18h`: Output a stream of characters to the VDP (MOS 1.03 or above)
- `RST 38h`: Outputs a crash report (MOS 2.3.0 or above)

In addition, if you are using the Zilog ZDS II assembler you may wish to include the file `mos_api.inc` in your project.  The latest Agon Platform (previously known as Console8) version can be found in the folder [src](https://github.com/AgonPlatform/agon-mos/tree/main/src) of project [agon-mos](https://github.com/AgonPlatform/agon-mos).  The original Quark versions of this file can be found in the folder [src](https://github.com/breakintoprogram/agon-./tree/main/src) of project [agon-mos](https://github.com/breakintoprogram/agon-mos).

NB:

- Using the `RST.LIS` opcode in an eZ80 assembler will ensure the MOS `RST` instructions are called regardless of the eZ80s current addressing mode
    - The Agon MOS RST handlers are written with the assumption that they have been called using `RST.LIS`
    - Programs written to run in Z80 mode, using only plain Z80 opcodes, would therefore need to set up their own RST handlers to call through to to MOS using `RST.LIS`
- This documentation generally uses the term `RST` in place of `RST.LIS` for simplicity
- In the `mos_api.inc` file you will find:
    - EQUs for all the MOS commands, data structures and [system state variables (sysvars)](#sysvars)
    - An incomplete list of VDP control variables.  For a full list, see the [VDP documentation](../VDP.md)
    - A complete list FatFS APIs, however it should be noted that many these are not implemented in MOS prior to MOS 3.0

Further information on the `RST` handlers provided by MOS are as follows:

### `RST 08h`: Execute a MOS command

Parameters:

- `A`: MOS command number to execute

NB:

- There is a macro in `mos_api.inc` with EQUs for all the MOS commands
- Other MOS-command dependant parameters may be required

Macro:

```
;
; Macro for calling the API
; Parameters:
; - `F`unction: One of the function numbers listed above
;
MOSCALL:	MACRO	function
			LD	A, function
			RST.LIS	08h
			ENDMACRO
```

Example:

```
; OSRDCH: Read a character in from the ESP32 keyboard handler
;
OSRDCH:	MOSCALL	mos_getkey
		OR	A
		JR	Z, OSRDCH		; Loop until key is pressed
		RET
```

### `RST 10h`: Output a single character to the VDP

Parameters:

- `A`: Character to output

Example:

```
; OSWRCH: Write a character out to the ESP32 VDU handler via the MOS
; A: Character to write
;
OSWRCH:	RST.LIS	10h			; This calls a RST in the eZ80 address space
		RET
```

### `RST 18h`: Output a stream of characters to the VDP (MOS 1.03 or above)

Parameters:

- `HL(U)`: Address of the data stream (16-bit for Z80 mode, 24-bit for ADL mode)
- `BC`: Length of stream (or 0 if the stream is delimited)
- `A`: Stream delimiter (if `BC`=0)

Returns:

- `A`: Last character displayed (length mode) OR Delimiter (delimiter mode)
- `BC`: 0
- `HL(U)`: Address of last character displayed + 1 (length mode) OR location of delimiter (delimiter mode)
- `E`: Value of `A` upon entry

Example:

```
; Write a stream of characters to the VDP
; HLU: Address of buffer containing data - if in 16-bit segment, U will be replaced by MB
;  BC: Number of characters to write out, or 0 if the data is delimited
;   A: End of data delimiter, i.e. 0 for C strings
;
		LD	HL, text		; Address of text
		LD	BC, 0			; Set to 0, so length ignored...
		LD	A, 0			; Use character in A as delimiter
		RST.LIS	18h			; This calls a RST in the eZ80 address space
		RET
;
text:	DB	"Hello World", 0
```

### `RST 38h`: Outputs a crash report (MOS 2.3.0 or above)

This command will output a crash report to the screen.  This report will show the current processor state, and the top of the stack.  This can be useful for debugging purposes.

This command works in conjunction with the fact that as of MOS 2.3.0, on initial startup memory will be reset to contain `0xFF` bytes in every location, which equates to a `RST 38h` instruction.  This means that for many system crashes execution will end up at the `RST 38h` instruction, and a crash report will be displayed on the screen.


## The MOS API

MOS API calls can be executed from a classic 64K Z80 segment or whilst the eZ80 is running in 24-bit ADL mode. For classic mode, 16 bit registers are passed as pointers to the MOS commands; these are automatically promoted to 24 bit by adding the MB register to bits 16-23 of the register. When running in ADL mode, a 24-bit register will be passed, but MB must be set to `0`.

Many, but not all, of the MOS API calls will return a [status code](#status-codes) in the `A` register.  This status code will indicate the success or failure of the operation.  If the operation was successful, the status code will be `0`.  If the operation failed, the status code will be non-zero, and will indicate the nature of the failure.  Some API calls, such as those for I2C communications or string comparisons, use different sets of status codes, which are documented in the API call's description.

The APIs available from MOS have changed over time, and some of the APIs described are only available in later versions of MOS.  If you attempt to call an API that is not available in the version of MOS you are using the API will return the status value `23` in the `A` register to indicate it is not supported.  Please note that MOS 2.1.0 and earlier, including Quark MOS 1.04, do support detecting unknown/unsupported API calls and will produce unexpected results if you attempt to call an API that is not supported.  This is a known issue with these versions of MOS, and it is recommended to upgrade to a later version of MOS if you are using these versions.

### Advice on file handling

As of MOS 3.0, all the MOS API calls that accept any kind of filepath string as a parameter, whether that is to a filename or a directory, will support the use of [system variables](./System-Variables.md) and [custom file paths](./System-Variables.md#path-variables) within the string.  These will automatically be handled in native MOS file handling API calls.  This allows for more flexible and powerful file handling in your applications.

Please note that the FatFS API calls (which named with an `ffs_` prefix) do _not_ support this behaviour, and will only work with fully resolved file paths.  There is an API to [resolve the path](#0x38-mos_resolvepath) which can be used to convert a path with system variables into a path suitable for use with the fatfs APIs.

In general, to read and/or write files files, it is recommended to use the MOS file APIs as these will automatically handle system variables and file paths.  MOS file APIs use a "file handle" to reference an open file, whereas the FatFS APIs expect a pointer to a `FIL` structure.  You can get a `FIL` structure for a MOS file handle by using the [`mos_getfil` API](#0x19-mos_getfil).  This will allow you to use the FatFS APIs directly if you need to, but in most cases it is recommended to use the MOS file APIs.  It is planned that future versions of MOS (beyond 3.0) will support using the MOS file APIs to open data streams other than files, such as the serial UART, I2C devices, and the VDP connection.  This will allow you to use the same APIs to read/write data across different all data streams.

### A note on "system variables"

Please note that MOS 3.0 [system variables](./System-Variables.md) are a distinct and different feature from [system state information (sysvars)](#sysvars).  Some older code and documentation may use the term "system variables" to refer to sysvars.

### APIs that use 32-bit values

As of MOS 3.0 the standard for APIs that either require or return a 32-bit value is to pass a pointer to the value in a register.  Care should be taken to ensure that the pointer is pointing to a valid 4-byte block of memory.  There are two older APIs that date back to MOS 1.03, namely [`mos_flseek`](#0x1c-mos_flseek) and [`ffs_flseek`](#0x84-ffs_flseek) that used a different approach and return a 32-bit value spread across two registers, with the lower 24-bits in one register and the upper byte in a separate register.  As this is not friendly to Z80 code, MOS 3.0 has added new equivalent APIs that use the new 32-bit pointer approach.  The old APIs are still available for backwards compatibility, but it is recommended to use the new APIs where possible.

### Core MOS APIs, and Modules

A future version of MOS may support the use of [modules](./Modules.md) to extend the functionality of the system.  This may include adding new APIs, commands, and functions.

When this happens, the concept of "Core MOS" will be introduced.  Core MOS will be the set of APIs, commands, and functions that are guaranteed to be available to all programs.  

The exact set of APIs and commands that Core MOS will be comprised of has yet to be determined, but for compatibility with existing programs they will definitely include all commands and APIs that were available in MOS 2.3 and earlier.  It is expected that many of the APIs added in MOS 3.0 will also be included in Core MOS, but this is not guaranteed.

Functionality provided by modules would only be available to programs that are "module safe" or "module compatible".  Guidance on how to ensure that your program is "module safe" or "module compatible" will be provided in the [MOS Modules](./Modules.md) documentation.


## MOS API calls

The following MOS commands are supported:

### `0x00`: mos_getkey

Read a keypress from the VDP

Parameters: None

Returns:

- `A`: The keycode of the character pressed

NB: This is a blocking function. This routine will wait and only return once a key is pressed.

### `0x01`: mos_load

Load a file from SD card

Parameters:

- `HL(U)`: Address of filename (zero terminated)
- `DE(U)`: Address at which to load
- `BC(U)`: Maximum allowed size (bytes)

Preserves: `HL(U)`, `DE(U)`, `BC(U)`

Returns:

- `A`: Status code
- `F`: Carry reset if no room for file, otherwise set

### `0x02`: mos_save

Save a file to SD card

Parameters:

- `HL(U)`: Address of filename (zero terminated)
- `DE(U)`: Address to save from
- `BC(U)`: Number of bytes to save

Preserves: `HL(U)`, `DE(U)`, `BC(U)`

Returns:

- `A`: Status code
- `F`: Carry set

### `0x03`: mos_cd

Change current directory on the SD card

Parameters:

- `HL(U)`: Address of directory path (zero terminated)

Preserves: `HL(U)`

Returns:

- `A`: Status code

### `0x04`: mos_dir

List SD card folder contents to screen.

This is a simple directory listing command that will list the contents of the current directory to the screen.  More advanced directory listing functionality for applications to use is available via the [FatFS commands API](#fatfs-commands).

Parameters:

- `HL(U)`: Address of directory path (zero terminated)

Preserves: `HL(U)`

Returns:

- `A`: Status code

### `0x05`: mos_del

Delete a file or folder from the SD card

Parameters:

- `HL(U)`: Address of filepath (zero terminated)

Preserves: `HL(U)`

Returns:

- `A`: Status code

### `0x06`: mos_ren

Rename a file on the SD card

Parameters:

- `HL(U)`: Address of source filepath string (zero terminated)
- `DE(U)`: Address of destination filepath string (zero terminated)

Preserves: `HL(U)`, `DE(U)`

Returns:

- `A`: Status code

As of MOS 3.0 the `mos_ren` API can support the use of wildcards in the source filepath leaf name.  This allows you to rename multiple files at once.  The destination filepath must point to a directory in this case.  Destination paths cannot include wildcards.

### `0x07`: mos_mkdir

Make a folder on the SD card

Parameters:

- `HL(U)`: Address of path (zero terminated)

Preserves: `HL(U)`

Returns:

- `A`: Status code

### `0x08`: mos_sysvars

Fetch a pointer to the [system state variables](#sysvars)

NB as this returns a pointer in `IXU`, and is therefore difficult to use from C code, as of MOS 3.0 an alternative way to access the sysvars address is available via a C function obtainable from the [`mos_getfunction`](#0x50-mos_getfunction) API.

Parameters: None

Returns:

- `IXU`: Pointer to the MOS SysVars area (this is always 24 bit)

### `0x09`: mos_editline

Invoke the line editor

Parameters:

- `HL(U)`: Address of the buffer
- `BC(U)`: Buffer length
- `E`: Flags to control editor behaviour

Preserves: `HL(U)`, `BC(U)`, `DE(U)`

Returns:

- `A`: Key that was used to exit the input loop (CR=13, ESC=27)

Editor behaviour flags are as follows:

| Bit | Description |
| --- | ----------- |
| 0   | When set, buffer will be cleared before use |
| 1   | When set, tab-completion for MOS commands and files is enabled * |
| 2   | When set, hotkeys are _disabled_ * |
| 3   | When set, input history will be _disabled_ * |
| 4-7 | Reserved for future use (for future compatibility, ensure these are set to zero) |

\* Support for editor control flags was added in MOS 2.2.0.  Prior to this the only documented values for `E` were 0 and 1 to indicate whether the buffer should be cleared.

### `0x0A`: mos_fopen

Get a file handle

Parameters:

- `HL(U)`: Address of filename (zero terminated)
- `C`: File open mode

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: File handle, or 0 if couldn't open

Please note that whilst this API has always been documented as requiring a zero-terminated string for the filename, this was not actually _strictly_ true.  It actually interpreted any control character (0-31) as a termination character.  MOS 3.0.0 changed this behaviour to be a strict zero-terminated string, but it was found that several keywords in BBC BASIC (`OPENIN`, `OPENOUT` and `OPENUP`) were failing to work because they were sending a carriage-return (13) as their termination character.  MOS 3.0.1 has been changed to support control-character termination again, thus fixing compatibility with BBC BASIC.  You are, however, strongly advised to use zero-terminated strings in your code, as relying on control character termination can cause other issues.

#### File open modes 

The mode is a number that indicates rules as to how the file will be opened.  Several different modes are available, and these values can be combined using a bitwise OR operation.  The values supported are inherited from FatFS, and are as follows:

| Mode | FatFS constant | Description |
| ---- | -------------- | ----------- |
| 0x01 | `FA_READ`	| Open file for reading |
| 0x02 | `FA_WRITE`	| Open file for writing.  Combine with `FA_READ` for read/write access |
| 0x00 | `FA_OPEN_EXISTING`	| Open file if it exists, fail if it doesn't |
| 0x04 | `FA_CREATE_NEW`	| Create a new file, fail if it already exists |
| 0x08 | `FA_CREATE_ALWAYS`	| Create a new file.  If the file already exists it will be truncated and overwritten |
| 0x10 | `FA_OPEN_ALWAYS`	| Open file if it exists, create it if it doesn't |
| 0x30 | `FA_OPEN_APPEND`	| Same as `FA_OPEN_ALWAYS`, except the read/write pointer will be set to the end of the file |

You may note that the "open existing" mode has a value of zero.  Setting either, or both, of the "create" options will override this.

NB: If you open the file using `mos_fopen`, you must close it using `mos_fclose`, not `ffs_api_fclose`

### `0x0B`: mos_fclose

Close a file handle

Parameters:

- `C`: File handle, or 0 to close all open files

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: Number of files still open

### `0x0C`: mos_fgetc

Get a character from an open file

Parameters:

- `C`: File handle

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: Character read
- `F`: C set if last character in file, otherwise NC (MOS 1.04 or greater)

### `0x0D`: mos_fputc

Write a character to an open file

Parameters:

- `C`: File handle
- `B`: Character to write

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

None

### `0x0E`: mos_feof

Check for end of file

Parameters:

- `C`: File handle

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: 1 if at end of file, otherwise 0

### `0x0F`: mos_getError

Translates a status code into a human-readable message.

Parameters:

- `E`: The status code
- `HL(U)`: Address of buffer to copy message into
- `BC(U)`: Size of buffer

Preserves: `DE(U)`, `HL(U)`, `BC(U)`

Returns:

None

### `0x10`: mos_oscli

Execute a MOS command

Parameters:

- `HL(U)`: Pointer the the MOS command string

Preserves: `HL(U)`

Returns:

- `A`: Status code

NB previously documentation for this command was incorrect, as it documented additional parameters in `DE(U)` and `BC(U)`.  These registers are not currently used.

### `0x11`: mos_copy

Copy a file on the SD card

Parameters:

- `HL(U)`: Address of source filepath string (zero terminated)
- `DE(U)`: Address of destination filepath string (zero terminated)

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: Status code

NB: Requires MOS 1.03 or greater

Please note that this API only supports copying files; it does not support copying directories.

As of MOS 3.0, the `mos_copy` API supports the use of wildcards in the source filepath leaf name.  This allows you to copy multiple files at once.  The destination filepath must point to a directory in this case.  Destination paths cannot include wildcards.

### `0x12`: mos_getrtc

Get a time string from the RTC (Requires MOS 1.03 or above)

Parameters:

- `HL(U)`: Pointer to a buffer to copy the string to (at least 32 bytes)

Preserves: `HL(U)`

Returns:

- `A`: Length of time string

### `0x13`: mos_setrtc

Set the RTC (Requires MOS 1.03 or above)

Parameters:

- `HL(U)`: Pointer to a 6-byte buffer with the time data in

```
+0: Year (offset from 1980, so 1989 is 9)
+1: Month (1 to 12)
+2: Day of Month (1 to 31)
+3: Hour (0 to 23)
+4: Minute (0 to 59)
+5: Second (0 to 59)
```

Preserves: `HL(U)`

Returns:

None

### `0x14`: mos_setintvector

Set an interrupt vector (Requires MOS 1.03 or above)

Parameters:

- `E`: Interrupt vector number to set
- `HLU`: Address of new interrupt vector (24-bit pointer)

Preserves: `HLU`, `DEU`

Returns:

- `HL(U)`: Address of the previous interrupt vector (24-bit pointer)

### `0x15`: mos_uopen

Open UART1 (Requires MOS 1.03 or above)

To handle the received interrupts, you will need to assign a handler to UART1's interrupt vector (0x1A).

NB as this API uses a pointer in `IX(U)`, and is therefore difficult to use from C code, as of MOS 3.0 th underlying function this API uses is available via a C function accessible using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Parameters:

- `IX(U)`: Pointer to a UART struct

```
+0: Baud rate (24-bit, little endian)
+3: Data bits (5, 6, 7 or 8)
+4: Stop bits (1 or 2)
+5: Parity bits (0: None, 1: Odd, 3: Even)
+6: Flow control (0: None, 1: Hardware)
+7: Enabled interrupts
    - Bit 0: Set to enable received data interrupt
    - Bit 1: Set to enable transmit data interrupt
    - Bit 2: Set to enable line status change interrupt
    - Bit 3: Set to enable modem status change interrupt
    - Bit 4: Set to enable transmit complete interrupt
```

Preserves: `HL(U)`

Returns:

- `A`: Error code (always 0)

Please note that before MOS 3.0 the return value from this API was not always `0` owing to a bug in how the return value was handled.  This has been fixed in MOS 3.0 and later.

### `0x16`: mos_uclose

Close UART1 (Requires MOS 1.03 or above)

### `0x17`: mos_ugetc

Read a character from UART1 (Requires MOS 1.03 or above)

Returns:

- `A`: The character read
- `F`: C if successful, NC if the UART is closed

NB: If UART1 is open, this is a blocking function which means this routine will wait and only return once a character is received.

### `0x18`: mos_uputc

Write a character to UART1 (Requires MOS 1.03 or above)

Parameters:

- `C`: The character to write

Returns:

- `F`: C if successful, NC if the UART is closed

### `0x19`: mos_getfil

Get a pointer to a `FIL` structure in MOS (Requires MOS 1.03 or above)

This call is useful if you wish to use the FatFS API directly, but need to pass a `FIL` structure to a FatFS API call.

Parameters:

- `C`: File handle

Preserves: `BC(U)`

Returns:

- `HLU`: 24-bit pointer to a `FIL` structure (in MOS RAM)

### `0x1A`: mos_fread

Read a block of data from a file (Requires MOS 1.03 or above)

Parameters:

- `C`: File handle
- `HLU`: Pointer to a buffer to read the data into
- `DEU`: Number of bytes to read

Preserves: `HL(U)`, `BC(U)`

Returns:

- `DEU`: Number of bytes read

### `0x1B`: mos_fwrite

Write a block of data to a file (Requires MOS 1.03 or above)

Parameters:

- `C`: File handle
- `HLU`: Pointer to a buffer that contains the data to write
- `DEU`: Number of bytes to write out

Preserves: `HL(U)`, `BC(U)`

Returns:

- `DEU`: Number of bytes written

### `0x1C`: mos_flseek

Move the read/write pointer in a file (Requires MOS 1.03 or above)

NB this API is deprecated and kept for compatibility reasons.  You are advised to use the [`mos_flseek_p`](#0x24-mos_flseek_p) API instead.  As this API requires a full 24-bit value to be provided in the `HLU` register it is not directly compatible with programs written to run in Z80 mode.

This API can be used to expand the size of a file, although you should note that the file data in the expanded part will be undefined.

Please note that on MOS releases prior to MOS 3.0, the status code returned in the `A` register will be incorrect.

Parameters:

- `C`: File handle
- `HLU`: Least significant 3 bytes of the offset from the start of the file
- `E`: Most significant byte of the offset (set to 0 for files < 16MB)

Preserves: `HL(U)`, `BC(U)`, `DE(U)`

Returns:

- `A`: Status code

### `0x1D`: mos_setkbvector

Allows user programs to access VDP keyboard packets without overriding the entire uart0 interrupt handler.
The user program registered, will be called during the uart0 interrupt handler, being passed the address of the full VDP keyboard packet.

Parameters:

- `C`: Address length in HL (0 = 24bit, 1 = 16bit). If 1 then set the top byte of HLU (callback address) to MB (for ADL=0 callers)
- `HL(U)`: Callback address of user program to register, or 0 to clear any previously registered vector

Returns: Nothing upon registration. The user program can expect the full VDP packet address in DE(24-bit) upon entry.

Be sure to clear the kbvector before your program exits (call mos_setkbvector again with HL=0).

[example code](https://github.com/tomm/toms-agon-experiments/blob/main/custom_kbvector_demo/custom_kbvector_demo.asm) that registers a custom handler.

### `0x1E`: mos_getkbmap

Fetch a pointer to the virtual keyboard map (Requires MOS 1.04 RC2 or above)

NB as this returns a pointer in IXU, and is therefore difficult to use from C code, as of MOS 3.0 an alternative way to access the keyboard bitmap address is available via a C function obtainable from the [`mos_getfunction`](#0x50-mos_getfunction) API.

Parameters: None

Returns:

- `IXU`: Pointer to the keyboard bitmap (this is always 24 bit)

### `0x1F`: mos_i2c_open

Open the I2C bus as Master (Requires MOS 1.04 RC3 or above)

Parameters:

- `C`: Frequency ID (1: 57600, 2: 115200, 3: 230400)

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns: None

### `0x20`: mos_i2c_close

Close the I2C bus (Requires MOS 1.04 RC3 or above)

Parameters: None

Preserves: `HL(U)`, `BC(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns: None

### `0x21`: mos_i2c_write

Write a block of bytes to the I2C bus (Requires MOS 1.04 RC3 or above)

Parameters:

- `C`: I2C Address
- `B`: Number of bytes to write (maximum 32)
- `HL(U)`: Pointer to a buffer to read the bytes from

Preserves: `HL(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: Status
	- `0`: OK
	- `1`: No response from I2C slave
	- `2`: Data NACK
	- `4`: Bus arbitration lost
	- `8`: Bus error

### `0x22`: mos_i2c_read

Read a block of bytes from the I2C bus (Requires MOS 1.04 RC3 or above)

Parameters:

- `C`: I2C Address
- `B`: Number of bytes to read (maximum 32)
- `HL(U)`: Pointer to a buffer to write the bytes to

Preserves: `HL(U)`, `DE(U)`, `IX(U)`, `IY(U)`

Returns:

- `A`: Status
	- `0`: OK
	- `1`: No response from I2C slave
	- `2`: Data NACK
	- `4`: Bus arbitration lost
	- `8`: Bus error

### `0x23`: mos_unpackrtc

Unpack the RTC (Real-Time Clock) data into a buffer (Requires MOS 3.0 or above)

Parameters:

- `HL(U)`: Pointer to a buffer to copy the RTC data to
- `C`: Flags

The buffer should be at least 10 bytes long.

Flags are a bit-field to enable various different options.  Currently the following bits are supported:

- bit 0 = refresh RTC sysvar before unpacking
- bit 1 = refresh RTC sysvar after unpacking

The RTC data sysvar is updated by sending a command to the VDP to request the current time, and MOS will store the response in the [RTC system state information area](#sysvar_rtc).  If you set bit 0, this API call will send the command to the VDP to request updated RTC data and wait for the response before unpacking the data.  When you set bit 1, the command will be sent after the data has been unpacked, and the API will return without waiting for the response.

If you do not want to refresh the RTC data stored in MOS, set the flags to 0.  You should note that if you do this the data may be stale or, if no request has been sent to the VDP at all for RTC information, be invalid.

NB you should not set either refresh flag if you are in the middle of sending a command to the VDP, as that will both cause the update command to not be understood by the VDP, and will cause your command to fail or produce unexpected results.

Preserves: `HL(U)`, `BC(U)`

Returns:

Data returned in the buffer at `HL(U)` will be in the following order, with 16-bit values stored in little-endian order:

```
	UINT16 year;
	UINT8  month;
	UINT8  day;
	UINT8  dayOfWeek;
	UINT16 dayOfYear;
	UINT8  hour;
	UINT8  minute;
	UINT8  second;
```

### `0x24`: mos_flseek_p

Move the read/write offset pointer in a file (Requires MOS 3.0 or above)

This API can be used to expand the size of a file, although you should note that the file data in the expanded part will be undefined.

Whilst the [`mos_flseek`](#0x1c-mos_flseek) API can essentially perform the same function as this API, this is the preferred API to use for moving the current read/write pointer offset within a file.  As it accepts a pointer to the 32-bit offset value, it is compatible with both Z80-mode and ADL-mode code.

Parameters:

- `C`: File handle
- `HL(U)`: Pointer to a 32-bit value for the desired new offset from the start of the file

Preserves: `HL(U)`, `BC(U)`

Returns:

- `A`: Status code

***

### `0x28-0x2C`: String functions

API calls in this range are string manipulation functions.

All of the API calls in this range require MOS 3.0 or above.

### `0x28`: mos_pmatch

Pattern matching function, with support for various flags to control how the comparison is made.

This exposes the pattern matching function that MOS 3.0 uses internally for matching commands and filenames.

Parameters:

- `HL(U)`: Address of pattern (zero terminated)
- `DE(U)`: Address at string to compare against pattern (zero terminated)
- `C`: Flags

The flags are a bit-field to enable various different pattern matching options

| Bit | Description |
| --- | ----------- |
| 0   | Case insensitive |
| 1   | Disable star (`*`) wildcard matching |
| 2   | Disable hash (`#`) wildcard matching |
| 3   | "Dot as star" mode (treats `.` at the end of pattern as a star, used in MOS for matching abbreviated commands) |
| 4   | "Begins with" mode (only matches if the string starts with the pattern) |
| 5   | "Up to space" mode (only matches up to the first space in the pattern) |

Preserves: `HL(U)`, `DE(U)` and `BC(U)`

Returns:

- `A`: Status
    - `0` if pattern matches
	- Positive number if string does not match, and is logically sorted after the pattern
	- Negative number if string does not match, and is logically sorted before the pattern

### `0x29`: mos_getargument

Extract a (numbered) argument from a string

Parameters:

- `HL(U)`: Pointer to source string
- `BC(U)`: Argument number

Returns:

- `HL(U)`: Address of the argument or zero if not found
- `DE(U)`: Address of the argument end (next character after the argument)

Preserves: `BC(U)`

### `0x2A`: mos_extractstring

Extract a string, using a given divider.

Parameters:

- `HL(U)`: Pointer to a zero-terminated source string to extract from
- `DE(U)`: Pointer to string for divider matching, or `0` for default (space)
- `C`: Flags. Depending on flags, the result string will be zero terminated or not

The flags are a bit-field to enable various different options

| Bit | Description |
| --- | ----------- |
| 0   | Zero terminate the result string |
| 1   | Omit skipping of divider characters at beginning of source string |
| 2   | Disable matching of double-quotes |
| 3   | Include double-quotes in results string |

If string extraction is matching double-quotes and an end quote is not found, a status code of `25` (Bad string) will be returned.  If it is not possible to extract a result, a status code of `19` (Invalid parameter) will be returned.

Returns:

- `A`: status code (`0` = OK, `19` = Invalid parameter, `25` = Bad string)
- `HL(U)`: Address of the result string
- `DE(U)`: Address of next character after end of result string

Preserves: `BC(U)`

### `0x2B`: mos_extractnumber

Extract a number, using given divider.  Various number formats are supported - for more information see notes on numbers interpreted by the [MOS CLI](./Star-Commands.md)

Parameters:

- `HL(U)`: Pointer to a zero-terminated source string to extract from
- `DE(U)`: Pointer to string for divider matching, or 0 for default (space)
- `C`: Flags

The flags are a bit-field to enable various different options

| Bit | Description |
| --- | ----------- |
| 0   | Decimal numbers only |
| 1   | Positive numbers only |
| 2   | Allow `h` suffix to indicate hexadecimal numbers |

If the source string does not contain a valid number, in accordance with the flags, a status code of `19` (Invalid parameter) will be returned.

Returns:

- `A`: status code (`0` = OK, `19` = Invalid parameter)
- `HL(U)`: Number extracted
- `DE(U)`: Address of next character after end of number

Preserves: `BC(U)`

### `0x2C`: mos_escapestring

"Escape" a string for display, converting control characters to be pipe-prefixed

Parameters:

- `HL(U)`: Pointer to source string
- `DE(U)`: Pointer to destination buffer (optional)
- `BC(U)`: Length of destination buffer

If no destination buffer is provided (i.e. `DE` is zero), the function will return the length of the escaped string.

If the destination buffer is too short then a status code of `22` (Out of memory) will be returned, and as much of the source string that could be converted will be copied to the destination buffer.

Returns:

- `A`: Status code
- `BC(U)`: Length of escaped string

***

### `0x30-0x37`: System variables and string translations

API calls in this range are used for setting and reading system variables, and for performing string translations.

All of the API calls in this range require MOS 3.0 or above.

### `0x30`: mos_setvarval

Set, update, replace or remove a [System Variable](./System-Variables.md)

Parameters:

- `HL(U)`: Pointer to variable name (zero-terminated string, can include wildcards)
- `IX(U)`: Variable value (number, or pointer to zero-terminated string)
- `IY(U)`: Pointer to variable name (0 for first call)
- `C`: Variable type, or -1 (255) to delete the variable

If the name used includes a wildcard, then the first matching variable will be set.  Subsequent calls can be made to set the next variable that matches the pattern, so long as you preserve `IY(U)` between calls.

Variable types supported are:

| Type | Description |
| ---- | ----------- |
| 0    | String (Will be run through GSTrans before storing) |
| 1    | Number (a 24-bit integer value) |
| 2    | Macro (A string that will be GSTrans'd each time it is used) |
| 3	   | "Expanded" (Expression that will be evaluated before stored) |
| 4    | Literal string (GSTrans will not be called before storage) |

NB at the time of writing the expression engine has yet to be written in MOS 3 so type 3 support is limited to either a number or a string to indicate a variable name to copy as a string type.

If either a type of 3 or 4 is used then the variable type used for storage of the variable will be either a string or a number.

Internally MOS also supports "Code" type variables, which are used for several things, such as exposing date/time information.  If such a variable variable supports being set then you can call the "set" functions of these variables by using the String type with a matching name.  You cannot remove a "Code" type variable using this function.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code
- `C`: Actual variable type
- `IY(U)`: Pointer to actual variable name (for next call)

Other registers will be preserved.

### `0x31`: mos_readvarval

Read a variable value

Parameters:

- `HL(U)`: Pointer to variable name (can include wildcards)
- `DE(U)`: Length of buffer to store the value
- `IX(U)`: Pointer to buffer to store the value (null/0 to read length only)
- `IY(U)`: Pointer to variable name (0 for first call)
- `C`: Flags (3 = expand value into string - other values will be ignored)

If the name includes a wildcard then the first matching variable will be read.  Subsequent calls can be made to read the next value that matches the pattern, so long as you preserve `IY(U)` between calls.

Please note that whilst numeric variables are set using [`mos_setvarval`](#0x30-mos_setvarval) by providing their value directly in a register, when reading a numeric variable the value will be returned in the buffer pointed to by `IX(U)`.  That buffer must be at least 3 bytes long, unless the "flag" value is set to `3`, in which case it must be large enough to contain a string representation of the number in decimal.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code
- `C`: Actual variable type
- `DE(U)`: Length of variable value
- `IY(U)`: Pointer to variable name (for next call)

### `0x32`: mos_gsinit

Initialises a GSTrans operation.

GSTrans is a process of taking a source string and translating it, replacing any variables referenced, and converting any control codes into raw control bytes.  The [`echo` command](./Star-Commands.md#echo) in MOS is an example of a command that uses the GSTrans process, and the documentation for that command contains a more complete description of how the source string will be interpreted.

The process of translating a string is a two-step process.  The first step is to call `mos_gsinit` to initialise the process, and the second step is to repeatedly call `mos_gsread` to actually perform the translation, fetching one character at a time until the whole string has been translated, and a zero character is read.

Various options are available to control how the translation process operates, and these are set using the flags parameter.  By default, the translation process will continue until the end of the source string is reached.  It is possible to instead translate only up to the first space.  The process also supports detecting double-quotes to surround a string, in which case a request to terminate at a space will be ignored if the space is inside the double-quotes.  (It should be noted that the `echo` command sets this flag.)  Finally by default the process will translate characters preceeded by a `|` character as control codes, as explained in the documentation for the [`echo` command](./Star-Commands.md#echo).  If you wish to disable this behaviour then you can set the "no pipe" flag.

Parameters:

- `HL(U)`: Pointer to source buffer to translate
- `DE(U)`: Address of pointer used to store trans info
- `C`: Flags

`DE(U)` must point to an address that will be used to store a (3-byte) pointer to an information block used by the GSTrans process.  It should be noted that failing to complete the GSTrans process can result in a memory leak.

The flags are a bit-field with options to control how the translation process operates.  Their meanings are as follows:

| Bit | Description |
| --- | ----------- |
| 0   | Terminate at first space character in source buffer |
| 1   | No pipe (do not treat `|` as a control code) |
| 2   | Do not treat double-quotes `"` as markers surrounding a string |
| 3-6 | Reserved for future use (for future compatibility, ensure these are set to zero) |
| 7   | No tracking (do not automatically track memory used by GSTrans) |

It should be noted that failing to complete translation of a string will result in some memory inside MOS remaining set aside to store the GSTrans process information.  This memory will either be freed when the GSTrans process is completed, or a new call to `mos_gsinit` is made.  If the "No tracking" flag is set however then MOS will not track the memory being used, and it will not be freed on a subsequent call to `mos_gsinit` - you must complete the process of reading through the string to free the memory, or you will cause a memory leak inside MOS.

In general you should not use the "No tracking" flag unless you have a need to perform two or more gstrans operations simultaneously.

Returns:

- `A`: Status code

### `0x33`: mos_gsread

Perform a GSTrans "read" operation.

When the final character of the translated string has been read, this function will return a null character (`0`) to indicate the end of the string.

Parameters:

- `DE(U)`: Address of pointer used to store trans info (same pointer as used with gsInit)

Returns:

- `A`: Status code (`0` = Success, various other values may indicate an invalid GSTrans string)
- `C`: Character read.  This will be `0` if the end of the string has been reached.

### `0x34`: mos_gstrans

Perform a complete GSTrans operation from source into dest buffer

Parameters:

- `HL(U)`: Pointer to source buffer
- `IX(U)`: Pointer to destination buffer (can be null to just count size)
- `DE(U)`: Length of destination buffer
- `C`: Flags

If the destination buffer given for this command is less than the size required for the translated string, then the API call will succeed, but the translated string will be truncated to fit in the buffer.

The flags here are identical those used with `mos_gsinit`, with the exception that the "No tracking" flag will be ignored.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code
- `BC(U)`: Calculated total length of translated string

### `0x35`: mos_substituteargs

Substitute arguments into a string from template

Parameters:

- `HL(U)`: Pointer to template string
- `IX(U)`: Pointer to arguments string
- `DE(U)`: Length of destination buffer
- `IY(U)`: Pointer to destination buffer (can be null to just count size)
- `C`: Flags

The only flag currently supported is bit 0, which when set indicates that the "rest" arguments (i.e. those not explicitly used in the template) should be omitted from the destination string.  When this bit is clear they will be automatically appended.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `BC(U)`: Calculated length of destination string

### `0x36`: mos_evaluateexpression

As of MOS 3.0 this function has not yet been implemented.  Support for this function is planned for a future release.

***

### `0x38-0x3C`: File path functions

These APIs provide a set of functions for working with and manipulating file paths.

All of the API calls in this range require MOS 3.0 or above.

### `0x38`: mos_resolvepath

Resolves a path, creating a new resolved path string that expands [system variables](./System-Variables.md), and also replaces [prefixes](./System-Variables.md#path-variables) and leafnames with actual values.  System variables will be expanded first, and then the prefix and leafname will be resolved.  The result is a fully resolved path that can be used with the FatFS API calls.

If the leafname contains wildcards then the first matching file will be returned.  Please note that this will be the first match found in a directory, and owing to how directories are managed this may not be the first alphabetical match.  Subsequent calls can be made to find the next matching file, so long as you provide a pointer to an empty directory object to persist between calls, and preserve the `C` register between calls too.

NB path resolution does not support resolving paths that contain wildcards in the directory part of the path.  If you need to do this then you should gradually build the path up by calling `mos_resolvepath` multiple times to resolve the path up to each wildcard, using flags to filter for results that are a directory matching the wildcard until the directory part of the path is fully resolved.  You can then call `mos_resolvepath` again with different flags to resolve the leafname part of the path.

Parameters:

- `HL(U)`: Pointer to the path to resolve (zero terminated)
- `IX(U)`: Pointer to destination buffer to store the resolved path (optional - set to zero for length count only)
- `DE(U)`: Length of the destination buffer
- `IY(U)`: Pointer to a directory object to persist between calls (optional, set to zero to omit)
- `B`: Flags
- `C`: Index of the resolved path (zero for first call)

Path resolution will resolve prefixes in files paths, which are a string followed by a colon character, such as `Library:`.  The string must match up with a corresponding system variable, in this example the variable would be named `Library$Path`.  Prefixes are not case-sensitive, so `library:`, `Library:` and `LIBRARY:` would all match in this example.  Such path variables can contain multiple values separated by commas.  The "index" argument in the `C` register is used to work out which prefix to use when there are multiple matches, and must be set to zero on a first call.

If you are resolving for a file that does not exist, the path will be resolved to the first matching directory, returning a "no file" (`4` status code).  If no matching directory can be found the path will not be resolved, and a "no path" (`5` status code) will be returned, and the returned path will be empty.  If the path is resolved successfully to an existing file the status code will be `0`.

The `flags` argument is a bit-field used to indicate which files are valid to be returned.  These bits are used to filter the results based on the file attributes, which are inherited from the file system (FatFS).  If you wish to always return all results you should set the flags to `0`.  The bits are as follows:

| Bit | FatFS constant | Description |
| --- | -------------- | ----------- |
| 0 | `AM_RDO` | Read only |
| 1 | `AM_HID` | Hidden |
| 2 | `AM_SYS` | System |
| 3 | n/a | Reserved/Unused - set to zero |
| 4 | `AM_DIR` | Directory |
| 5 | `AM_ARC` | Archive |
| 6 | n/a | Set to disable system variable expansion |
| 7 | n/a | Include/exclude in results |

System variable expansion passes the source path through the [GSTrans](#0x34-mos_gstrans) process, replacing any system variables used in the path with their values.  If you have already performed this step then you can set bit 6 to disable the expansion.

Bit 7 is used to indicate how to apply the attributes to filter results.  When it is set, any result must include _all_ of the attributes set.  When it is clear, the result will not include any of the attributes set.  This can be used, for example, to filter out hidden or system files, or alternatively to only include results that are directories.

NB any attributes that are not set in the `flags` argument will be ignored, so if you perform a search with a flags value of `0x06`, which will filter out hidden and system files, your results may include items that have their directory, read-only, and/or archive bits set.  Similarly a search with a flags value of `0x90` will only include directories, but those directories could have any of the other attributes set.

If you wish to further filter results you should pass in a `DIR` object in `IY(U)` to persist between calls, and use the [`ffs_stat`](#0x96-ffs_stat) API call to get a `FILINFO` object where you can check the full attributes of the the returned result (stored in the `fattrib` field).  If the results do not match your requirements you can then call `mos_resolvepath` again to find the next matching file.

If the leafname in your path contains wildcards then the first matching file will be returned.  If you wish to find the next matching file then you should provide a pointer to an empty directory object in `IY(U)` with your first call, and use the same object with subsequent calls (also using the same `C` register value).  If you do not include a pointer to a directory object then only the first matching file within a single directory will be returned.

Besides finding filecard matches, the other main use for this API is to resolve a path so that it can be used with [FatFS API calls](#fatfs-commands).  This step is needed to ensure that the path is fully resolved, as the FatFS API calls do not support using path prefixes or system variables.  Attempting to use an unresolved path with a fatfs API call may either fail or produce unexpected results.

It is not necessary to use this API call to resolve paths for use with the MOS-native API calls, as those will all automatically resolve paths for you.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code (`0` = Success, `22` = Out of memory, `5` = No path, `4` = No file)
- `C`: Index of the resolved path (for next call)
- `DE(U)`: Length of the resolved path

If this is called with a null pointer in `IX(U)` then the maximum possible length of of all possible resolved paths will be returned in `DE(U)`.  This is useful if you wish to allocate a buffer for the resolved path, but do not know how long it will be.

A result of `5` indicates that no matching directory could be found in the filing system.  A result of `4` indicates that a matching directory was found, but no matching file for that directory.

A result of `22` indicates that either the resolved path was too long for the buffer provided, or that there was an error allocating memory whilst searching for a matching path.

### `0x39`: mos_getdirforpath

Get the directory for a given path.

This function works with strings only - it resolves path prefixes for the given index, but does not check whether the path actually points to a valid file or directory.  The index is used which prefix to use when the path prefix variable contains multiple options.

The returned path will be the directory part of the path, and will not include the leafname.

Parameters:

- `HL(U)`: Pointer to the path to get the directory for
- `IX(U)`: Pointer to the buffer to store the directory in (optional - omit for count only)
- `DE(U)`: Length of the buffer
- `C`: Search index

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code
- `DE(U)`: Length of the directory path

### `0x3A`: mos_getleafname

Get the leafname for a given path.

This is a utility function that will scan a file path string and return a pointer within that string to the leafname.  The leafname is the last part of a file path, and may refer to a file or a directory.  This call does not check whether the path actually points to a valid file or directory, and it does not resolve any path prefixes.  It should be noted that a leafname may be empty if the path is empty or ends with a `/` or `:` character.

If the path does not contain a valid leafname then the function will return a pointer to the end of the string.

Parameters:

- `HL(U)`: Pointer to the path to get the leafname from

Returns:

- `HL(U)`: Pointer to the leafname

### `0x3B`: mos_isdirectory

Checks if a given path points to a directory

NB this call only works with fully resolved paths, and as such does not perform path prefix resolution.  You may therefore need to use [`mos_getdirforpath`](#0x39-mos_getdirforpath) or [`mos_resolvepath`](#0x38-mos_resolvepath) first.

Parameters:

- `HL(U)`: Pointer to the path to check

Returns:

- `A`: Status code (`0` = Success, `5` = No path)


### `0x3C`: mos_getabsolutepath

Get the absolute version of a (relative) path

NB currently as of MOS 3.0, unlike similar API calls above, this API does not support being called with a null pointer to count the length of the resolved path, and does not return the length.  If called with a null pointer from ADL mode, the API will return a status code of `19` (Invalid parameter).  Calling with a null pointer from Z80 mode code is not currently supported/checked and may cause unexpected results or crashes.  This will likely change in a future MOS release.

Parameters:

- `HL(U)`: Pointer to the path to get the absolute version of
- `IX(U)`: Pointer to the buffer to store the absolute path in
- `DE(U)`: Length of the buffer

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: Status code

If the buffer is too short for the resolved path then a status code of `22` (Out of memory) will be returned.  Path resolution problems may result in status codes of `5` (No path) or `4` (No file).

***

### `0x40-0x50`: VDP protocol, and miscellaneous functions

Functions in this range are used for VDP protocol, and other miscellaneous functions.

All of the API calls in this range require MOS 3.0 or above.

### `0x40`: mos_clearvdpflags

Clears [VDP protocol status flags](#VDP-pflags-vdpflags) from the `sysvar_vpd_pflags` [sysvar](#sysvars).

Bits in this status value will be set when various different VDP Protocol message packet types are received by MOS from the VDP.  Such packets may be sent for user-initiated actions, such as pressing a key on the keyboard, or moving the mouse, or for system-initiated actions, such as a VDU command being sent to the VDP.  Please note that in general use these bits are not automatically cleared, so if you wish to detect a response from the VDP to a VDU command your program sends it is important to clear out the corresponding protocol flags before sending the command.  You can then use the [`mos_waitforvdpflags`](#0x41-mos_waitforvdpflags) API call to wait for the VDP to respond, and check the status of the flags in `sysvar_vpd_pflags` to see if the command was successful.

Further information on the VDP protocol and the flag bits can be found in the [VDP Protocol documentation](../vdp/System-Commands.md#vdp-serial-protocol).

Parameters:

- `C`: Bitmask of flags to clear

Returns:

- `A`: New VDP flags

### `0x41`: mos_waitforvdpflags

Waits for corresponding [VDP protocol status flags](#VDP-pflags-vdpflags) to be set in the `sysvar_vpd_pflags` [sysvar](#sysvars).

Typically your program should first clear whichever protocol flag you are interested in detecting, using the [`mos_clearvdpflags`](#0x40-mos_clearvdpflags) API call.  Once you have done that, you should send a VDU command to the VDP for which you are expecting a response, and then use this API call to wait for that response to be received.

MOS contains inbuilt support for handling VDP protocol messages.  Typically on receipt of a message a flag will be set in the `sysvar_vpd_pflags` variable, and also other corresponding [sysvars](#sysvars) will be updated from the contents of the message.

This API call will wait for approximately 1 second for the VDP to respond, which should be more than enough time for any VDU command to be processed.  If the VDP does not respond within this time, the API call will return with a status code of `15` (Timeout).  If the VDP does respond, the API call will return with a status code of `0` (Success).

Parameters:

- `C`: Bitmask of flags to wait for

Returns:

- `A`: Status code (`0` = Success, `15` = Timeout)

### `0x50`: mos_getfunction

This API call will return a pointer to a function that follows the Zilog eZ80 C calling convention.  As these are 24-bit pointers, and passing arguments needs to be done using the 24-bit stack pointer (`SPL`), this API call is not usable by programs running in Z80 mode.

Information on the C calling convention can be found [here](https://github.com/pcawte/AgDev?tab=readme-ov-file#c-calling-conventions-for-interfacing-with-asm-applications)

Parameters:

- `C`: Flags (must be `0` in MOS 3.0)
- `B`: Function number

Returns:

- `A`: Status code (`0` = Success, `19` = Invalid parameter, `20` = Invalid command)
- `HLU`: Pointer to function (or `0` if the request was invalid invalid)

If this API is called from Z80 mode code, then it will return a status code of `20` (Invalid command) as this API is only supported in ADL mode.

This API includes a flags byte which must be set to zero in MOS 3.0.  This is reserved for future use to allow for additional functionality to be added in future versions of MOS.  If flags are set to anything other than `0` then the API will return a status code of `19` (Invalid parameter).

Similarly, if the function number passed to this API is higher than the highest available function, then the API will return a status code of `20` (Invalid command).

Full details of the functions available through this API, and more information about how to use them, can be found in the documentation about [C Functions](./C-Functions.md)

***

## `0x70-0x73`: Low-level SD card access 

These APIs provide low-level access to the SD card.  They are not intended for general use, but are provided for some special use-cases, such as for an operating system that uses a different filing system than FatFS, or for tools that wish to access the SD card in ways that are not supported by the FatFS library.

As the use of these APIs can potentialy cause data corruption in order to use them you need to use an "unlock code".  It is possible to avoid using the unlock mechanism by using the underlying functions directly via the [`mos_getfunction`](#0x50-mos_getfunction) API call.

All of the APIs in this range require MOS 3.0 or above.

### `0x70`: sd_getunlockcode

This API call is used to obtain the unlock code needed to use the low-level SD card APIs.  The unlock code is a randomly generated 24-bit value, created the first time this API is called.

Parameters:

- `HL(U)`: Pointer to a 24-bit value to store the unlock code

Returns:

Nothing

### `0x71`: sd_init

Initialises the SD card support system.  MOS automatically calls this when it mounts an SD card.  If you are writing support for an operating system you may need to call this to restart the SD card system if the SD card is removed and reinserted.

Parameters:

- `HL(U)`: Pointer to the unlock code (24-bit value) as fetched by the [`sd_getunlockcode`](#0x70-sd_getunlockcode) API call

Returns:

- `A`: Status code
    - `0` = Success/Ready
	- `1` = Error
	- `2` = Locked (incorrect unlock code provided, or no lock code yet set)

### `0x72`: sd_readblocks

Read raw blocks from the SD card.

Parameters:
- `HL(U)`: Pointer to a 32-bit sector number, followed by the 24-bit unlock code
- `DE(U)`: Pointer to a buffer to store the read data
- `BC`: Number of blocks to read (16-bit value)

Returns:

- `A`: Status code
    - `0` = Success/Ready
	- `1` = Error
	- `2` = Locked (incorrect unlock code provided, or no lock code yet set)

### `0x73`: sd_writeblocks

Writes raw blocks to the SD card.

Parameters:

- `HL(U)`: Pointer to a 32-bit sector number, followed by the 24-bit unlock code
- `DE(U)`: Pointer to a buffer containing the data to write
- `BC`: Number of blocks to write (16-bit value)

Returns:

- `A`: Status code
	- `0` = Success/Ready
	- `1` = Error
	- `2` = Locked (incorrect unlock code provided, or no lock code yet set)

***

## `0x80-0xA6`: FatFS APIs {#fatfs-commands}

MOS makes use of the [FatFS library](http://elm-chan.org/fsw/ff/00index_e.html) to access the SD card.  Some of FatFS's functionality is exposed as APIs in MOS.  These APIs are essentially provide a way to call the underlying FatFS functions that MOS uses to perform file operations.  The API calls are prefixed with `ffs_` to indicate that they are FatFS functions, and the `mos_` prefix is used for the native MOS API calls.

Our naming convention for FatFS APIs in MOS is to remove the `f_` prefix from the FatFS function name, and replace it with `ffs_`, plus an optionally `f` or `d` prefix.  For example, the API that exposes the `f_open` FatFS function is named `ffs_fopen`.

The variety of FatFS APIs supported in the MOS 1.x and MOS 2.x releases was limited to a restricted subset of functionality.  The first version of MOS that supported FatFS API calls was 1.03.  Versions of MOS 2.x added support for some additional APIs.  MOS 3.0 includes support for all of the possible FatFS APIs that are practical to support with our current configuration of FatFS, which greatly expands the available set of APIs.  For completeness, all potential FatFS APIs are documented below, including those that are not supported by our current configuration.  The APIs that are not supported will return a status code of `23` (Not implemented) in the A register.

When reading the documentation below, you should assume that the API calls are only available in MOS 3.0 or above, unless otherwise stated.

Please note that the FatFS API calls documented below expect file paths to be fully resolved, i.e. they should not include [path prefixes](./System-Variables.md#path-variables) or [system variables](./System-Variables.md).  This means programs running on MOS 3 should use the [`mos_resolvepath` API call](#0x38-mos_resolvepath) to resolve any paths before using them with a FatFS API call.  If you do not resolve the path then the FatFS API call may fail or produce unexpected results.  For many API calls you can instead open the file using the MOS API call [`mos_fopen`](#0x0a-mos_fopen) and then use [`mos_getfil`](#0x19-mos_getfil) to get a pointer to a `FIL` structure that many FatFS API calls require.

For more information on FatFS data structures (the `FIL`, `DIR` and `FILINFO` objects), functions, and info on which bits to set in fields such as "File open mode" please see the [FatFS documentation](http://elm-chan.org/fsw/ff/00index_e.html).  The FatFS configuration can affect the contents of these data structures - our configuration has the `FF_USE_LFN` and `FF_USE_FIND` options set, and does _not_ set `FF_READ_ONLY` or `FF_USE_FASTSEEK`.

### `0x80`: ffs_fopen

Open a file (Available from MOS 1.03)

Parameters:

- `HL(U)`: Pointer to an empty `FIL` structure
- `DE(U)`: Pointer to a C (zero-terminated) filename string
- `C`: File open mode

The file open mode on this API is a bit-field that is identical to the [mode used](#file-open-modes) in the [`mos_fopen`](#0x0a-mos_fopen) API call.

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`, `C`

Example:

```
			LD	HL, fil				; FIL buffer
			LD	DE, filename			; Filename (0 terminated)
			LD	C, fa_read			; Mode
			MOSCALL	ffs_fopen			; Open the file
			LD	DE, buffer			; Where to store the read file
			LD	BC, 256				; Number of bytes to read
			MOSCALL	ffs_fread			; Read the data in
			PUSH	BC				; Preserve number of bytes read
			MOSCALL	ffs_fclose			; Close the file
			POP	BC				; BC: Number of bytes read
			RET

filename:		DB	"example.txt", 0		; The file to read

fil:			DS	FIL_SIZE			; FIL buffer (defined in mos_api.inc)
buffer:			DS	256				; Buffer for storing read data
```

### `0x81`: ffs_fclose

Close a file (Available from MOS 1.03)

Parameters:

- `HL(U)`: Pointer to a `FIL` structure

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`

See [`ffs_fopen`](#0x80-ffs_fopen) for an example.

NB: you should not use this call to close a file that had been opened using [`mos_fopen`](#0x0a-mos_fopen), as doing so will mean that MOS will be reserving an file handle for a file that has been closed.

### `0x82`: ffs_fread

Read from a file (Available from MOS 1.03)

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a buffer to store the data in
- `BC(U)`: Number of bytes to read (typically the size of the buffer)

Returns:

- `A`: `FRESULT`
- `BC(U)`: Number of bytes read

Preserves: `HL(U)`, `DE(U)`

See ffs_fopen for an example

### `0x83`: ffs_fwrite

Write to a file (Available from MOS 1.03)

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a buffer to read the data from
- `BC(U)`: Number of bytes to write (typically the size of the buffer)

Returns:

- `A`: `FRESULT`
- `BC(U)`: Number of bytes written

Preserves: `HL(U)`, `DE(U)`

Example:

```
			LD	HL, fil				; FIL buffer
			LD	DE, filename			; Filename (0 terminated)
			LD	C, fa_write | fa_create_always	; Mode
			MOSCALL	ffs_fopen			; Open the file
			LD	DE, buffer			; Location of data to write
			LD	BC, 256				; Number of bytes to write
			MOSCALL	ffs_write			; Write the data
			MOSCALL	ffs_fclose			; Close the file
			RET

filename:	DB	"example.txt", 0		; The file to read

fil:		DS	FIL_SIZE			; FIL buffer (defined in mos_api.inc)
buffer:		DS	256				; Buffer containing data to write out
```

### `0x84`: ffs_flseek

Move the read/write pointer in a file (Available from MOS 1.03)

NB this API is deprecated and kept for compatibility reasons.  You are advised to use the [`ffs_flseek_p`](#0xa6-ffs_flseek_p) API instead.  As this API requires a full 24-bit value to be provided in the `DE(U)` register it is not directly compatible with programs written to run in Z80 mode.

This API call can also be used to expand the file size, by moving the pointer to a location beyond the current end of the file.  It should be noted that the extra allocated disk space will not be cleared, so the data in the new space will be undefined.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Least significant 3 bytes of the offset from the start of the file
- `C`: Most significant byte of the offset (set to 0 for files < 16MB)

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`, `BC(U)`

### `0x85`: ffs_ftruncate

Truncate a file to the current file pointer offset (Available from MOS 2.3.0)

To truncate to a specified size you will need to use [`ffs_flseek_p`](#0xa6-ffs_flseek_p) to move the file pointer to the desired location before calling `ffs_ftruncate`.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`

### `0x86`: ffs_fsync

Flushes cached information of a writing file

When writing to a file, file data may be cached in memory until the file is closed.  This function will flush the cache to the SD card, ensuring that all data has been written.  This can be useful to minimise the risks of data loss in the event of a power failure, SD card removal, or other unexpected shutdown.

Please note that MOS 3.0 does not currently cache writes to the SD card.  This API call is provided in case we do add caching in the future.  You may still wish to use this call to ensure that data is guaranteed to be written to the SD card in the event that a future version of MOS adds caching support.  Calling this API on systems that do not support caching will have no effect and be harmless.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure

Returns:

- `A`: `FRESULT`

### `0x87`: ffs_fforward

This API is not implemented, as the FatFS `f_forward` function is not supported by the current configuration of the FatFS library used in MOS.

The documented purpose of the `f_forward` function is to read file data and forward it to a "data streaming device" (a callback function).  It is unlikely that this feature would be enabled in the future as we do not have a clear use-case for it, and similar functionality can be achieved by other means.

Returns:

- `A`: `23` (Not implemented)

### `0x88`: ffs_fexpand

This API is not implemented, as the FatFS `f_expand` function is not supported by the current configuration of the FatFS library used in MOS.

The purpose of the `f_expand` function is to expand a file allocating a contiguous data area to the file.  Files can be expanded using the [`ffs_flseek`](#0x84-ffs_flseek) API, although this will not guarantee that the data area is contiguous.  It is unlikely that this API will be implemented in the future, as on an SD card there is little advantage to be had in allocating a contiguous data area for a file.

Returns:

- `A`: `23` (Not implemented)

### `0x89`: ffs_fgets

Reads a string from a file

Reads characters into a buffer until a newline `\n` character is reached, the end of file encountered, or the buffer is filled.  The string read will be zero terminated.

It should be noted that this API does not return a status code in the `A` register.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a buffer to store the string in
- `BC(U)`: Buffer size

Returns:

- `DE(U)`: Pointer to the target buffer, or NULL if an error occurred

Preserves: `HL(U)`, `BC(U)`

### `0x8A`: ffs_fputc

Writes a single character to a file

It should be noted that this API does not return a status code in the `A` register.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `C`: Character to write

Returns:

- `BC(U)`: Number of bytes written

Preserves: `HL(U)`

### `0x8B`: ffs_fputs

Writes a zero-terminated string to a file.  The termination character will not be written to the file.

It should be noted that this API does not return a status code in the `A` register.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a zero-terminated string

Returns:

- `BC(U)`: Number of bytes written

Preserves: `HL(U)`, `DE(U)`

### `0x8C`: ffs_fprintf

Whilst our configuration of FatFS does support the `f_printf` function, we do not currently support it in the MOS API.  This is because the function accepts a variable number of arguments, and it is not clear how this could be implemented in a way that is consistent with the rest of the APIs that MOS supports.

The `f_printf` function is made available via the [`mos_getfunction`](#0x50-mos_getfunction) API call.  It can therefore be used from any (ADL mode) code that complies with the Zilog eZ80 C calling convention.  This API is not available in Z80 mode code.

### `0x8D`: ffs_ftell

Get the current read/write offset pointer of a file

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a 4-byte buffer to store the returned 32-bit offset in

Returns:

- `A`: `FRESULT` (`0` = Success, or `19` = Invalid parameter)

Preserves: `HL(U)`, `DE(U)`

### `0x8E`: ffs_feof

Detect end of file

Please note that whilst this API has been present since MOS 1.03 its implementation had an error which meant that it would return the value of the `L` register passed in as a parameter, rather than the correct value.  This was fixed in MOS 3.0.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure

Returns:

- `A`: 1 if at the end of the file, otherwise 0

Preserves: `HL(U)`

### `0x8F`: ffs_fsize

Get the size of a file

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a 4-byte buffer to store the returned 32-bit file size in

Returns:

- `A`: `FRESULT` (`0` = Success, or `19` = Invalid parameter)

Preserves: `HL(U)`

### `0x90`: ffs_ferror

Tests for an error on a file

Returns a non-zero value if a hard error has returned, otherwise will return a status of `0` (OK).

Parameters:

- `HL(U)`: Pointer to a `FIL` structure

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`

### `0x91`: ffs_dopen

Open a directory (Available from MOS 2.2.0)

Parameters:

- `HL(U)`: Pointer to a blank `DIR` structure
- `DE(U)`: Pointer to a C (zero-terminated) directory path string

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`

### `0x92`: ffs_dclose

Close a directory (Available from MOS 2.2.0)

Parameters:

- `HL(U)`: Pointer to a `DIR` structure

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`

### `0x93`: ffs_dread

Read next directory entry into a `FILINFO` data structure (Available from MOS 2.2.0)

Parameters:

- `HL(U)`: Pointer to a `DIR` structure
- `DE(U)`: Pointer to a `FILINFO` structure

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`

### `0x94`: ffs_dfindfirst

Searches a directory for a matching item

Parameters:

- `HL(U)`: Pointer to a blank `DIR` struct
- `DE(U)`: Pointer to a blank `FILINFO` struct
- `BC(U)`: Pointer to directory path string
- `IX(U)`: Pointer to matching pattern string

The directory path provided in `BC(U)` must be a fully resolved path, and the matching pattern in `IX(U)` should be a zero-terminated string.  The matching pattern can include wildcards, such as `*` and `?`, and will be used to match against the directory entries.  Subsequent entries can be found using [`ffs_dfindnext`](#0x95-ffs_dfindnext) passing in the same `DIR` structures as passed to this call, and the pattern string must also be preserved.

If a matching file is found, the `FILINFO` structure will be populated with information about the file.

NB as this API requires a pointer in `IX(U)`, and is therefore difficult to use from C code, the underlying C function this API uses can be accessed using the [`mos_getfunction`](#0x50-mos_getfunction) API.

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`, `BC(U)`, `IX(U)`

### `0x95`: ffs_dfindnext

Find next matching item in a directory

This API call is used to find the next matching item in a directory after a successful call to [`ffs_dfindfirst`](#0x94-ffs_dfindfirst).  It will return the next matching file or directory in the same way as `ffs_dfindfirst`, but will not require the path and pattern strings to be passed in again.  Please note that whilst it is not a parameter for this API, the pattern string passed to `ffs_dfindfirst` must be preserved, as it will be used to match against the directory entries.

Parameters:

- `HL(U)`: Pointer to a `DIR` structure, as set up by [`ffs_dfindfirst`](#0x94-ffs_dfindfirst)
- `DE(U)`: Pointer to a `FILINFO` structure

NB for completeness/convenience as the [`ffs_dfindfirst`](#0x94-ffs_dfindfirst) API is available as a C function via the [`mos_getfunction`](#0x50-mos_getfunction) API, this API is also available as a C function.

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`

### `0x96`: ffs_stat

Get file information (Available from MOS 1.03)

Parameters:

- `HL(U)`: Pointer to a `FILINFO` structure
- `DE(U)`: Pointer to a C (zero-terminated) filename string

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`

Example:

```
			LD	HL, filinfo				; FILINFO buffer
			LD	DE, filename			; Filename (0 terminated)
			MOSCALL	ffs_stat
			RET

filename:	DB	"example.txt", 0		; The file to read

filinfo:	DS	FILINFO_SIZE			; FILINFO buffer (defined in mos_api.inc)
```

### `0x97`: ffs_unlink

Removes ("unlinks") a file or sub-directory from the volume (Requires MOS 3.0 or above)

Parameters:

- `HL(U)`: Pointer to a zero-terminated file path string

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`

### `0x98`: ffs_rename

Rename and/or move a file or sub-directory

This is a raw rename function that does not perform any path resolution, and does not support wildcards.  For more sophisticated behaviour you should use the [`mos_ren` API](#0x06-mos_ren) instead.

Parameters:

- `HL(U)`: Pointer to a zero-terminated source file path string
- `DE(U)`: Pointer to a zero-terminated destination file path string

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `DE(U)`

### `0x99`: ffs_chmod

This API is not implemented, as the FatFS `f_chmod` function is not supported by the current configuration of the FatFS library used in MOS.

The purpose of the `f_chmod` function is to change the attributes set against a file or directory.  A future version of MOS may include support for this API.

Returns:

- `A`: `23` (Not implemented)

### `0x9A`: ffs_utime

This API is not implemented, as the FatFS `f_utime` function is not supported by the current configuration of the FatFS library used in MOS.

The purpose of the `f_utime` function is to change the timestamp of a file or directory.  A future version of MOS may include support for this API.

Returns:

- `A`: `23` (Not implemented)

### `0x9B`: ffs_mkdir

Creates a new directory

Parameters:

- `HL(U)`: Pointer to a zero-terminated directory name string

Returns:

- `A`: `FRESULT`

### `0x9C`: ffs_chdir

Change the current working directory

It is strongly recommended to use the [`mos_cd` API](#0x03-mos_cd) API call instead of this one, as it will automatically resolve the path for you.  Using this API may result in MOS not understand the current working directory until a call to `mos_cd` is made or a version of the [`CD` command](./Star-Commands.md#cd) is run.

Parameters:

- `HL(U)`: Pointer to a zero-terminated directory name string

Returns:

- `A`: `FRESULT`

### `0x9D`: ffs_chdrive

This API is not implemented, as no Agon platform computer currently supports multiple drives.

### `0x9E`: ffs_getcwd

Get the current working directory (Available from MOS 2.2.0)

Parameters:

- `HL(U)`: Pointer to a buffer to store the directory path in
- `BC(U)`: Maximum length of the buffer

Returns:

- `A`: `FRESULT`

Preserves: `HL(U)`, `BC(U)`

### `0x9F`: ffs_mount

Mounts a volume/SD card

Whilst this API has documented parameters, as the current hardware configurations of Agon machines do not support multiple volumes or drives all of the parameters will be ignored.  They are documented here for completeness, and to allow for future expansion of the API in case a future Agon model is released that does support multiple SD cards.  For future compatibility you are advised to set all parameters to zero.

Parameters:

- `HL(U)`: Pointer to a blank FATFS `FATFS` structure (set to zero)
- `DE(U)`: Pointer to a zero-terminated volume path string (set to zero)
- `C`: Options byte (set to zero)

Returns:

- `A`: `FRESULT`

### `0xA0`: ffs_mkfs

This API is not implemented, as the FatFS `f_mkfs` function is not supported by the current configuration of the FatFS library used in MOS.

Returns:

- `A`: `23` (Not implemented)

### `0xA1`: ffs_fdisk

This API is not implemented, as the FatFS `f_fdisk` function is not supported by the current configuration of the FatFS library used in MOS.

Returns

- `A`: `23` (Not implemented)

### `0xA2`: ffs_getfree

Get free space information on a volume

Please note that this call does not directly return the number of free bytes on the volume, but instead returns the number of free clusters and the size of each cluster.  The number of free bytes can be calculated by multiplying these two values together.

Parameters:

- `HL(U)`: Pointer to a path string (ideally caller should set this to zero)
- `DE(U)`: Pointer to a block of memory to store number of free clusters, 32-bit value
- `BC(U)`: Pointer to a block of memory to store cluster size, 32-bit value

Returns:

- `A`: `FRESULT`

Whilst this API will let you pass in a pointer to a path in the `HL(U)` register, you are recommended to set this to `0`, as the way that MOS uses FatFS means that the path is not needed, and the call will likely fail if any path other than an empty string is provided.  This API call also differs slightly from the underlying `f_getfree` function which does not directly return the cluster size, but instead returns a pointer to the `FATFS` structure for the volume, which will contain the cluster size.  As the `FATFS` structure is sensitive to the FatFS configuration and may change in future versions of MOS, we have chosen to return the cluster size directly instead.

### `0xA3`: ffs_getlabel

Gets the label of a volume

- `HL(U)`: Pointer to a path string (ideally caller should set this to zero)
- `DE(U)`: Pointer to a buffer to store the label in (for safety and future proofing this should be 23 bytes long)
- `BC(U)`: Pointer to a block of memory to store the 32-bit volume serial number

Returns:

- `A`: `FRESULT`

As with the [`ffs_getfree` API](#0xa2-ffs_getfree) you should set the `HL(U)` parameter to `0`, as the way that MOS uses FatFS means that the path is not needed, and the call will likely fail if any path other than an empty string is provided.

It should be noted that this API call does not include a parameter for the size of the buffer to store the label in.  If your buffer is not large enough any memory after the label may be overwritten.  As of MOS 3.0 the maximum size of a volume label is 12 bytes (11 characters plus a zero terminator), however for future proofing you are advised to ensure your buffer is 23 bytes long.  The reason for this is that in the future we may enable support for discs using the exFAT filing system, which supports longer volume labels.

### `0xA4`: ffs_setlabel

Sets the label of a volume

Please note that the label string must be a valid FAT volume label, which basically means that it must be 11 characters or less.  Setting a label to an empty string will remove the label.

Owing to how filing systems work some labels may be rejected, and characters may be converted to upper-case.  The `f_setlabel` function that this API calls will attempt to look for a "drive prefix" in the label to specify a drive number, but owing to how we use FatFS in MOS this will not work.  As such you should not include a drive prefix in the label.

Parameters:

- `HL(U)`: Pointer to a zero-terminated volume label string

Returns:

- `A`: `FRESULT`

### `0xA5`: ffs_setcp

This API is not implemented, as the FatFS `f_setcp` function is not supported by the current configuration of the FatFS library used in MOS.

The purpose of `f_setcp` is to set the active code page for file paths, and this is not available as our FatFS configuration is restricted to a single code page.  Adding this feature would greatly expand the size of FatFS, and consequently also MOS, which would almost certainly exceed the available flash space on the eZ80.  As such it is highly unlikely that this API will be implemented in the future.

Returns:

- `A`: `23` (Not implemented)

### `0xA6`: ffs_flseek_p

Move the read/write offset pointer in a file

This API can be used to expand the size of a file, although you should note that the file data in the expanded part will be undefined.

Whilst the [`ffs_flseek`](#0x84-ffs_flseek) API can essentially perform the same function as this API, this is the preferred API to use for moving the current read/write pointer offset within a file.  As it accepts a pointer to the 32-bit offset value, it is compatible with both Z80-mode and ADL-mode code.

Parameters:

- `HL(U)`: Pointer to a `FIL` structure
- `DE(U)`: Pointer to a 32-bit value for the desired new offset from the start of the file

Preserves: `HL(U)`, `BC(U)`

Returns:

- `A`: Status code

***

## Status Codes

Many MOS API calls will return a status code in the `A` register.  In general these follow a consistent set of result values, with a few exceptions which are documented against the individual API calls.  Programs and commands will also return a status code when they complete.  If you are using MOS 3, the status code value can be captured in a system variable if the command or program is run using the [`try` command](./Star-Commands.md#try).

API calls that are documented above as returning an `FRESULT` value are returning a status code from the FatFS library.  These directly equate to status codes 0-19 in the table below.

Status codes can be translated into a human-readable string using the [`mos_getError`](#0x0f-mos_geterror) API call.

The possible status codes are as follows:

| Code | Error message | Description |
| ---- | ------------------- | ----------- |
| 0    | OK | Call succeeded (NB this will not be displayed when running a program or star command) |
| 1	   | Error accessing SD card | An error has occurred when trying to access the SD card |
| 2    | Internal error | An internal error has occurred.  Generally, if possible, a more specific error will be used |
| 3    | SD card failure | The SD card has failed |
| 4    | Could not find file | The file could not be found |
| 5    | Could not find path | The path could not be found |
| 6    | Invalid path name | The path name is invalid |
| 7    | Access denied or directory full | A file could not be saved to a directory either because of an access error or too many files in the directory |
| 8    | Access denied | A file or directory cannot be accessed because it is not readable |
| 9    | Invalid file/directory object | This is an internal filing system error |
| 10   | SD card is write protected | The SD card cannot be written to as it is write protected |
| 11   | Logical drive number is invalid | This is an internal filing system error, which should not occur |
| 12   | Volume has no work area | This is an internal filing system error |
| 13   | No valid FAT volume | The SD card does not contain a FAT format volume your Agon can understand |
| 14   | Error occurred during mkfs | This is an internal filing system error |
| 15   | Volume timeout | A problem has occured attempting to access your SD card |
| 16   | Volume locked | The FAT volume cannot be written to |
| 17   | LFN working buffer could not be allocated | This is an internal filing system error |
| 18   | Too many open files | Occurs when too many separate files have been opened |
| 19   | Invalid parameter | A parameter for the command or API is incorrect |
| 20   | Invalid command | The command was not recognised or found on disc |
| 21   | Invalid executable | An executable program does not have a valid header |
| 22   | Out of memory | Either MOS has run out of internal memory, or a buffer provided to an API was not large enough |
| 23   | Not implemented | The API call is not implemented in this version of MOS |
| 24   | Load overlaps system area | File load prevented to stop overlapping system memory |
| 25   | Bad string | A bad or incomplete string has been encountered |
| 26   | Too deep | Too many nested commands have been detected<br>This is usually caused by a faulty [alias](./System-Variables.md#command-aliases) definition including [file load/run types](./System-Variables.md#file-type-variables) |

Please note that Quark MOS 1.04 will only return status codes 0-21.  The MOS 2.x release series added status codes 22-25, and MOS 3.0 added status code 26.  

## System State Information (SysVars) {#sysvars}

The MOS API command [mos_sysvars](#0x08-mos_sysvars) returns a pointer to the base of the MOS SysVars (system state variables/information) area in IXU as a 24-bit pointer.  These are different from [System Variables](./System-Variables.md) which can be used in commands and scripts, so these these internal MOS system state variables are often simply referred to as sysvars.

The following sysvars are available in [mos_api.inc](#rst):

```
; SysVars (System State Information) indexes for api_sysvars
; Index into _sysvars in globals.asm
;
sysvar_time:			EQU	00h	; 4: Clock timer in centiseconds (incremented by 2 every VBLANK)
sysvar_vpd_pflags:		EQU	04h	; 1: Flags to indicate completion of VDP commands
sysvar_keyascii:		EQU	05h	; 1: ASCII keycode, or 0 if no key is pressed
sysvar_keymods:			EQU	06h	; 1: Keycode modifiers
sysvar_cursorX:			EQU	07h	; 1: Cursor X position
sysvar_cursorY:			EQU	08h	; 1: Cursor Y position
sysvar_scrchar:			EQU	09h	; 1: Character read from screen
sysvar_scrpixel:		EQU	0Ah	; 3: Pixel data read from screen (R,B,G)
sysvar_audioChannel:	EQU	0Dh	; 1: Audio channel
sysvar_audioSuccess:	EQU	0Eh	; 1: Audio channel note queued (0 = no, 1 = yes)
sysvar_scrWidth:		EQU	0Fh	; 2: Screen width in pixels
sysvar_scrHeight:		EQU	11h	; 2: Screen height in pixels
sysvar_scrCols:			EQU	13h	; 1: Screen columns in characters
sysvar_scrRows:			EQU	14h	; 1: Screen rows in characters
sysvar_scrColours:		EQU	15h	; 1: Number of colours displayed
sysvar_scrpixelIndex:	EQU	16h	; 1: Index of pixel data read from screen
sysvar_vkeycode:		EQU	17h	; 1: Virtual key code from FabGL
sysvar_vkeydown			EQU	18h	; 1: Virtual key state from FabGL (0=up, 1=down)
sysvar_vkeycount:		EQU	19h	; 1: Incremented every time a key packet is received
sysvar_rtc:				EQU	1Ah	; 8: Real time clock data
sysvar_keydelay:		EQU	22h	; 2: Keyboard repeat delay
sysvar_keyrate:			EQU	24h	; 2: Keyboard repeat rate
sysvar_keyled:			EQU	26h	; 1: Keyboard LED status
sysvar_scrMode:			EQU	27h	; 1: Screen mode (from MOS 1.04)
sysvar_rtc_enable:		EQU 28h ; 1: RTC enable status (from MOS 2.0.0)
sysvar_mouseX:			EQU 29h ; 2: Mouse X position
sysvar_mouseY:			EQU 2Bh ; 2: Mouse Y position
sysvar_mouseButtons:	EQU 2Dh ; 1: Mouse left+right+middle buttons (bits 0-2, 0=up, 1=down)
sysvar_mouseWheel:		EQU 2Eh ; 1: Mouse wheel delta
sysvar_mouseXDelta:		EQU 2Fh ; 2: Mouse X delta
sysvar_mouseYDelta:		EQU 31h ; 2: Mouse Y delta
```

Example: Reading a virtual keycode in ADL mode (24-bit):
```
		MOSCALL	mos_getkey
		LD	A, (IX + sysvar_vkeycode)	; Load A with the virtual keycode from FabGL
```

Example: Reading a virtual keycode in Z80 mode (16-bit):
```
		MOSCALL	mos_getkey
		LD.LIL	A, (IX + sysvar_vkeycode)	; Load A with the virtual keycode from FabGL
```

### VDP pflags (vdpflags)
The bits to manipulate the flags for `sysvar_vpd_pflags` are defined in the MOS `equs.inc`. They names are relatively self documenting, describing the category of update each is related to. The bits assigned are defined as follows:
```
VDPP_FLAG_CURSOR:	EQU		00000001b
VDPP_FLAG_SCRCHAR:	EQU		00000010b
VDPP_FLAG_POINT:	EQU		00000100b
VDPP_FLAG_AUDIO:	EQU		00001000b	
VDPP_FLAG_MODE:		EQU		00010000b
VDPP_FLAG_RTC:		EQU		00100000b
VDPP_FLAG_MOUSE:	EQU		01000000b
```

### Real Time Clock {#sysvar_rtc}

For efficiency, the real-time clock data in the sysvars is stored in a packed format, using subsets of bits within the 8 bytes of the `sysvar_rtc` data.  An API is provided from MOS 3 onwards to allow for this to be unpacked [mos_unpackrtc](#0x23-mos_unpackrtc) into a buffer in a friendlier, more easy to use format.  MOS uses the following C function to unpack the RTC data into a `vdp_time_t` object:
```c
void rtc_unpack(UINT8 * sysvar_rtc, vdp_time_t * t) {
	UINT32	d = *(UINT32 *)sysvar_rtc;

	t->month =		(d & 0x0000000F);		// uint32_t month : 4; 		00000000 00000000 00000000 0000xxxx : 00 00 00 0F >> 0
	t->day =		(d & 0x000001F0) >> 4;	// uint32_t day : 5;		00000000 00000000 0000000x xxxx0000 : 00 00 01 F0 >> 4
	t->dayOfWeek = 	(d & 0x00000E00) >> 9;	// uint32_t dayOfWeek : 3;	00000000 00000000 0000xxx0 00000000 : 00 00 0E 00 >> 9
	t->dayOfYear = 	(d & 0x001FF000) >> 12;	// uint32_t dayOfYear : 9;	00000000 000xxxxx xxxx0000 00000000 : 00 1F F0 00 >> 12
	t->hour = 		(d & 0x03E00000) >> 21;	// uint32_t hour : 5;		000000xx xxx00000 00000000 00000000 : 03 E0 00 00 >> 21
	t->minute =		(d & 0xFC000000) >> 26;	// uint32_t minute : 6;		xxxxxx00 00000000 00000000 00000000 : FC 00 00 00 >> 26

	t->second = sysvar_rtc[4];
	t->year = (char)sysvar_rtc[5] + 1980;
}
```

This real-time clock data is also available to programs, scripts, and the command line via [system variables](./System-Variables.md#time-and-date).
