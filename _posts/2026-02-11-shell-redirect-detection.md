---
layout: default
title: Detecting Targets of Redirections in Unix tools
summary: Classic Unix command-line tools process text from standard input into standard output without caring which files those are redirected to, if any.
         Mechanisms to examine their paths based on file descriptors exist, and there are occasional reasons to use them.
---

# Detecting Targets of Redirections in Unix tools

Classic Unix command-line tools work by consuming text input and producing some output.
You can pipe them together in the shell, or redirect their I/O to a file with "`<`" and "`>`" operators.
Most of the time the tool itself doesn't need to care whether this is happening, but there are cases where you need to identify the file your stdin or stdout is connected to.

Often you don't need to know the path of the file; you only care about certain properties like whether it's a TTY or whether it has a known size.
For example, you might want to output ASCII control sequences or have some light TUI features like progress bars.
Knowing that your input is seekable or has a fixed size allows you to make some optimizations or preallocate buffers.
POSIX provides functions like `isatty()` or `lseek()` that you can use, as long as you handle the failure case properly.
Here I focus on the specific scenario where you care whether your stdin or stdout are actual files on the filesystem.

The most common reason for this is when you need to detect input/output overlap.
Tools like `ag` or `rg`, that perform a recursive text search of files in some directory, have a funny failure mode if their output is redirected to one of the files they might scan.
Since the output contains lines matching the search query, they might end up being appended to the output file again and again.
It might not always play out exactly that way depending on buffering, but it's still a corner case worth catching.

## Verifying if you already know the path

For purposes of detecting input/output overlap, you don't really need to obtain the path to which your stdin or stdout is redirected.
You can use standard POSIX APIs to check this before you open a candidate file and either skip it or fail with an error if overlap occurs.
This is done by calling `fstat()` on the file descriptor and `stat()` on the path to obtain `struct stat` metadata and compare the device and inode numbers.
It's important to compare both, as inode numbers are only unique within a filesystem.
You can use the following function:

```c
#include <sys/stat.h>
#include <unistd.h>

int is_same_file(int fd, const char* path)
{
    struct stat st_fd, st_path;
    if (fstat(fd, &st_fd) != 0 || stat(path, &st_path) != 0) {
        return -1;
    }
    if (st_fd.st_dev == st_path.st_dev && \
        st_fd.st_ino == st_path.st_ino) {
        return 1;
    }
    return 0;
}
```

You would typically pass either 0 (`STDIN_FILENO`) or 1 (`STDOUT_FILENO`) as the first argument.
If you intend to perform this check many times, it can be slightly faster to call `fstat()` once and reuse the result.

Note that, as with most filesystem operations, some races are unavoidable.
A file can be unlinked and another created in its place immediately after `is_same_file()` runs.
Do not rely on this for permission checks or security.

## Finding the full path to the file on Linux

A file descriptor does not have to be associated with a filesystem path.
It can refer to a pipe or a socket, and even if it was originally produced by opening a file, that file might have been unlinked afterwards.
POSIX does not require the OS to track those changes, so there is no standard way to obtain the path as a string.

On Linux, `/proc/self/fd/<fd>` is a symlink that points to the kernel's idea of the path associated with a file descriptor.
This mechanism allows you to craft a shell command that will write the name of a redirected file to the file itself:

```console
/tmp$ readlink /proc/self/fd/1
/dev/pts/0
/tmp$ readlink /proc/self/fd/1 >out.txt && cat out.txt
/tmp/out.txt
```

The link target is best treated as debugging or diagnostic information and is not guaranteed to be a valid filesystem path, for example:

```console
/tmp$ echo hi | readlink /proc/self/fd/0
pipe:[76020656]
/tmp$ (sleep 1 && rm out.txt && readlink /proc/self/fd/0) <out.txt
/tmp/out.txt (deleted)
```

To make use of this mechanism in C, use the standard `readlink()` function to write the target path into a buffer.
Since the file descriptor is either 0 or 1, you can save on string processing and use a compile-time string constant as the argument.
Afterwards, you must validate the path with a call to `is_same_file()` defined above.
If it's one of the special non-path values or a path with `(deleted)` at the end, that call will fail.
If the result is a device node or a TTY (like `/dev/pts/0` in the output above), this verification step will succeed since devices *are* files.
Depending on what you're trying to accomplish, this might not be the result you want.
