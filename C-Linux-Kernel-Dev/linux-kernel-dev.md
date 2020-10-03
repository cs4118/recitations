Linux Kernel Development
========================
This document is meant to supplement our [compilation guide](https://cs4118.github.io/dev-guides/debian-kernel-compilation.html),
which goes over how to install a new kernel on your machine.

Kernel Module Makefile
----------------------
More info in our [kernel module guide](https://cs4118.github.io/dev-guides/linux-modules.html), but for the sake of comparison, here it is again:
```
obj-m += hello.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
This Makefile is separate from the top-level kernel source Makefile we're about to work with. Here, we're adding to the `obj-m` "goal definition" (build object as module) and using a specialized Makefile located at `/lib/modules/$(shell uname -r)/build` to link our module into the kernel.

Top-level Kernel Makefile
-------------------------
`linux/Makefile` is the only Makefile you'll be executing
when developing with the kernel. At a high level, this
Makefile is responsible for
- recursively calling Makefiles in subdirectories
- generating kernel configurations
- building and installing kernel images

Here's an overview of the common operations you'll be doing
with this Makefile.

### Preparing for development
It's good practice to ensure you have a clean kernel source before beginning
to development. Here's the cleaning options that this Makefile provides, as
reported by `make help`:
```
    Cleaning targets:
  clean       - Remove most generated files but keep the config and
                enough build support to build external modules
  mrproper    - Remove all generated files + config + various backup files
  distclean.  - mrproper + remove editor backup and patch files
```

`make mrproper` is usually sufficient for our purposes.
Be warned that you'll have to completely rebuild the kernel
after running this!
### Configuration and compilation
`linux/.config` is the kernel's configuration file. It lists a bunch of
options that determine the properties of the kernel you're
about to build. It also determines what code will be
compiled and linked into the final kernel image. For example, if `CONFIG_SMP` is
set, you're letting the kernel know that you have more than one processor so it
can provide multi-processing functionality.

There's a bunch of different ways to generate this configuration file.
Here's the relevant options we have, as reported by `make help` in the
top-level kernel directory:
```
    Configuration targets:
  config          - Update current config utilising a line-oriented program
  menuconfig      - Update current config utilising a menu based program
  oldconfig       - Update current config utilising a provided .config as base
  localmodconfig  - Update current config disabling modules not loaded
  defconfig       - New config with default from ARCH supplied defconfig
  olddefconfig	  - Same as oldconfig but sets new symbols to their
                    default value without prompting
```

Summarizing from our [compilation guide](https://cs4118.github.io/dev-guides/debian-kernel-compilation.html), we set up our kernel config in a couple of
steps:
```
make olddefconfig # Use current kernel's .config + ARCH defaults
make menuconfig   # Manually edit some configs
```
If you want to **significantly** reduce your build time, you can also set
your config to skip unloaded modules during compilation:
```
yes '' | make localmodconfig
```

As you're developing in the kernel, you might add additional source files that need to be compiled and linked in. Open up
the directory's Makefile and add your desired object file to
the `obj-y` goal definition, which is for "built-in" functionality (as opposed to kernel modules). For example, here's the relevant portion of `linux/kernel/Makefile`:
```
obj-y     = fork.o exec_domain.o panic.o \
	    cpu.o exit.o softirq.o resource.o \
	    sysctl.o sysctl_binary.o capability.o ptrace.o user.o \
	    signal.o sys.o umh.o workqueue.o pid.o task_work.o \
	    extable.o params.o \
	    kthread.o sys_ni.o nsproxy.o \
	    notifier.o ksysfs.o cred.o reboot.o \
	    async.o range.o smpboot.o ucount.o
```
If you were adding a source file to `linux/kernel`, you'd add the object file target here.


Once you're ready to compile your kernel, you run the following in the top-level source directory:
```
make -j $(nproc)
```
This will compile your kernel across all available CPUs, as reported by `nproc`. Note that this top-level Makefile will
recursively build source code in subdirectories for you!
### Kernel installation
Like in our [compilation guide](https://cs4118.github.io/dev-guides/debian-kernel-compilation.html), you run the following command once
the compilation finishes:
```
sudo make modules_install && sudo make install
```
The first time you install a kernel, you must build `modules_install`.
All subsequent times, `install` is sufficient.

Finally, reboot and select the kernel version you just installed via the
GRUB menu!

Kernel Code Style
-----------------
The kernel source comes with a linter written in Perl (located at `linux/scripts/checkpatch.pl`).

checkpatch will let you know about any stylistic errors and
warnings that it finds in the codebase. You should get into
the habit of running checkpatch and fixing what it suggests
before major checkpoints.

If you want a general overview of kernel code style, here's
[one](https://www.kernel.org/doc/html/v4.19/process/coding-style.html). You can also find this in `linux/Documentation/process/coding-style.rst`.

Debugging Techniques
--------------------
- Take [snapshots](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/snapshots.html) of your VM before installing software that may corrupt it.
- Use `printk/pr_info` to log messages to the kernel log buffer (viewable in userspace with by running `sudo dmesg`)
- Set up a [serial port](http://cs4118.github.io/freezer/#tips) to redirect log buffer to your host machine (in case VM crashes before you can check it with `dmesg`).
