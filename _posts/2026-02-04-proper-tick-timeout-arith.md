---
layout: default
title: Dealing with 32-bit Tick Arithmetic
summary: Many embedded systems maintain a global "tick" counter that is incremented on every system
         timer interrupt. This article offers advice to avoid the misery of dealing with bugs that
         result from this counter overflowing.
---

# Dealing with 32-bit Tick Arithmetic

Many embedded systems maintain a global "tick" counter that is incremented on every system timer
interrupt (often every millisecond). RTOSes and microcontroller libraries provide functions to
access it, e.g.:

- `HAL_GetTick()` in [STM32 HAL](https://www.st.com/resource/en/user_manual/dm00105879.pdf), returning `uint32_t`
- `xTaskGetTickCount()` in [FreeRTOS](https://www.freertos.org/media/2018/FreeRTOS_Reference_Manual_V10.0.0.pdf), returning a `TickType_t`
- `millis()` in [Arduino libraries](https://www.arduino.cc/reference/en/language/functions/time/millis/), returning an `unsigned long`
- `rtos::Kernel::get_ms_count()` in [ARM Mbed](https://os.mbed.com/docs/mbed-os/v6.16/feature-hal-spec-spi-doxy/group__rtos.html) OS, returning `uint64_t`

The main purpose of those counters is handling timeouts, but they are sometimes used as a local
timestamp source to store in internal data structures along with events or data samples. This is
similar to the `CLOCK_MONOTONIC` clock available on modern Unixoid systems.

On microcontrollers, however, instead of a humongous `struct timespec` for your monotonic clock,
you'll often get a 32-bit unsigned integer that will wrap back to zero after about 50 days of use.
50 days is long enough for any problems to be easy to miss during testing, but short enough that
realistic usage scenarios for many devices will reach it.

Many bugs are known to crawl out around the two-month mark, including a slightly mysterious one in
Boeing 787 airplanes leading to the
[recommendation](https://www.theregister.com/2020/04/02/boeing_787_power_cycle_51_days_stale_data/)
that they should never be operated longer than 51 days without a power cycle. This seems close to
either a 32-bit millisecond counter overflow (49.71 days) or a 16-bit minute counter overflow (45.51
days) but larger than either value.
[Rumor](https://www.i-programmer.info/news/149-security/13597-power-cycle-your-boeing-787-to-keep-it-flying.html?utm_source=chatgpt.com)
has it, it could be a 42-bit microsecond-resolution time value (50.9 days). Perhaps it's some kind
of weird hardware register, or the upper bits of a 64-bit value are
unintentionally zeroed or used for something else?

## Then just use 64 bits, right?

One option would be to avoid the problem completely by using a 64-bit system tick counter. At a 1 kHz
tick interrupt rate, it will overflow after more than half a billion years, and surely the extra
safety is worth a little bit of runtime overhead? ARM Mbed in the list above has taken such
an approach, although its documentation states:

> If the underlying RTOS only provides a 32-bit tick count, this method expands it to 64 bits.

The method is also marked as deprecated, and it looks like it has been removed in favor of a more
POSIX-like structure. I haven't used ARM Mbed and I don't know how often you'll actually receive a
32-bit tick while expecting a 64-bit one -- perhaps it's something that can only happen in very
exotic configurations.

On FreeRTOS, both the tick rate and the bit size of `TickType_t` are configurable (either 16, 32, or
64 bits), meaning that with a 1 kHz system tick, the counter could wrap around after roughly one
minute or over half a billion years.

Since `TickType_t` is a simple typedef, both in FreeRTOS and on ARM Mbed, the library or application
code can implicitly downcast a 64-bit counter to a smaller value if one is not careful. Default
compiler settings will often not give you any warnings about that, though [MISRA
C](https://en.wikipedia.org/wiki/MISRA_C) forbids implicit narrowing and static analyzers targeting
it should be able to catch this mistake.

If those tick values are stored as timestamps of events or samples in microcontroller RAM, the
overhead might be non-negligible. Otherwise, it seems reasonable to use 64 bits when doing
everything from scratch, but libraries and existing codebases might make it difficult.

## Handling overflows properly

A much more common solution is to simply let the counter overflow and deal with this possibility at
its point of use. There are many common ways of doing this, and a few of them are even correct! The
incorrect ones can lead to the following behavior:

- An I/O operation fails immediately because the computed "deadline" wrapped around 0 and is a
  numerically smaller value than the current tick.
- An operation that should time out doesn't because the clock has wrapped back to 0 and a computed
  "deadline" is near the end of the range. Since this happens after prolonged use and the system
  seems to hang forever, it can be easily mistaken for a race-condition-caused deadlock.
- A sequence of timestamped records or samples is assumed to be sorted but in reality is not.
  Corrupted files are produced or weird stuff happens elsewhere, perhaps even on another side of a
  communication link.

One correct (but slightly confusing) solution is to use 32-bit signed integers for a difference of
time values and use that signed difference in timeout calculations (instead of a "deadline" time
instant value). My favourite approach is to avoid using any signed values altogether:

```c
bool is_tick_before(uint32_t a, uint32_t b)
{
    return (a - b) >> 31 != 0;
}
```

This can be used when checking for timeouts with a "deadline" value that can safely wrap around
to the beginning of the range. This deadline can then be passed around to different functions, which
is a lot more composable than having a "timeout" argument. A sequence of timestamps extended at the
end will be sorted according to this ordering, but not necessarily according to the usual comparison
operator "`<`".

Why does this work? Checking the most significant bit effectively amounts to checking if the
difference is larger than half of the range (a bit over 24 days for a millisecond counter).
Since addition and subtraction in `uint32_t` are required to follow the rules of modular arithmetic,
if one of the values passed is `t` and another `t + dt`, the result will either be `dt` or `0 - dt`,
depending on the order. As long as both timestamps were captured within 24 days of each other, `dt`
will not have its most significant bit set, but `0 - dt` will.

If possible, it's probably best to avoid "naked" `uint32_t` for those values since accidental use of
ordinary integer comparison is difficult to fully prevent. In C, one can introduce an opaque wrapper
struct and a set of safe operations as functions. In C++, the temptation to build a whole class with
operator overloading will be impossible to resist.

## Overflow quickly or never?

A somewhat controversial practice is to initialize this system tick to a high value on startup,
ensuring its overflow within a few minutes of operation. It should be at least a few minutes to
ensure the system is in a steady state and not performing some kind of initialization, when many of
the potentially buggy components are not yet running.

This makes bugs easier to catch during testing, but the ones that you've missed will be more likely
to cause problems during normal customer use. For many systems, 50 days of continuous use without a
power cycle is an allowed, but very unusual usage pattern. Perhaps a more pragmatic approach is to
only enable this tick initialization in test builds.
