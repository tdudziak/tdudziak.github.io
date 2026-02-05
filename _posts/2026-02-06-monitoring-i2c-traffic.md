---
layout: default
title: Monitoring I²C traffic on Linux
summary: How to record incoming and outgoing traffic on an I²C bus without external hardware, heavy
         dependencies, or complicated tools.
---

# Monitoring I²C traffic on Linux

When working with I²C sensors on embedded Linux, you often find yourself with a mixture of kernel
drivers and userspace applications, sharing multiple buses with many peers attached to them.
For a prototype on your desk, you can record all bus traffic with a logic analyzer. With more
finished devices, this is often impractical or outright impossible if the system in question is
deployed in the field and only accessible via SSH.

Unlike CAN, I²C buses are not treated as network interfaces on Linux, and the usual packet sniffing
tools will not work on them. Luckily, the kernel supports a lot of tracing, debugging, and
monitoring functionality. So much, in fact, that it can quickly become overwhelming.

There's [ftrace](https://docs.kernel.org/trace/ftrace.html) with several different selectable
tracers, there's the [eBPF](https://ebpf.io/) virtual machine together with the
[bpftrace](https://bpftrace.org/) tool based on it. Then, there are even more flexible dynamic tools
like [kprobes](https://docs.kernel.org/trace/kprobes.html). It's a lot of machinery for functionality
you only need sporadically, while debugging, and often not in a mental state conducive to reading
multiple book chapters' worth of documentation.

In reality, it's quite simple. The only thing you need to dump I²C traffic is the tracefs
filesystem. It's often mounted and enabled by default on many consumer Linux distros. For a custom
embedded image, you might need to enable it in kernel config or provide it as a loadable module. If
the directory `/sys/kernel/tracing/events/i2c` exists on your system, you're in luck and you don't
need to install or enable anything.

## Recording traces

The tracefs filesystem is usually mounted at `/sys/kernel/tracing` and is simple enough to be
usable without a dedicated userspace tool. Enabled events are readable in line-oriented ASCII format
from `trace_pipe`, which you can examine manually with `cat` or compress and store on disk. When no
writable filesystem is available, you can also pipe it over SSH:

```bash
ssh root@xxx sh -c \
    "</sys/kernel/tracing/trace_pipe bzip2" >out.bz2
```

If your intention is to measure jitter, the extra CPU load of compression and encryption performed
by SSH could interfere with the measurement.

The I²C subsystem exposes several different tracepoints:

- `i2c_read`: a read message will be dispatched to the driver
- `i2c_write`: a write message will be dispatched to the driver (payload included in the event)
- `i2c_reply`: a read I²C transaction has been performed (reply payload included)
- `i2c_result`: a transfer (possibly comprised of multiple reads and writes) has been finished

The events are all generated within the
[`__i2c_transfer()`](https://elixir.bootlin.com/linux/v6.18.6/source/drivers/i2c/i2c-core-base.c#L2221)
function in the kernel sources (look for calls to `trace_*`). They can be enabled selectively by
writing ASCII `1` to the `enable` file within the directory that represents them, e.g.:

```bash
echo 1 >/sys/kernel/tracing/events/i2c/i2c_reply/enable
```

If you want all of them, just do:

```bash
echo 1 >/sys/kernel/tracing/events/i2c/enable
```

## Understanding the output

Each line in the ASCII format begins with some common trace fields: task name, PID, CPU, flags,
timestamp. The format of the event-specific payload that follows can be examined via `tracefs`:

```bash
cat /sys/kernel/tracing/events/i2c/i2c_reply/format
```

Since I²C is not a complicated protocol, the format is mostly self-explanatory: adapter number and
addresses are included, and any payload is printed in hexadecimal.

If you want to validate the sampling rate or measure timing jitter, the best approach is to record a
trace and process it on your local system. With `grep` and simple regular expressions, you can
filter only relevant events and then compute timing statistics with something like this Python
few-liner:

```python
import sys, re, numpy as np
dt = np.diff(np.array([\
        float(m.group(1))
        for l in sys.stdin
        if (m := re.search(r'(\d+\.\d+):', l))\
]))
print(np.median(dt), np.percentile(dt, 95), np.max(dt))
```

This will compute differences (in seconds) between consecutive events and print out the median, 95th
percentile, and the maximum value.
