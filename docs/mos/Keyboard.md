# Keyboard Input

There are several different methods of reading the keyboard from within your code, each with advantages and disadvantages, so the method selected will depend on your specific need at that point in your code. You don't have to stick with one method, you can choose the best for a particular task.

## Method 1 - mos_getkey

This call will wait until a key has been pressed, and return the ascii key code of the key which was pressed.

This method should be used with caution. Calling this method will wait indefinitely until a key is pressed, with no other means of escape, apart from a system reset.

In assembler, an example might be:

```
ld a, $00         ; put $00 into A
rst.lil $08       ; make a MOS call with command $00 (mos_getkey).
                  ; system will wait until a key is pressed, then...
                  ; A now contains the ascii code of the pressed key
```

The call has not been implemented in AgonDev C, as the same result could be achieved with the next method, using a loop if result is 0.


## Method 2 - Single Key Checks

The MOS API command `mos_sysvars` returns a pointer to the base of the MOS SysVars (system state variables/information) area in IXU as a 24-bit pointer. 

There are three useful keyboard bytes store in this area - `sysvar_keyascii`, `sysvar_keymods` and `sysvar_vkeydown`.

### sysvar_keyascii

The byte at offset $05 after IXU provides an ascii code of the current key being pressed (or most recent if several are pressed), or 0 if no key is pressed.

This is useful to check for a single key press. E.g., in a game menu where there may be multiple options, but only a single decision is needed to move on.

In assembler, an example might be:

```
ld a, $08         ; put $08 into A
rst.lil $08       ; make a MOS call with command $08 (mos_sysvars).
                  ; IXU is now loaded with the base address
ld a, (ix + $05)  ; A is loaded with the byte at offset +$05 from the base address
                  ; A now contains the ascii code of the pressed key, or 0 if no key
```

In AgonDev C, the following example can be used:

```
#include "agon/vdp.h"

...

uint8_t theKey = vdp_getKeyCode();      // put ascii code, or 0, into theKey
```

### sysvar_keymods

It is also possible to check the status of the modifier keys (SHIFT, CTRL, etc).

The byte at offset $06 after IXU provides a bit code of the modifier keys which are pressed.

```
ld a, $08         ; put $08 into A
rst.lil $08       ; make a MOS call with command $08 (mos_sysvars).
                  ; IXU is now loaded with the base address
ld a, (ix + $06)  ; A is loaded with the byte at offset +$06 from the base address
                  ; A now contains a bit pattern of any modifier keys pressed
```

The following bits represent the given modifier keys:

| Bit |  Hex |Modifier |
| :---: | :---: | :---: |
| 0     | $01 | CTRL |
| 1     | $02 | SHIFT |
| 2     | $04 | ALT L |
| 3     | $08 | ALT R |
| 4     | $10 | CAPS |
| 5     | $20 | |
| 6     | $40 | |
| 7     | $80 | WINDOWS |

### sysvar_vkeydown

You can also do a simple test to see if any of the _keys_ are pressed.

The byte at offset $18 (sysvar_vkeydown) after IXU provides an indication if there are any keys pressed.

```
ld a, $08         ; put $08 into A
rst.lil $08       ; make a MOS call with command $08 (mos_sysvars).
                  ; IXU is now loaded with the base address
ld a, (ix + $18)  ; A is loaded with the byte at offset +$18 from the base address
                  ; A now contains 1 if any key is pressed, or 0 if none are pressed
```

In AgonDev C there are also two useful functions to aid the programmer which utilise this MOS call, which are pretty obvious what they do. One will wait until _any_ key is pressed down, the other will wait until there are no keys pressed (all keys up):

```
vdp_waitKeyDown();           
vdp_waitKeyUp();
```

These are useful for that _press any key to continue_ scenario.

## Method  3 - mos_getkbmap

This is probably the most complex method, but also the most comprehensive and flexible. 

The MOS API command `mos_getkbmap` returns a pointer to the base address of the MOS _virtual keyboard map_ in IXU as a 24-bit pointer. 

The keyboard map is an array of 16 bytes, where each bit within those bytes contains the current status of each key on the keyboard, bit = 1 for pressed, bit = 0 for not pressed.

To find out if any key (including modifer keys) is pressed, read the correct byte with the offset after IXU and then check the specific bit for its status.

This method is useful to check for a multiple key presses. E.g., In a game where multiple directions are possible (up and right), or movement plus a fire button need to be detected at the same time. 

