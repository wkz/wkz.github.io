---
title: "/dev/kmem + GDB Stub = kmemd"
date: 2022-09-24T01:45:00+02:00
---

This is an introduction to [kmemd](https://github.com/wkz/kmemd) - a
tool for exploring a live Linux kernel's memory in a non-intrusive way
using GDB.

When debugging issues in the Linux kernel, I often find myself wanting
to inspect the state of internal data structures. This typically
involves non-trivial operations like walking liked lists and
extracting enclosing objects using the equivalent of
[`container_of`](http://www.kroah.com/log/linux/container_of.html)
etc. Fortunately the kernel comes with a heap of Linux kernel specific
Python extensions to GDB, that you can easily extend to make whatever
tools you need - things like custom pretty printers for example.

However, to make use of all that goodness, we need a GDB stub through
which GDB can access the kernel's memory. Typically, this means you
need some form of debugger; either a piece of hardware that connects
to a JTAG TAP on the CPU, or using Linux's KGDB, which is pretty much
a software implementation of the same thing.

As these tools give you complete control of a CPU core's execution,
they are naturally very disruptive on a system level: All processes in
the system will grind to a halt, requests from the network go
unanswered, hardware watchdogs may trip, etc.

Crucially though, there are many cases where I don't actually need
that level of control. Often, I would be happy with having the kernel
executing at full speed, as long as I can inspect its memory.


## Enter kmemd

```
                    .-----.
                    | GDB |
                    '-v-^-'
                      | |
          - File, UNIX or TCP socket -
                      | |
                   .--v-^--.
                   | kmemd |
                   '-v---^-'
           .---------'   '-----------.
    .------v------.             .----^----.
    | BPF Program |             | BPF Map |
    '------v------'             '----^----'
           '--- probe_read_kernel ---'
```

Following the time-honored software tradition of telling blatant lies
to the layers above us, kmemd implements just enough of the GDB Remote
Server Protocol to convince an attaching instance of GDB that it is
talking to a halted machine, who's memory it can therefore freely
read.

When a request to read a region of memory is received, kmemd uses
[`BPF_PROG_RUN`](https://docs.kernel.org/bpf/bpf_prog_run.html) to
trigger execution of a BPF program that stores a copy of the requested
region in a map. The data is read back to kmemd via the BPF map, and
is then sent back to GDB.{{< sidenote >}}In days of yore, we could
have just `mmap`ed `/dev/kmem` to get the information. Alas, that
interface has been [removed](https://lwn.net/Articles/851531/).{{<
/sidenote >}}

With that, the full power of GDB's DWARF parsing, Python scripting,
etc. is available to use on your live kernel.{{< sidenote >}}A
read-only view of memory is all you get though. Forget about
single-stepping, setting breakpoints etc. Your system is still very
much live.{{< /sidenote >}}


## Next steps

If this sounds useful to you, I would propose that you head on over to
the project's [README](https://github.com/wkz/kmemd#readme) to get
more practical information.
