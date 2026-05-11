# System Variables

System variables are global variables that store various settings relating to different parts of the current operating system environment.  MOS support a few different types of system variables, and they are used by the system in several different ways.

Applications/programs are welcome to use system variables for their own needs.  MOS provides two API calls to allow for variables to be read and written by application code, and the command line also has several commands that will make use of system variables.

System variables are a new feature added to MOS 3.0.


## Type of System Variables

There are a few different data types that system variables can be.  The following are the types of system variables that are supported by MOS:

| Type | Description | How to set | How to remove |
|------|-------------|------------|--------------|
| String | A string of characters | [`*Set`](./Star-Commands.md#set) | [`*Unset`](./Star-Commands.md#unset) |
| Integer | Three-byte integers | [`*SetEval`](./Star-Commands.md#seteval) | [`*Unset`](./Star-Commands.md#unset) |
| Macros | Macros are strings that can be expanded, passed through the GSTrans mechanism, when they are read | [`*SetMacro`](./Star-Commands.md#setmacro) | [`*Unset`](./Star-Commands.md#unset) |
| Code | A machine code routine will be called when the variable is read or set. This allows the OS to provide some more sophisticated behaviour, such as access to real-time clock data | n/a | n/a |

The GSTrans mechanism is used by the command line in various ways.  For instance, when setting a system variable using the [`*Set`](./Star-Commands.md#set) command, the string value in the command line will first be passed through the GSTrans mechanism, allowing for variables to be expanded in the string before the variable is set.  The [`*Echo`](./Star-Commands.md#echo) command also uses the GSTrans mechanism, allowing for variables to be expanded in the string that is echoed to the screen.


## Naming of system variables

Users are free to name their system variables as they wish, but there are some limitations to bear in mind.

- Using wholly numeric names is not recommended, as this can cause difficulties with GSTrans operations when looking up a variable.  For instance, GSTrans will always interpret `<123>` to mean the ASCII character with the value 123, rather than looking up the name as a variable.
- When setting a variable, the case of a name is preserved, however variable lookup is not case-sensitive.  This means that `MyVar` and `myvar` are considered to be the same variable.
- Names can contain essentially any non-space, or non-control character
- When looking up variables, wildcard characters `#` and `*` can be used.  `#` will match any single character, and `*` will match any number of characters.

By default, MOS's system variables follow a naming convention where the variable is named loosely in the format `Class$VariableName`.  You do not have to follow this convention, but following it can help to keep your variables organised.


## Path variables

Variables that are named in the format `Name$Path` are considered to be path variables.  When loading or running a file, MOS will supports prefixing filenames with a path and a colon.  Let's say we have a variable named `Foo$Path`, which is set to `/home/foo/`.  If you try to load a file named `Foo:Bar`, the system will look for the file `/home/foo/Bar`.

Path variables can contain multiple paths to search, separated by commas.  For instance, if `Foo$Path` is set to `/home/foo/,/home/bar/`, the system will look for `Foo:Bar` in `/home/foo/Bar` and `/home/bar/Bar`, opening the first it finds.  If the file is not found in any of the paths and the file was opened for creation/update, the system will create the file in the first path in the list.

It is important to note that directories used in path variables should end with a `/` character.  If this is omitted then the system will not be able to find files in that directory.  For example, if `Foo$Path` is set to `/home/foo`, then the system will look for `Foo:Bar` in `/home/fooBar`, which is unlikely to be correct.

### System Path variables: `Run$Path` and `Moslet$Path` {#system-path-variables}

The MOS system makes use of two path variables, which it will set to default values on startup.  These are `Run$Path` and `Moslet$Path`.

The `Moslet$Path` variable is used to identify the directory where "moslet" programs are located.  By default this will be set to `/mos/`.  Moslets differ from normal programs in that they are written to be loaded and run from a different memory location, which should usually allow them to be used whilst a different program is running.  Their purpose is essentially to allow for new star commands to be added to MOS.  When you run a program, the system will use this variable to check to see whether the program you are trying to run is a moslet, and if it is, it will load the moslet and run it in the moslet memory space.

The `Run$Path` variable used to identify directories in which the command line should search when trying to run a program.  By default it is a macro variable set to `<Moslet$Path>, ./, /bin/`, which provides the same default search path for running programs as MOS 2.

These variables can be redefined to allow you to change where the system looks for moslets and programs to run.


## Application variables

By convention, application variables are named in the format `App$VariableName` where `App` is the name of the application that the variable is associated with.  This helps prevent clashes between variables that serve similar purposes for different applications.

If an application is started up using an `obey` file, then often the obey file will set a variable named in the format `App$Dir` or `App$Path` to represent the directory the program is located in, which may be used by the program to locate its resources.  Typically this is achieved with a command such as `set App$Dir <Obey$Dir>`.


## Command aliases

If you set up a system variable in the format `Alias$Name` then this will set up a new command alias that can be used from the command line with the given name.  For example, one could add a new `Mode` command using `*set Alias$Mode vdu 22 %0`.  This would allow you to run the command `mode 1` to change the screen mode to mode 1.  The command will be expanded to `vdu 22 1` when the command is run.

When attempting to interpret a command, the MOS command interpreter checks for aliases first.  As with regular commands the matching of the command name is not case-sensitive, and the command can be abbreviated with a `.` character.  So if you have defined the `Mode` command alias above, and no other aliases beginning with `m`, then `*m. 1` will be interpreted as `*mode 1`.  Only if no matching alias is found will the system look for a matching in-built command.

Alias resolution can be disabled by prefixing a command with the [`%`](./Star-Commands.md#skip-aliases) character.  So `*%m.` will skip alias resolution and run the first in-built command that matches `m.`, which in MOS 3.0 would be the [`Mem`](./Star-Commands.md#mem) command.  Disabling alias resolution can be useful in script files to ensure you're running the command you expect, and not an alias that may have been defined by the user.

Aliases support [argument substitution](Argument-Substitution.md).  Any unused arguments (beyond the last used argument) will be automatically appended to the end of the resultant command.

An alias can include multiple commands, separated by carriage return characters (`|M`).  If one command in an alias sequence fails, the rest of the commands will not be executed, and the error reported.

As an alias will append any unused arguments to the end of the command, the example above for a `mode` command is actually flawed.  Whilst `mode 1` would result in a `vdu 22 1` command, `mode 1 2 3 4` would result in the command `vdu 22 1 2 3 4` which is still technically a valid VDU command, but would send 3 extra bytes to the VDP, performing a `VDU 2`, `VDU 3` and `VDU 4` commands (enable printer, disable printer, and "write text at text cursor"), and is unlikely to be the intended behaviour.  Therefore a better version of the command would be `*set Alias$Mode vdu 22 %0|M#`.  This splits the alias into two separate commands, the first being a `vdu 22` passing in the first argument, and the second command will be a `#` with all other arguments following it.  The command interpreter considers commands beginning with a `#` to be a comment, and will ignore it.

### File load/run type aliases {#file-type-variables}

System variables are used to define how certain files should be loaded or run when using either the `LoadFile` or `RunFile` commands.  Additionally the MOS 3 CLI will use run type aliases to determine how to run a file if the user enters a filename without a preceding command.

Load and run type variables are a special type of [command alias](#command-aliases) named in the format `Alias$@LoadType_extension` or `Alias$@RunType_extension`, where the `extension` matches the filing system extension for that file type.  As with command aliases, these support [argument substitution](Argument-Substitution.md).  The system defines a few such aliases by default.

Attempting to load or run a file with an extension that does not have a corresponding alias (using `LoadFile` or `RunFile`) will result in an `Invalid command` error.

The system will set a variable named either `LastFile$Run` or `LastFile$Load` to the alias expansion that was used to run or load the file.  This can be useful for debugging purposes, as it allows you to see exactly what command (or sequence of commands) was executed when the file was loaded or run.

As load and run aliases are essentially a special type of command alias, they work in the same way as other aliases.  They can therefore also contain mutiple commands, separated by carriage return characters (`|M`), and will automatically append any unused arguments to the end of the expanded command string.

MOS 3.0 contains a few built in file load and run-types.  These are:

| Type variable | File extension | Definition | Description |
|---------------|----------------|------------|-------------|
| `Alias$@LoadType_bin` | `.bin` | `Load %*0` | Load a binary file |
| `Alias$@LoadType_obey` | `.obey` | `Type %*0` | Prints the contents of an obey file to the screen |
| `Alias$@RunType_bas` | `.bas` | `BBCBasic %*0` | Run a text-format BBC BASIC program * |
| `Alias$@RunType_bbc` | `.bbc` | `BBCBasic %*0` | Run a BBC BASIC program * |
| `Alias$@RunType_bin` | `.bin` | `RunBin %*0` | Run a binary file |
| `Alias$@RunType_exec` | `.exec` | `Exec %*0` | Run a script using the `*exec` command |
| `Alias$@RunType_obey` | `.obey` | `Obey %*0` | Run a script using the `*obey` command |

\* The aliases to run BBC BASIC assume that a `BBCBasic.bin` executable is present in your current [run path](#system-path-variables).  Often this is the plain Z80 version of BASIC.  If you wish to use the eZ80 version of BASIC, or BASIC V, then you can either rename their executable to `BBCBasic.bin` or redefine the `Alias$@RunType_bas` and `Alias$@RunType_bbc` variables to point to the correct executable.  For example, if you have a copy of BBC BASIC V in your current run path, with an executable named `BBCBasicV.bin`, you could use the following commands to redefine the aliases:

```
*set Alias$@RunType_bas BBCBasicV %*0
*set Alias$@RunType_bbc BBCBasicV %*0
```

It is common for [boot-up scripts](../MOS.md#boot-script) to set up load and run type variables to add support for other file types, or change the in-built behaviour.


## CLI

### `CLI$Prompt`

The command prompt that the MOS CLI displays is defined using the variable named `CLI$Prompt`.  By default this is a macro set to `<Current$Dir> *` to match the default prompt from MOS 2.2 onwards.  The prompt can be changed to anything you like, and if it is set as a macro it can include variables that will be expanded when the prompt is displayed.

If you unset the `CLI$Prompt` variable, the system will revert to displaying the default prompt which is `*`.

For example, to set your prompt to be written in yellow on black, returning to white on black for your commands, you can add the following line to your [boot script](../MOS.md#boot-script):

```
setmacro CLI$Prompt <17><128><17><11><Current$Dir> *<17><15>
```

Note that the above prompt change relies on being in default colours, in 16 or 64 colour modes. In 2 or 4 colour modes the prompt will appear to be white on black. It works by taking advantage of the GSTrans mechanism interpreting the `<123>` entries as literal characters to send. This means the the first part of the new prompt is the equivalent of `vdu 17 128 17 11`, and after the `*`, the ending is the equivalent of `vdu 17 15`.


### `CLI$AutoPaged` {#autopaged}

Setting a variable `CLI$AutoPaged` to any value will enable the auto-paging feature of the CLI, if you are using VDP 2.14.0 or later by [temporarily enabling paged mode](../vdp/System-Commands.md#vdu-23-0-9a).  By default this variable is not set.

When this variable is set then commands that can produce a lot of output will automatically use the VDP's "paged mode" for output.  This means that output will pause after a page of text has been displayed, and output will continue when the user presses the "shift" key.  This behaviour specifically applies to [`*Dir`](./Star-Commands.md#dir) (and its various aliases), [`*Help`](./Star-Commands.md#help), [`*Show`](./Star-Commands.md#show) and [`*Type`](./Star-Commands.md#type).  Other commands that could produce a lot of output such as running a program or a script explicitly do not use this feature so as to allow the user to control the output.

Auto-paging applies whether you are using the MOS command line, or executing MOS commands via some other means, such as using the [`mos_oscli`](./API.md#0x10-mos_oscli) API, or using the command from a BBC BASIC program.

Support for this variable was added in MOS 3.0.1.  In MOS 3.0.0 the system would always use auto-paging, and the feature could not be disabled.

To automatically enable paged mode for the CLI, you can add the following line to your [boot script](../MOS.md#boot-script):

```
*set CLI$AutoPaged 1
```


## Obey files: `Obey$Dir` {#obey-files}

When an obey file is run with the `obey` command, the system will set the variable `Obey$Dir` to the directory that the obey file is located in.  This can be used by the obey file to locate resources that it needs to run.

It should be noted that the `Obey$Dir` variable will not include a trailing `/` character, so if you are using it to build a path to a file, you will need to add the `/` character yourself.  For example, if you have an obey file located in `/home/foo/`, and wished from that obey file to run a program called `foo.bin` located in the same directory, your script should use the command `*RunBin <Obey$Dir>/foo.bin`.

## Time and date: `Sys$Date`, `Sys$Time` and `Sys$Year` {#time-and-date}

MOS provides three system variables that expose the current real-time clock information.  These are `Sys$Date`, `Sys$Time` and `Sys$Year`.  The date variable is in the format of `Day, n Mon`.  It is possible to set these variables, and the real-time clock should update accordingly.  The exception to that is setting `Sys$Date` if the value cannot be interpreted as a valid date.

The real-time clock information can also be displayed and updated using the [`*Time`](./Star-Commands.md#time) command


## Hotkeys: `Hotkey$<n>` {#hotkeys}

Hotkey definitions are stored as system variables in the format `Hotkey$n` where `n` is a number from 1-12 equating to the F1-F12 keys on your keyboard.  Hotkeys can also be defined using the [`*Hotkey`](./Star-Commands.md#hotkey) command.

Hotkey definitions support [argument substitution](Argument-Substitution.md).  Any unused arguments are discarded.

If a hotkey definition is set as a macro variable (using [`SetMacro`](./Star-Commands.md#setmacro)), then the system will expand the macro when the hotkey is pressed.

A hotkey definition that ends with a carriage return character (`|M`) will automatically press return when the hotkey is pressed.  When used in a CLI this means the command will execute immediately.  Hotkeys set using the [`*Hotkey`](./Star-Commands.md#hotkey) will automatically append a carriage return to the end of the command.

In MOS 3.0, whilst a hotkey variable can technically contain multiple lines, split by `|M`, the system will only use the first line of the variable, and the rest will be discarded.  This limitation is due to how the MOS line editor works.

If you wish to create a hotkey that can run a sequence of commands then there are two options.  Firstly you could create an obey file, and define a hotkey to run the obey file passing in all the arguments.  Alternatively you could define a command alias that can run a sequence of commands, and define a hotkey to use that alias.  Please note that both of these options are only suitable for running multiple MOS commands, and cannot be used for multi-line input in other environments such as the BBC BASIC command line.

Support for multiple commands in a hotkey may be added in a future version of MOS.


## System

As noted above, there are several "system" variables that MOS provides to give access to system information and control how MOS behaves.

Another two system variables are the `Console` variable and `Keyboard`.  These are two special write-only variables that will turn on or disable "console mode", and set the current keyboard layout respectively.  These variables are named to be compatible with earlier versions of MOS.


## Return codes

When a command or program finishes running MOS will set a numeric variable `Sys$ReturnCode` to indicate the success or failure of the program.  This variable is set to 0 if the program ran successfully, and to a non-zero value if the program reported an error.

The `Try` command can also be used to run a command and set a system variable `Try$ReturnCode` to the return code of that command.  If an error occurred then a variable `Try$Error` will also be set to the error message.  This can be useful in script files to check for errors and take appropriate action.  Running commands without using the `Try` command inside a script file (executed using either `exec` or `obey`) that fail will cause the script to stop running.  You can then use the `if` command to check the return code and take appropriate action.