This can also be used to check for less common combinations that would not return a standard ascii character, such as pressing the LEFT ALT and SPACE at the same time to perform a special function.

In assembler, an example might be:

```
ld a, $1E             ; put $1E into A
rst.lil $08           ; make a MOS call with command $1E (mos_getkbmap).
                      ; IXU is now loaded with the base address of the keyboard map
ld a, (ix + $0C)      ; A is loaded with the byte at offset +$0C from the base address
                      ; A now contains the status of 8 differnt keys
bit 2, A              ; The Z flag register now determines whether the SPACE key (bit 2) is pressed
jp nz, SPACE_PRESSED  ; do something as a result of key status
```

In AgonDev C, the following example can be used:

```
#include "agon/vdp.h"
#define GETBIT(var, bit)	(( var & (1 << (bit) ) ) ? 1 : 0 )

...

uint8_t keyRead = vdp_getKeyMap($0C);   // read the key map from offset +$0C
if (GETBIT(keyRead, 2)) {               // SPACE key
  // do some code here
}
```

    
The following chart lists which key is defined for each _bit_ within each _byte_ offset from $00 to $0F:


| IX+\Bit |   7    |   6    |     5     |     4     |    3     |    2     |     1     |     0     |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| $00     | CTRL R | SHIFT R| ALT L     | CTRL L    | SHIFT L  |          |           | ALT R     |
| $01     |        |        |           |           |          |          |           |           |
| $02     | -      | F7     | 8         | F4        | 5        | 4        | 3         | q         |
| $03     | Scroll | F10    | F12       | F11       | 7 (pad)  | 6 (pad)  |           | ⇐         |
| $04     | 0      | 9      | I         | 7         | T        | E        | W         | PRT SCR   |
| $05     | BK SPC | ` ~    |           | 9 (pad)   | 8 (pad)  |          |           | ⇓         |
| $06     | P      | O      | U         | 6         | R        | D        | 2         | 1         |
| $07     | PageUp | Home   | Insert    | Enter(pad)| - (pad)  | + (pad)  | ⇧         | [         |
| $08     | ‘(@)   | K      | J         | Y         | F        | X        | A         | CAPS LK   |
| $09     | PageDn | NUM LK | ./del(pad)| / (pad)   |          |          |           | ENTER     |
| $0A     | ;      | L      | N         | H         | G        | C        | S         |           |
| $0B     |        | - (+)  |           | *(pad)    |          |          |           | DELETE    |
| $0C     | . >    | , <    | M         | B         | V        | SPACE    | Z         | TAB       |
| $0D     |        |        | 3(pad)    | 1(pad)    | 0(pad)   | End      | / ?       |           |
| $0E     | F9     | F8     | F6        | F5        | F3       | F2       | F1        | ESC       |
| $0F     | WIN R  | WIN L  | 2(pad)    | 5(pad)    | 4(pad)   |          |           | ⇨         |

Keys located on an extended keyboard number pad area are indicated with (pad).

NOTE: There are a few gaps, so there may be more keys as not every keyboard has been tested.


## Method  4 - mos_editline

There may be times when you want a user to enter some text, or even just a number. The MOS API provides a useful method of allowing the user to type in as string of text without the programmer having to deal with every key press.
The programmer needs to define a buffer of bytes where the typed in string will be stored and then invoke the `mos_editline` command. Note that the buffer needs to allow an extra byte for a $00 terminator. So, a 32 byte buffer will be 31 string characters, plus the $00 terminator.

When the call has been completed, the A register will contain the character used to exit.
If user pressed ENTER, then it will be 13. 
If user pressed ESC, then it will be 27. This can used used as a check for _cancel_.

In assembler, an example might be:

```
ld a, $09         ; put $09 into A
ld hl, myBuffer   ; HL is where the string data will be stored once entered
ld bc, 32         ; BC is the max length of string to be captured
ld e, 1           ; E contains flags. 1 = buffer will be cleared before use
rst.lil $08       ; make a MOS call with command $09 (mos_editline).
                  ; the data will now be stored at address starting _myBuffer_
                  ; The A regster will contain the charater used to exit, ESC or ENTER

myBuffer:
    .ds 32        ; define 32 bytes of space for the buffer
```


In AgonDev C, the function `uint8_t exitCode = mos_editline(buffer, length, flags);` can be used.





