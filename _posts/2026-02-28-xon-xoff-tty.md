---
layout: default
title: Flow control and other underused terminal features
summary: The ancient POSIX TTY subsystem provides features that are often misunderstood and rarely used.
         I discuss the software flow control mechanism and how it can be used instead of pagers or to communicate backpressure to a program writing on standard output.
---

# Flow control and other underused terminal features

A common occurrence when working with the command line is running a command that produces a lot of rapid output, like `cat`-ing a file or following logs.
To be able to follow it, most people will kill the command with `ctrl-c` and re-run it with a pager like `more` or `less`.
Graphical terminal emulators and terminal multiplexers also give you history and the ability to scroll back to examine output you missed.
If you're running the command under [tmux](https://github.com/tmux/tmux), for example, you can pause and examine the output by entering the copy mode (`ctrl-b` followed by `[` by default).

Few know this, but there is a much more fundamental way of pausing the output that is probably going to work on any modern Unix-like system.
You do this by pressing `ctrl-s` to stop and then `ctrl-q` to resume (although very likely any other key will work as well).

As a small aside, key combinations like `ctrl-c` are often written in [caret notation](https://en.wikipedia.org/wiki/Caret_notation) as `^C` or similar.
This ancient tradition goes back to the terminals of the 1960s, but causes unending confusion in modern times when keyboards have an actual separate `^` character.
Strictly speaking, `^C` doesn't refer to the key combination but to the ASCII control character `0x03` (`ETX`, End of Text), with the caret acting as a sort of negation: `0x03` is the ASCII code for `C` with bit 6 flipped.
Other combinations work in a similar manner:

- `^D` is `0x04` (`EOT`, End of Transmission)
- `^S` is `0x13` (`XOFF`)
- `^Q` is `0x11` (`XON`)

Each one of those corresponds to the ASCII code of the capital letter (even though shift was *not* pressed!) with bit 6 flipped.
The same logic would suggest that, for example, you can type a newline character (`0x0a`) as `^J`, and indeed you can, as well as `^H` instead of backspace.

A similar convention lives on in Apple keyboards, where the Control key is labeled with a slightly wider "⌃" caret symbol, different from the "^" found above the "6" key.
Mac GUIs will often use the Unicode `U+2303` symbol for displaying keyboard shortcuts that involve the control key, e.g. ⌃C for `ctrl-c`, just like they use other Apple-isms like ⌥ or ⌘.
The decision to use the confusing caret symbol is uncharacteristic for Apple, and [looking at historical Apple keyboards](https://en.wikipedia.org/wiki/Apple_keyboards), it appears to be a recent invention.

## Software flow control

The combination `ctrl-s` (`^S`) that pauses the command output corresponds to the `XOFF` ("transmit off") control character.
This is part of the [software flow control](https://en.wikipedia.org/wiki/Software_flow_control) functionality sometimes used with serial interfaces.
`XOFF` and `XON` (i.e. `ctrl-q`) are sent in-band, along with all other input, and allow the receiver to turn off and on the transmitter's output.
This is a form of backpressure, albeit a limited one, since there is no way for the receiver to communicate the data rate at which it would like to consume its input.

As with everything terminal-related, it's often unclear which part of the software stack handles these key combinations.
Is it your terminal emulator? The shell?
Perhaps a bit surprisingly, in this case it's actually the operating system kernel.
As anyone who has tried to directly deal with serial ports on Linux knows, `/dev/tty*` devices have some extra special confusing magic applied to them by default.
The [POSIX Standard](https://pubs.opengroup.org/onlinepubs/9799919799/) mandates a lot of this behavior in its description of the [terminal interface](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/V1_chap11.html).

TTYs are not simple "naked" character devices; there's an additional layer between the character device layer and the hardware.
On Linux, this is the so-called "`N_TTY` line discipline" implemented in [n_tty.c](https://codebrowser.dev/linux/linux/drivers/tty/n_tty.c.html).
This layer handles echo behavior, line buffering of user input, and most of the control characters like `^C` or `^Z`.
Many of those get translated into signals that are then delivered to the foreground process group.
In default configuration, your program reading from the standard terminal input will not see the `0x03` byte but will instead receive a `SIGINT` signal.

The behavior of this layer can be customized from userspace using the `stty` command or an appropriate C API.
It can also be completely swapped for other kernel-provided line disciplines with the [TIOCSETD](https://man7.org/linux/man-pages/man2/TIOCSETD.2const.html) ioctl, although this is a pretty obscure feature probably only used by `pppd`.
Many things can be customized, echo can be disabled (useful for password input), and most of the control character behavior can be remapped or disabled.
Running `stty -a` lets you examine your current settings, and the manual page documents available options (which differ between Linux, macOS, and other operating systems).
On Linux, we have the following options relevant to flow control:

- `ixon` enables both `XON` and `XOFF` typed by the user to be processed by the kernel
- `ixany` allows any character to restart output, not just `XON`
- `ixoff` enables software flow control in the other direction -- the kernel will send `XOFF` if your typing speed exceeds its processing speed (unlikely to be the case)

## Detecting terminal output backpressure

So how does this flow control stuff look from the program's point of view?
Could we use it to make semi-interactive command line tools that intelligently throttle their output speed?
To some extent yes, but one needs to be careful as there are multiple layers of buffering involved.

The runtime library of your programming language will most likely buffer standard output.
This needs to be either disabled, or you need to find a way to invoke the `write()` system call directly.
The TTY subsystem in the kernel has its own additional buffer, which you need to flush using the [tcdrain](https://pubs.opengroup.org/onlinepubs/9799919799/functions/tcdrain.html) C API.
If you don't do this, `write()` calls will not block until this (considerably large) buffer is full.

The following simple Python example echoes each line it reads from standard input while estimating the desired data rate of the consumer.
The consumer can stop input with an `XOFF` but cannot request the data to be sent faster, so we need to slowly increase the rate up to a certain point (hence the 1.1 factor in the `bps` calculation).

```python
import time, os, termios, sys

fd_out = sys.stdout.fileno()  # almost always 1
t_start = time.monotonic()
n_bytes = 0

for data in sys.stdin.buffer:
    t_line = time.monotonic()
    os.write(fd_out, data)  # assumes full write for simplicity
    try:
        termios.tcdrain(fd_out)
    except termios.error:
        # will throw when stdout not a TTY, real code should
        # distinguish this from other errors
        pass
    t = time.monotonic()
    n_bytes += len(data)
    bps = min(1000, max(1, 1.1 * n_bytes / (t - t_start)))
    time.sleep(max(0, t_line + len(data) / bps - t))
```

This will work fine with flow control, but not with pagers like `more` or `less`.
Those programs read a lot of data into their internal buffers as they, by design, try not to interfere with their input process.
It *will* reduce the output rate when the process is suspended with `^Z` and subsequently resumed using a shell job-control command like `fg`.
This is because `time.monotonic()` is a system-wide clock that continues advancing when the process is suspended.

It might seem enough to replace the system monotonic clock with `time.process_time()` but this will not have the desired effect.
Process time stops during blocking I/O, which includes the `tcdrain()` call.
If you need to detect flow-control suspends explicitly, it might be necessary to time the `tcdrain()` execution time explicitly or examine the process state.
