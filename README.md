# Zemu Client - Binary Tracing in the Cloud

This is the *preview* of the Zemu cloud client for generating dynamic binary traces. It provides `strace`-like functionality for Linux/IoT binaries, as well as an instruction-level `overlay` for execution visualization in IDA Pro.

## How it Works

Zemu is a lightweight, high-speed, emulator for user space binaries. It has a similar use-case to QEMU user-mode emulation. However, whereas QEMU relies on native host syscall execution, Zemu emulates the syscalls as well. This gives Zemu the flexibility to emulate binaries designed to run multiple different platforms, regardless of the host system, as well as the ability to sandbox and modify exection on-the-fly. At the same time, it does not require full system virtualization, and thus scales well in terms of memory and configuration overhead.

This client script submits binaries to an *Azure Function* running Zemu, which automatically scales-out based on the total demand.

## Current Support Status

Zemu can run Linux and Windows binaries. Statically-linked malware binaries are our primary focus and the most well-tested.

|         | x86     | x86-64  | ARM 32-bit | ARM 64-bit | MIPS |
|---|---|---|---|---|---|
| Windows (PE) | &#9745; | &#9744; | &#9744; | &#9744; | &#9744; |
| Linux (ELF)   | &#9745; | &#9745; | &#9745; | &#9744; | &#9744; |

### Limitations

* There is a 30 second time limit.
* Network requests are not allowed. Empty data is returned rather than a hard fail.
* Around 25% of the most used Linux syscalls are implemented. Mileage may vary with other syscalls.

### Roadmap

* Continue to add more syscalls.
* Improved architecture support.
* A limited number of *environments* are supported, which are automatically selected based on the binary. We plan to add more environments and make them selectable. This is particularly important for *dynamically-linked* binaries.
* Incorporation of concolic execution, using fuzzing and symbolic exection, to explore malware binaries in-depth.


## Setup

### Requesting an API Key

API keys are currently by invite-only (as of 10/02/2019). We will later release a portal to provide any vetted security researcher access to the API.

### Client

*The client runs on any platform supporting Python.*

Paste your API key into `zemu.py`. For example, replace:

```
API_KEY = os.environ.get(
    'ZEMU_API_KEY', 'PASTE_YOUR_API_KEY_HERE')
```

with your API key:

```
API_KEY = os.environ.get(
    'ZEMU_API_KEY', 'c929270f53a0c27162659c932746fbe3')
```

*Note: By default, API keys limit the submission rate to 1 analysis per second.*

### IDA Pro Plugin

*The plugin requires IDA Pro 7.0 or above.*

To install the plugin, copy `ida-plugin/zemu_ida.py` into the IDA Pro `plugins` directory. In Windows, for example, this directory is located at `C:\Program Files\IDA 7.0\plugins`. In Linux, the path is `~/.idapro/plugins`. Once installed, the plugin is available under the *View->Load Zemu Overlay...* menu.

## Usage

Python is required. Example usage and output:

### System Call Tracing

The `strace` functionality provides output similar to the Linux utility. The first column indicates the calling thread (e.g. `main`, `thread1`, etc.), the second column indicates the return address post-syscall, followed by the `syscall` itself with arguments and return value.

```
$ python zemu.py strace testdata/static_elf_arm_helloworld

Zemu Copyright (c) 2019 Zeropoint Dynamics, LLC

Submitting.................done.
Queued. Awaiting results...done.

[main] [0x4b7ec] brk ( addr=0x0 ) -> 90000048
[main] [0x4b7ec] brk ( addr=0x90000d50 ) -> 90000d50
[main] [0x108d8] set_tls ( env=0x90000510 ) -> void
[main] [0x4b0ec] uname ( buf=0xff08fbe0 ) -> 0
[main] [0x55214] readlink ( pathname=0x719f8 ("/proc/self/exe"), buf=0xff08ed20, bufsiz=0x1000 ) -> 18
[main] [0x4b7ec] brk ( addr=0x90021d50 ) -> 90021d50
[main] [0x4b7ec] brk ( addr=0x90022000 ) -> 90022000
[main] [0x531fc] access ( pathname=0x71620 ("/etc/ld.so.nohwcap"), mode=0x0 ) -> -1
[main] [0x284ec] fstat64 ( fd=0x1, statbuf=0xff08fc80 ) -> 0
[main] [0x28820] write ( fd=0x1, buf=0x60b60 ("hello world!"), count=0xc ) -> c
[main] [0x28820] write ( fd=0x1, buf=0x90000fa8 ("
"), count=0x1 ) -> 1
[main] [0x27cd4] exit_group ( status=0x0 ) -> void
```

### Instruction-level Overlays in IDA Pro

The `overlay` functionality provides JSON output containing information about the functions and instructions executed, as well as register and memory values at each instruction. First, generate the overlay file:

```
$ python zemu.py overlay testdata/static_elf_arm_helloworld > static_elf_arm_helloworld.overlay

Zemu Copyright (c) 2019 Zeropoint Dynamics, LLC

Submitting.................done.
Queued. Awaiting results...done.
```

Then, open the binary in IDA Pro and select *View->Load Zemu Overlay...* from the menu:

![Load Zemu Overlay](https://github.com/zeropointdynamics/zemu-client/raw/master/images/ida-plugin-load-overlay.png)

Select the overlay you just generated and press *Open*. The instructions executed in the run will now be highlighted, with comments added indicating register and memory values at the time of instruction execution (note that these values are from the *first* execution of the instruction). Functions that were executed will be now be prefixed with `zmu_`.

![Zemu Overlay](https://github.com/zeropointdynamics/zemu-client/raw/master/images/ida-plugin-overlay.png)


### More Examples

There are additional samples in the [examples](https://github.com/zeropointdynamics/zemu-client/tree/master/examples) directory, generated from real malware interrogations.
