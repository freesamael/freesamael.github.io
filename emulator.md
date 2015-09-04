# Study on Emulator
[toc]

## Prefix
The study focuses on Android's latest dev branch `studio-1.4-dev` implementation of the classic emulator, in combination of gonk layer and kernel used in `emulator-x86-kk`. Mozilla's customization is not a main focus in this document, but might still be referred.

> I say *classic* because there's another new emulator implementation for Android M or later at [qemu-android](https://android.googlesource.com/platform/external/qemu-android/).

## Emulator Basics

### Launch Emulator from AOSP / B2G Tree
Besides using AVD or passing everything through the command line arguments, emulator is capable of finding images from the built AOSP / B2G tree directly if you have proper environment variables set. Here's an example:
```bash
# In ${B2G} or ${AOSP}
$ export ANDROID_BUILD_TOP=${PWD}
$ export ANDROID_PRODUCT_OUT=${PWD}/out/target/product/generic_x86/
$ ./out/host/darwin-x86/bin/emulator -gpu on 
```

If the emulator doesn't boot, try deleting `${ANDROID_PRODUCT_OUT}/userdata-qemu.img` first. Note that `-gpu on` is necessary for B2G. 

To show kernel message, use `-show-kernel`. The message is redirected through the emulated system's `ttyS0` or `ttyGF0`.

For older emulator builds, qemu monitor can be redirected to stdio through `-qemu -monitor stdio`, but it's no longer support in newer code. See [console.c](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/android/console.c#L2640).

#### Use Writable System Image
By default, `system.img` and `userdata.img` are read only init images, while `system-qemu.img` and `userdata-qemu.img` are writable. See `emulator -help-disk-images` for more information.

In usual case, a writable image is created under `/tmp/android-xxx/`. To use a writable system image on boot directly, simply move `${ANDROID_PRODUCT_OUT}/system.img` to `${ANDROID_PRODUCT_OUT}/system-qemu.img`.

> The emulator version used in emulator-x86-kk has a bug and it complains `ko:Missing initial system image path!` when using `system-qemu.img`. It's fixed in newer versions.

## Build Goldfish Kernel 

### Special Setup for Mac OS X
Building qemu-kernel on Ubuntu Linux is quite straightforward, but on Mac OS X there are some tricks need to be done.

#### Copy elfutils headers:
```bash
cp ${B2G}/external/elfutils/host-darwin-fixup/* /usr/local/include/
cp ${B2G}/external/elfutils/libelf/elf.h /usr/local/include/
```

#### Create `/usr/local/include/linux/types.h`:
```c
#include <stdint.h>

#ifndef __u8
typedef uint8_t __u8;
#endif

#ifndef __u16
typedef uint16_t __u16;
#endif

#ifndef __u32
typedef uint32_t __u32;
#endif

#ifndef __u64
typedef uint64_t __u64;
#endif
```
#### Install GNU sed. 
Linux kernel makefiles are not compatible with BSD sed:
```bash
brew install gnu-sed
export PATH="/usr/local/opt/gnu-sed/libexec/gnubin:$PATH"
```

### Build the Kernel
This section applies to both Linux and Mac OS X.

Clean previous build with `make clean` or clean all configs (only necessary for the very first time) with `make mrproper`, then build the kernel (x86 as the example) with `build-kernel.sh`:
```bash
${B2G}/external/qemu/distrib/build-kernel.sh --arch=x86
# or ${AOSP}/prebuilts/qemu-kernel/build-kernel.sh --arch=x86
# omit --arch parameter to build arm kernel
```

The output should be at `/tmp/kernel-qemu` directory.

If the cross toolchain version differs from the default version (at the time this document is written, the default version is 4.8), pass `--gcc-version=<version>`. If you're using an older version of build script and it doesn't have the option, export the path to cross compiler directly. For example:
```
export PATH=${PATH}:${B2G}/prebuilts/gcc/darwin-x86/x86/i686-linux-android-4.7/bin/
```

