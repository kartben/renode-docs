# Using Python in Renode

Renode offers several different Python integrations which can be used to extend it, both from the inside and outside.
It is important to differentiate between them to avoid confusion:

* built-in IronPython integration (this is Python 2 running inside Renode) which is the main focus of this chapter
* external [pyrenode](https://github.com/antmicro/pyrenode) integration (this is a Python 3 wrapper using RPC calls to call Renode / execute Robot keywords)
* (NEW!) external [pyrenode3](https://github.com/antmicro/pyrenode3) integration (this is the new, native CLR bindings, esentially a Python 3 wrapper around all of Renode, set to replace the 'older' pyrenode when stabilized)

This chapter currently focuses on the built-in IronPython integration and use of Python from within Renode, hence the Python 2 syntax with old-style `print` statements you will notice throughout.
Eventually, the internal (Iron)Python integration will either be migrated to IronPython3 or moved over to the the same mechanism `pyrenode3` is using.

In the meantime, documentation will be added for `pyrenode3` to complement this chapter.

Thanks to the IronPython integration, you can use Python in runtime to do things like reacting to lines on UART, change of user state, peripheral access, or a value appearing in the memory.

Python offers a real programming language syntax with its flow control constructs, loops, etc.

Python also gives you access to emulation components available under context-specific variable names.
These variables are described in respective sections.
They serve as an API between the script and the rest of Renode.

## Direct Python execution in the Monitor

Python code can be executed directly in the Renode Monitor using the `python` command in the Monitor:

```
(monitor) python [ py ]
```

You can also use the `include` command to load an existing `.py` file:

```
(monitor) include @path/to/file.py
```

## Providing Python scripts from the Monitor

To provide a multiline Python script from the Monitor or a `.resc` file, use the following notation:

```
(monitor) set my_script """
> print "Hello"
> print "This is a multiline Python script"
> if 0 > 1:
>    print "This will not be printed"
> """
(monitor) python $my_script
```

Using the `print` Python command will output the text on the Monitor:

```
(monitor) python "print 'Hello'"
Hello
```

## Creating Monitor commands in Python

Running the `python` command in the Monitor executes the provided code immediately.
However, it is possible to use these Python scripts to define functions available for later use.

Such functions can be executed with subsequent `python` calls, but you can also expose them directly to the Monitor, if you follow a specific naming convention.

If the name of the function you define starts with the `mc_` prefix, it will become available as a new Monitor command, stripped of this prefix.

For example, consider the following Python definition:

```
def mc_sleep(time):
    sleep(float(time))
```

When executed in Renode, this definition provides a `sleep` function in the Monitor:

```
(monitor) sleep 5
```

A small set of predefined Python commands is available in the [monitor.py](https://github.com/renode/renode/blob/master/scripts/monitor.py) file.

## Python peripherals in a platform description

You can use Python in the `.repl` file of your platform description.
Several platforms included in Renode like [Zynq 7000](https://github.com/renode/renode/blob/master/platforms/cpus/zynq-7000.repl#L71-L124) use Python peripherals to implement simple logic.
The most common use of Python peripherals is mocking certain blocks that are not fully implemented but are required by the software.

To create a Python peripheral, you need to specify several variables:

```
variableName: Python.PythonPeripheral @ sysbus 0x7000F410
    size: 0x4
    initable: false
    script: "request.value = 0x60000"
```

To define Python peripherals, you can specify the following variables:

```{list-table} Request variable fields
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - size
  - Size of the peripheral as a hexadecimal value
* - initable
  - If `true` the peripheral can be initialized and executes code from the ``isInit`` section
* - script
  - Python script you want to execute
```

```{note}
You can also supply a path to the `.py` file if you use `filename:` instead of `script:`.
```

When programming a Python peripheral, you have access to the `request` variable, which describes the current transaction.

The variable `request` in Renode gives you access to the following fields:

```{list-table} Request variable fields
:header-rows: 1

* - Field name
  - Description
* - isInit
  - Is `true` during construction of the peripheral if it's marked as "initable"
* - isRead
  - Is `true` when the CPU is trying to read from a peripheral
* - isWrite
  - Is `true` when the CPU is trying to write to a peripheral
* - value
  - When `isWrite == true`, this is the value to be written, when `isRead == true` this is the return value
* - offset
  - Offset within the peripheral
* - type
  - Width of the access (8bit, 16bit, 32bit)
```

```{note}
The variable `isInit` is `true` when the peripheral is marked as possible to initialize and is currently being initialized. It can be useful for initializing the internal state of a peripheral.
```

An example Python peripheral available in mainline Renode is [repeater.py](https://github.com/renode/renode/blob/master/scripts/pydev/repeater.py), returning the same value that was written to it by software:

```
if request.isInit:
    lastVal = 0
elif request.isRead:
    request.value = lastVal
elif request.isWrite:
    lastVal = request.value

self.NoisyLog("%s on REPEATER at 0x%x, value 0x%x" % (str(request.type), request.offset, request.value))
```

There are also other variables that can be used in Python peripherals:

```{list-table} Other Python peripherals variables
:header-rows: 1
:widths: 25 75

* - Field name
  - Description
* - self
  - The peripheral itself
* - size
  - Size of the peripheral as a hexadecimal value
```

## Python hooks in Renode

There are multiple types of hooks in Renode, which allow you to execute code, e.g. a Python script, when certain specified conditions are met.
Each separate hook provides you with different functionality, and multiple types of hooks can be used in a single project.
Types of hooks in Renode:

- {ref}`UART hooks,<uart-hooks>`
- {ref}`CPU hooks.<cpu-hooks>`
- {ref}`system bus hooks,<system-bus-hooks>`
- {ref}`watchpoint hooks,<watchpoint-hooks>`
- {ref}`packet interception hooks,<packet-interception-hooks>`
- {ref}`user state hooks,<user-state-hooks>`

(uart-hooks)=

### UART hooks

UART hooks enable you to code a reaction to the event when a specific character or substring appears on UART.

UART hooks in Renode have access to the following variables:

```{list-table} UART hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - line
  - Current line matching the searched string
* - uart
  - UART object that produced the line
* - self
  - Alias for `uart`
```

You have to specify the string you want to search for in the UART's output and the Python script that will be executed if the substring appears on UART:

```
(machine) uart AddLineHook "searched value" "print 'Found the %s string' % line"
```

```{note}
You have to use a UART name that matches your use case.
```

(cpu-hooks)=

### CPU hooks

CPU hooks enable you to code a reaction to reaching a specific stage in code execution.
The hook can be triggered at the beginning, end of code of an executed block, or any specified point in the application.
CPU hooks can also react to the interrupts in code and be triggered at the beginning or an end of an interruption.

CPU hooks in Renode have access to the following variables:

```{list-table} CPU block hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - pc
  - Program counter of the current block
* - size
  - Size of the block as the number of instructions
* - machine
  - Current machine, allows you to access other components and peripherals
* - cpu
  - CPU executing the current block
* - self
  - Alias for `cpu`
```

```{list-table} CPU interrupt hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - machine
  - Current machine, allows you to access other components and peripherals
* - self
  - Alias for `cpu`
* - exceptionIndex
  - Exception index delivered to the CPU
```

To use a CPU hook, you need to name `cpu` you want to attach the hook to and specify the method.
To create a hook that triggers at the beginning of interrupt:

```
(machine) cpu AddHookAtInterruptBegin "print 'Interrupt has started'"
```

To create a hook that triggers at the end of interrupt:

```
(machine) cpu AddHookAtInterruptEnd "print 'Interrupt has ended'"
```

To create a hook that triggers at the beginning of an executed block:

```
(machine) cpu SetHookAtBlockBegin "print 'Execution of a code block has started'"
```

To create a hook that triggers at the end of an executed block:

```
(machine) cpu SetHookAtBlockEnd "print 'Execution of a code block has ended'"
```

You can also create a hook for any specified point in the application:

```
cpu AddHook 0x60000000 "print 'You have reached a hook'"
```

(system-bus-hooks)=

### System bus hooks

System bus hooks enable you to code a reaction to the event when peripherals are accessed to read or write.
This hook has access to the peripherals, and it can be assigned to react only to a specific peripheral.

System bus hooks in Renode have access to the following variables:

```{list-table} System bus hooks value to write variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - self
  - The target peripheral
* - sysbus
  - System bus of the current machine
* - machine
  - Current machine, allows you to access other components and peripherals
* - value
  - Value that will be written to the given `offset`
* - offset
  - An offset, to which the `value` will be written
```

```{list-table} System bus hooks value to read variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - self
  - The target peripheral
* - sysbus
  - System bus of the current machine
* - machine
  - Current machine, allows you to access other components and peripherals
* - value
  - Value read from the `offset`
* - offset
  - An offset, from which the `value` will be read
```

To create a system bus hook that executes a Python script after a specific peripheral is accessed to read:

```
(machine) sysbus SetHookAfterPeripheralRead peripheral "print '%s peripheral has been accessed to read'"
```

To create a system bus hook that executes a Python script before a specific peripheral is accessed to write:

```
(machine) sysbus SetHookBeforePeripheralWrite peripheral "print '%s peripheral has been accessed to write'"
```

(watchpoint-hooks)=

### Watchpoint hooks

Watchpoint hooks enable you to code a reaction to the event when a specific value appears in a specific memory address.

Watchpoint hooks in Renode have access to the following variables:

```{list-table} Watchpoint hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU issuing the memory access
* - address
  - Address of the access triggering the hook
* - width
  - Width of the access in bits: Byte = 1, Word = 2, DoubleWord = 4, QuadWord = 8
* - value
  - Value written to the address or read from it.
* - self
  - Sysbus of the current machine
```

To create a watchpoint hook that executes a Python script, when a `Read` access with `width` of 32 bits (DoubleWord) appears at `address` 0x70001000, run:

```
(machine) sysbus AddWatchpointHook 0x70001000 DoubleWord Read "print '32 bit value appeared at the address 0x70001000'"
```

```{note}
You can use either word (DoubleWord) or numeric (4) notation in `width`, as listed in the table above.
```

(packet-interception-hooks)=

### Packet interception hooks

Packet interception hooks enable you to code a reaction to the event when a packet appears on a radio medium.

Packet interception hooks in Renode have access to the following variables:

```{list-table} Packet interception hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - packet
  - Byte array with the contents of a packet
* - self
  - Radio that is about to receive the packet
* - machine
  - Your current machine
```

```{note}
Keep in mind that the hook is executed for the recipient, not the sender.
```

To have a functioning Packet interception hook, your machine needs to have a wireless medium.
To create a packet interception hook, run:

```
(machine) wireless SetPacketHookFromScript sysbus.radio "if packet[5] == 0x4: print("I'm interested in packets with 0x4 as their sixth byte")"
```

You can also execute a script from a file:

```
(machine) wireless SetPacketHookFromFile sysbus.radio @path/to/file.py
```

(user-state-hooks)=

### User state hooks

User state hooks enable you to code a reaction to the event when "UserState" string of a machine changes.

User state hooks in Renode have access to the following variables:

```{list-table} User states hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - state
  - A new string set as user state
* - self
  - A current machine
```

To create a user state hook that executes Python script when a specific `state` is set:

```
(machine) machine AddUserStateHook "state" "print 'User state has changed to: %s state '"
```

## RISC-V extensions

Renode has extensive support for RISC-V architecture and lets you code extensions using Python.
You can write the logic of Control/Status Register and custom instructions and then install them either from a file or a string.
Those instructions can be either 16-bit, 32-bit, or 64-bit.

Python implementations of RISC-V custom instructions have access to the following variables:

```{list-table} RISC-V custom instructions variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU that executes the instruction
* - machine
  - Current machine
* - state
  - Current user state of a CPU
* - instruction
  - Current opcode
```

```{note}
Variable `state` provides you with additional CPU state, mapping user-defined strings to arbitrary objects. It is useful when you want to share the state between different instructions.
```

Custom instructions from a string can be installed using:

```
(machine) sysbus.cpu InstallCustomInstructionHandlerFromString "10110011100011110000111110000010" "cpu.DebugLog('custom instruction executed!')"
```

If you want to install a custom instructions from a file, use:

```
(machine) sysbus.cpu InstallCustomInstructionHandlerFromFile "10110011100011110000111110000010" "path/to/file.py"
```

RISC-V custom instructions extensions in Renode have access to the following variables:

```{list-table} RISC-V custom CSR variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU that accesses the custom CSR
* - machine
  - Current machine
* - request
  - Structure of this variable is described below
```

Custom instructions have access to the `request` object, which is similar to bus access.
It has access to various properties:

```{list-table} The request structure
:header-rows: 1
:widths: 25 75

* - Property name
  - Description
* - CSR number
  - Number of the register as a hexadecimal value
* - Value
  - Read of written value
* - Type
  - Type of request: READ, WRITE, INIT
* - isInit
  - Is `true` during construction of the CPU if it's marked as "initable"
* - isRead
  - Is `true` when the CPU is trying to read from the register
* - isWrite
  - Is `true` when the CPU is trying to write to the register
```

To install custom Control/Status Register from a string, use:

```
(machine) sysbus.cpu RegisterCSRHandlerFromString 0xf0d "print 'CSR has been accessed'"
```

To provide a custom CSR from a file, instead use:

```
(machine) sysbus.cpu RegisterCSRHandlerFromString 0xf0d "path/to/file.py"
```

```{note}
You should adjust the CSR number, `0xf0d` in this example, according to your needs.
```

Examples of custom CSR and custom instructions can be found in [Renode's custom instructions tests](https://github.com/renode/renode/blob/master/tests/unit-tests/riscv-custom-instructions.robot#L178).
