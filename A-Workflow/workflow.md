VM/Kernel Workflow
==================
Before we get into the heavier assignments in this course,
you should invest sometime into setting up a workflow and develop
good coding habits.

Why Use a VM?
-------------

- Hypervisor: Software responsible for managing VMs (e.g. creation, deletion, resource allocation). In this class, we use VMware.
- Host machine: The hardware running the hypervisor (e.g. your personal laptop)
- Virtual machine (VM): A computer system created by the hypervisor that shares resources with the host machine. In this class, we virtualize Debian 11 (a flavor of Linux). VMs adhere to sandbox properties:
  - Isolate the work you'll be doing from the rest of the system
  - Be able to take snapshots of your development environment so that if it does get corrupted, you can revert to a clean state
  
  Working on a VM can be preferable to working on your host machine (i.e. "bare-metal"). You can virtualize other operating systems to test your applications. You can work on potentially buggy/dangerous code without compromising your host machine.


- Snapshot: A feature most hypervisors support. Capture the current running state of the VM, stores it, and then allows you to revert to that state sometime later. You should snapshot your VM before executing something that can compromise the system.

In this class, we will often be "hacking" the Linux kernel and attempting to boot into those hacked kernels. We need to use VMs since not all of us have a Linux host machine. Even if we did, we wouldn't want to ruin our host machines with our potentially buggy kernels.

VM Setup/Workflow
-----------------
We've already written a series of guides for working on your VM.

First and foremost, you should have a Debian VM installed already (see [Debian VM Setup](https://cs4118.github.io/dev-guides/debian-vm-setup.html) for a guide on how to do this).

### Working on your VM
You have more options than simply working on the VM's graphical
interface, which often feels clunky.

The most common workflow involves SSHing into your VM, which
we've written a [guide](https://cs4118.github.io/dev-guides/vm-ssh.html#ssh-into-your-local-vm) for. This is a good option
if you want to conserve processing power on your host machine and
disable the VM's graphical interface. 

Alternatively, you can setup an IDE to SSH into your VM. One option is using Visual Studio Code, which we've written up some [notes](https://cs4118.github.io/dev-guides/vm-ssh.html#using-visual-studio-code-optional) on how to use. This is nice alternative to command-line editors like vim/emacs if you're
not familiar with them.

### Additional Tools
If you do decide to work in a command-line environment, here are
some tools we've used to streamline our workflow:

- `bat`: A better version of `cat` [(installation)](https://github.com/sharkdp/bat#on-ubuntu-using-most-recent-deb-packages)
- `grep`: Pattern-match files (learn how to effectively use regexs to improve search results).
  Even better, `ripgrep`.
- Reverse-i search: Efficiently search through bash history instead of retyping long commands.
- `tmux`: Terminal multiplexer. (e.g. open 2 panes for vim, 1 for `dmesg`, 1 for running commands).

Kernel Workflow
---------------
### `vim` Enhancements
Before getting into hw4/supermom, please read through our 
[kernel developer workflow](https://cs4118.github.io/dev-guides/kernel-workflow.html)
guide. Here, we offer a bunch of cool `vim` additions/tricks to
help you develop more efficiently while working on the kernel assignments. Note that this guide is only relevant if you intend
to work on the command-line using `vim`. One notable mention here
is `cscope`. This is a kernel-source navigator that works 
directly in your terminal/vim session. This is far more powerful
than using `grep` to look for a symbol definition/use in the source-tree.

### Web Tools
If you don't want to use `cscope`, there's an popular online 
kernel-source navigator:
[bootlin](https://elixir.bootlin.com/linux/v5.10.57/source).
Note that kernel version matters when you're navigating code – be sure you select the correct version.

Like bootlin, you can look for symbols in the kernel-source and
look for instances of where they are defined and used. However,
bootlin doesn't index all symbols, so you might have better luck
searching with `cscope`.