#### Build the Kernel Manually
To build kernel manually, export necessary variables
```bash
export ARCH=x86 # or arm
export SUBARCH=x86 # or am
export REAL_CROSS_COMPILE=i686-linux-android-
# or x86_64-linux-android-
# or arm-linux-androideabi-
export CROSS_COMPILE=${B2G}/external/qemu/distrib/kernel-toolchain/android-kernel-toolchain-
# or CROSS_COMPILE=${AOSP}/prebuilts/qemu-kernel/kernel-toolchain/android-kernel-toolchain-
```

Then build the kernel with 
```bash
make goldfish_defconfig
make nconfig # optional, only if you want to change the config
make -j8
```

The output should be at `arch/arm/boot/zImage` or `arch/x86/boot/bzImage`. Copy and rename it to `kernel-qemu` before use.

## The Goldfish Virtual Board

### QEMUMachine Initialization
Each CPU-specific emulation engine (e.g. `emulator64-x86`) registers its emulated machines statically through `machine_init` statement. 

The macro generates a function to register a machine with gcc special attribute ` __attribute__((constructor))`, which conceptually resembles static constructor in modern languages, and is invoked when the program loads.

- [x86 machine registration](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/hw/i386/pc.c#L1332)
- [arm machine registration](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/hw/android/android_arm.c#L161)

Refer [machine_init](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/qemu/module.h#L34), [module_init](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/qemu/module.h#L18-L21), [register_module_init](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/util/module.c#L58-L69), and [QEMUMachine](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/hw/boards.h#L15-L23).

### Goldfish Device Registration
The core of a goldfish device registration is [goldfish_device_add](https://github.com/android/platform_external_qemu/blob/dc053e276a0484385c5f34d0b271e6704f0d4c3f/hw/android/goldfish/device.c#L99-L102):
- dev: the device description, refer [goldfish_device](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/hw/android/goldfish/device.h#L18-L29).
- mem_read: an array of callbacks for *byte*, *word* and *dword* memory read operations, respectively.
- mem_wrtie: an array of callbacks for *byte*, *word* and *dword* memory write operations, respectively.
- opaque: anything the caller wants to keep. It will be carried back to the mem_read / mem_write callback functions on invocation.
```c
int goldfish_device_add(struct goldfish_device *dev,
                       CPUReadMemoryFunc **mem_read,
                       CPUWriteMemoryFunc **mem_write,
                       void *opaque);
```
> Devices registered here are operated through memory-mapped I/O.

### Goldfish Platform Bus

#### Discoverability
For embedded systems, devices are not naturally discoverable, and the driver doesn't know which I/O port, memory address or interrupt it should use.

*Platform bus* is a special *platform device* to overcome this issue. Think about PCI.

> In the new qemu-android implementation for Android M and later, it no longer uses *platform bus* in favor of *device tree*. Refer the article [Platform devices and device trees](http://lwn.net/Articles/448502/) for more information about device trees.

#### Virtual Device Design
* Goldfish Platform Bus uses the MMIO address region 0xff001000 to 0xff801000. 
* 0xff001000 - 0xff001fff are reserved by `goldfish_device_bus` as a set of [32bit I/O registers](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/docs/GOLDFISH-VIRTUAL-HARDWARE.TXT#L73-L89) for bus operations.
* The bus device uses IRQ 1 on ARM with goldfish interrupt controller, and IRQ 4 on x86 with qemu's built-in virtual i8259.

#### Kernel Driver Implementation
[pdev_bus.c](https://github.com/mozilla-b2g/kernel_goldfish/blob/b2g-goldfish-3.4/arch/x86/mach-goldfish/pdev_bus.c) registers a *platform device* `goldfish_pdev_bus` at I/O port 0x1000, which plays the role of a *platform bus*.

The bus scan process is as following:
```sequence
kernel->pdev_bus: goldfish_init
pdev_bus->kernel: platform_device_register(&goldfish_pdev_bus_device)
kernel->pdev_bus: goldfish_pdev_bus_init
pdev_bus->kernel: platform_driver_register(&goldfish_pdev_bus_driver)

note right of kernel: found driver / device matches

kernel->pdev_bus: goldfish_pdev_bus_probe
pdev_bus->kernel: request_irq
kernel->pdev_bus: goldfish_pdev_bus_interrupt
pdev_bus->kernel: readl

note right of kernel: goes to the registered CPUReadMemoryFunc goldfish_bus_read

pdev_bus->pdev_bus: goldfish_new_pdev

note right of pdev_bus: all devices info loaded

pdev_bus->kernel: schedule_work(&pdev_bus_worker)
kernel->pdev_bus: goldfish_pdev_worker
pdev_bus->kernel: platform_device_register

note right of pdev_bus: loop until all devices registered
```

After the bus scan finishes, `goldfish_pdev_bus_device` can be removed through `platform_device_unregister`, although the driver didn't do so. 

Those platform devices registered still need corresponding drivers to work. For exmaple, the driver for *qemu_pipe* is at [qemu_pipe.c](https://github.com/mozilla-b2g/kernel_goldfish/blob/b2g-goldfish-3.4/drivers/misc/qemupipe/qemu_pipe.c).

#### /proc/ioports
/proc/ioports lists all registered PMIO. `goldfish_pdev_bus` is registered at I/O port 0x1000.
```text
0000-001f : dma1
0020-0021 : pic1
0040-0043 : timer0
0050-0053 : timer1
0060-0060 : keyboard
0064-0064 : keyboard
0070-0071 : rtc_cmos
  0070-0071 : rtc0
0080-008f : dma page reg
00a0-00a1 : pic2
00c0-00df : dma2
00f0-00ff : fpu
03c0-03df : vga+
0cf8-0cff : PCI conf1
1000-10ff : goldfish_pdev_bus
c000-c0ff : 0000:00:02.0
  c000-c01f : ne2k-pci
```
#### /proc/iomem
/proc/iomen lists all registered MMIO and real memory.

`goldfish_device_bus` is a special device which contains only the bus state for bus scanning only. It can be thought as the corresponding device implementation of `goldfish_pdev_bus_device`, and can also be removed after scan finishes.

```text
00000000-0000ffff : reserved
00010000-0009efff : System RAM
0009f000-0009ffff : reserved
000a0000-000bffff : Video RAM area
000c0000-000c8bff : Video ROM
000c9000-000c91ff : Adapter ROM
000e8000-000fffff : reserved
  000f0000-000fffff : System ROM
00100000-1ffeffff : System RAM
  00200000-0067f9ec : Kernel code
  0067f9ed-008906ff : Kernel data
  008dd000-00a3dfff : Kernel bss
1fff0000-1fffffff : ACPI Tables
ff001000-ff001fff : goldfish_device_bus
ff004000-ff004fff : goldfish_audio.0
ff005000-ff005fff : goldfish_mmc.0
ff010000-ff010fff : goldfish-battery.0
ff011000-ff011fff : goldfish_nand.0
ff012000-ff013fff : qemu_pipe
ff014000-ff014fff : goldfish_tty.0
ff015000-ff015fff : goldfish_tty.1
ff016000-ff016fff : goldfish_fb.0
ff017000-ff017fff : goldfish_events.0
fffc0000-ffffffff : reserved
```

### x86 Device Initialization
```sequence
vl_android->module: module_call_init(MODULE_INIT_MACHINE) 
module->pc: pc_machine_init
pc->vl_android: qemu_register_machine(pc_machine)
vl_android->pc_machine: init
pc_machine->pc: pc_init1
pc->device: goldfish_device_init
pc->device: goldfish_device_bus_init
device->device: goldfish_device_add
pc->pipe: pipe_dev_init
pipe->device: goldfish_device_add
```

Log printed at [goldfish_add_device_no_io](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/hw/android/goldfish/device.c#L86-L87), if enabled:
```
goldfish_device_bus_init: base ff001000, irq 4
goldfish_add_device: goldfish_device_bus, base ff001000 1000, irq 4 1
goldfish_add_device: goldfish-battery, base ff010000 1000, irq 5 1
goldfish_add_device: goldfish_nand, base ff011000 1000, irq 0 0
goldfish_add_device: qemu_pipe, base ff012000 2000, irq 6 1
goldfish_add_device: goldfish_mmc, base ff005000 1000, irq 7 1
goldfish_add_device: goldfish_tty, base ff014000 1000, irq 9 1
goldfish_add_device: goldfish_tty, base ff015000 1000, irq 10 1
goldfish_add_device: goldfish_fb, base ff016000 1000, irq 11 1
goldfish_add_device: goldfish_events, base ff017000 1000, irq 3 1
goldfish_add_device: goldfish_audio, base ff004000 1000, irq 14 1
```

### QEMU Pipe / Goldfish Pipe
QEMU Pipe is a special virtual device to provide very fast communication channel between the emulator and the emulated system. In the emulated system, it appears as either `/dev/qemu_pipe` or `/dev/goldfish_pipe`

#### QEMU Pipe Services
There are currently 4 types of services using qemu pipe, as described in the [document](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/docs/ANDROID-QEMU-PIPE.TXT#L288-L315). Services are registered through [goldfish_pipe_add_type](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/hw/android/goldfish/pipe.c#L83-L86):
- pipeName: an unique name of the pipe service.
- pipeOpaque: anything the caller wants to keep. It will be carried back to the handler functions.
- pipeFuncs: pipe handler functions. Refer [GoldfishPipeFuncs](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/hw/android/goldfish/pipe.h#L62-L119).
```c
void
goldfish_pipe_add_type(const char*               pipeName,
                       void*                     pipeOpaque,
                       const GoldfishPipeFuncs*  pipeFuncs );
```

##### Service Workflow
When a client in the emulated system tries to connect to a specific pipe service, `init` callback is invoked; while when the kernel is closing a pipe, `close` is invoked. Writing from a client redirects to `sendBuffers`, and reading redirects to `recvBuffers`. The basic workflow looks like this:
```sequence
note right of hw_pipe: client connects
hw_pipe->pipe_service: init
pipe_service-->hw_pipe: pipe

note right of hw_pipe: client writes
hw_pipe->pipe_service: sendBuffers(pipe, buffers)

note right of hw_pipe: client reads
hw_pipe->pipe_service: recvBuffers(pipe, buffers)

note right of hw_pipe: client closes pipe
hw_pipe->pipe_service: close(pipe)
```
>The service might not be able to consume more writes, or provide anything to read temporarily when a write / read request occurs. In this case, the service returns [PIPE_ERROR_AGAIN](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/include/hw/android/goldfish/pipe.h#L189). 
>
> The client can then choose to be signaled when writing / reading is possible. See [waiting / signaling](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/docs/ANDROID-QEMU-PIPE.TXT#L167-L229) sections.

#### Adaption of Legacy QEMUD
Before qemu pipe was implemented, communications between emulator and the emulated system were all through serial port with a multiplexing daemon [qemud](https://github.com/mozilla-b2g/device_generic_goldfish/blob/b2g-4.4.2_r1/qemud/qemud.c) which lives in the emulated system.

The daemon creates a socket `/dev/socket/qemud`, and client programs transmit data through the socket. The whole architecture looks like this:
```text
emulator <==serial==> qemud <---> /dev/socket/qemud <-+--> client1
                                                      |
                                                      +--> client2
```

The benefit of using qemud was to avoid writing device drivers for all services such as gsm, gps and other emulated sensors, but instead only needed to implement the multiplexing protocol clients.

The qemu pipe replaces the role and provides much better performance. However, to keep backward compatibility and to avoid too much change to the existing implementation, the [emulator side qemud](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/android/hw-qemud.c) is kept but it works as one of the qemu pipe services now. The serial port channel no longer in use, unless you're running a very old version of Android on the emulator, such as Gingerbread.

For emulated systems since ICS, both [qemud daemon](https://github.com/mozilla-b2g/device_generic_goldfish/blob/b2g-4.4.2_r1/qemud/qemud.c) and the corresponding socket `/dev/socket/qemud` are no longer in use, however it still keeps legagy code for backward compatibility, and consequently the logs / configs look very confusing. For example, You can still find qemud service in [init.goldfish.rc](https://github.com/mozilla-b2g/device_generic_goldfish/blob/b2g-4.4.2_r1/init.goldfish.rc#L129-L131).

##### Initialization Flow
The first part of the initialization flow keeps the original design -- qemud was initailized as a char device since it was using serial port. The adaption happens in `_android_qemud_pipe_init` function, in which it initialized itself as a pipe service.
```sequence
note right of vl_android: main function
vl_android->vl_android: serial_hds_add_at
vl_android->qemu_char: qemu_chr_open
qemu_char->qemu_char: qemu_chr_open_opts
qemu_char->qemu_char: backend_table[i].open
qemu_char->qemu_char: qemu_chr_open_android_qemud
qemu_char->hw_qemud: android_qemud_get_cs
hw_qemud->hw_qemud: android_qemud_init

note right of hw_qemud: adapt qemu-pipe
hw_qemud->hw_qemud: _android_qemud_pipe_init
hw_qemud->pipe: goldfish_pipe_add_type
```

Refer [backend_table](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/qemu-char.c#L2553-L2594) and [android_qemud_init](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/android/hw-qemud.c#L2268-L2278).

### QEMUD Services
[Qemud services](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/docs/ANDROID-QEMUD-SERVICES.TXT) are registered through [qemud_service_register](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/android/hw-qemud.c#L2304-L2310):
- service_name: an unique name for the service.
- max_clients: the maximum number of clients accepted by the service concurrently. If this value is 0, then any number of clients can connect.
- serv_opaque: anything the caller wants to keep. It will be carried back to the callback functions.
- serv_connect: see [QemudServiceConnect](https://github.com/android/platform_external_qemu/blob/dc053e276a0484385c5f34d0b271e6704f0d4c3f/android/hw-qemud.h#L114-L121).
- serv_save: see [QemudServiceSave](https://github.com/android/platform_external_qemu/blob/dc053e276a0484385c5f34d0b271e6704f0d4c3f/android/hw-qemud.h#L123-L126).
- serv_load: See [QemudServiceLoad](https://github.com/android/platform_external_qemu/blob/dc053e276a0484385c5f34d0b271e6704f0d4c3f/android/hw-qemud.h#L128-L131).
```c
QemudService*
qemud_service_register( const char*          service_name,
                        int                  max_clients,
                        void*                serv_opaque,
                        QemudServiceConnect  serv_connect,
                        QemudServiceSave     serv_save,
                        QemudServiceLoad     serv_load );
```

Launch the emulator with `-debug-qemud` option shows the log:
```
emulator: Registered QEMUD service hw-control
emulator: Registered QEMUD service boot-properties
emulator: Registered QEMUD service gsm
emulator: Registered QEMUD service gps
emulator: Registered QEMUD service camera
emulator: Registered QEMUD service sensors
emulator: Registered QEMUD service fingerprintlisten
```
> There were "adb" and "adb-debug" qemud services, but apparently adb services didn't work well and are currently disabled. See [propertyFile_getAdbdCommunicationMode](https://github.com/android/platform_external_qemu/blob/a7a04854907de2bb23eb48bac942b32ec2654ba8/android/avd/util.c#L238-L241).

#### Service Workflow

------

# Unsorted
Rild support for qemu pipe implemented in [commit 385a739](https://github.com/mozilla-b2g/android-hardware-ril/commit/385a73934b05fd28915e0ae17020dbfe3b20afd4).

## To Study
- qemud / charpipe
- display state / framebuffer / memory dirty flags 
- opengl emulation
- sdl / qt bindings and event loop handling
- device trees / virtio
