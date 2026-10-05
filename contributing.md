## Table of Contents
* [Definitions](#definitions)
    * [1. Error messages](#1-error-messages)
    * [2. i3 ipc return values](#2-i3-ipc-return-values)
    * [3. Payloads](#3-payloads)
* [Functions](#functions)
    * [1. Function readn](#1-function-readn)
        * [Source code](#source-code)
        * [Explanation](#explanation)
    * [2. Function simple_atoi()](#2-function-simple_atoi)
        * [Source code](#source-code-1)
        * [Explanation](#explanation-1)
    * [3. Function get_i3_socket_path()](#3-function-get_i3_socket_path)
        * [Source](#source)
        * [Explanation](#explanation-2)
    * [4. Function flush_reply()](#4-function-flush_reply)
        * [Source code](#source-code-2)
        * [Explanation](#explanation-3)
    * [5. Function connect_to_i3()](#5-function-connect_to_i3)
        * [Source code](#source-code-3)
        * [Explanation](#explanation-4)
    * [6. Function window_events_subscribe()](#6-function-window_events_subscribe)
        * [Source code](#source-code-4)
        * [Explanation](#explanation-5)
    * [7. Function read_single_window_event()](#7-function-read_single_window_event)
    * [8. Function send_i3_split_command()](#8-function-send_i3_split_command)
        * [Source code](#source-code-5)
        * [Explanation](#explanation-6)
    * [9. Function handle_signal()](#9-function-handle_signal)
        * [Source code](#source-code-6)
        * [Explanation](#explanation-7)
    * [10. Function main()](#10-function-main)
        * [Source code](#source-code-7)
        * [Explanation](#explanation-8)
---
# Contributing

To contribute to this project, please follow these standard steps:
1. **Fork** the repository and clone it locally.
2. **Make your changes**, ensuring you adhere to the [Linux Kernel Coding Style](https://docs.kernel.org/process/coding-style.html).
3. **Compile and test** your code to ensure it builds cleanly without warnings.
4. **Submit a Pull Request** with a clear explanation of what you changed and why.

Before writing any code, it is highly recommended that you read the logic of the program
below to understand the architecture.

*The spirit of this project is:*
1. We try to minimize the includes
2. We try to work only statically, minimizing memory allocations

# Definitions
The code starts with important definitions that help the program
adapt to future changes.
### 1. Error messages
They follow this format
```c
#define ERR_<error description> "Message for console"
#define LEN_<error description> sizeof(ERR_<error description>) - 1
```
*Note: `<error description>` is a placeholder for the example*
Usage:
```c
write(STDOUT_FILENO, ERR_<error description>, LEN_<error description>);
```
This way is preferred cause `sizeof()` is calculated during compilation.

### 2. i3 ipc return values
Self-explanatory. Named the return values for readability later on.
The names as of now are:
```c
#define EVENT_IGNORE  0
#define EVENT_FOCUS   1
#define EVENT_ERROR   -1
```
Usage snippet inside of `read_single_window_event()`:
```c
int read_single_window_event(int i3_fd_event, int *out_width, int *out_height)
{
        int result = EVENT_ERROR;

        char *is_focus = NULL;
        char *is_new   = NULL;
        char *rect_ptr = NULL;

        struct i3_ipc_header event_header = {0};
        char json_payload[4096] = {0};
        // Rest of the code
}
```
*In this instance the default return value of the function is set to EVENT_IGNORE*
Then in `main()`:
```c
int main(void)
{
    // Code inside main
    // .
    // .
    // .
        while (keep_running) {
                int event = read_single_window_event(i3_fd_event, &width, &height);

                if (!keep_running || event == EVENT_ERROR)
                        break;

                if (event == EVENT_FOCUS) {
                        if (width > height)
                                send_i3_split_command(i3_cmd_fd, "split h");
                        else
                                send_i3_split_command(i3_cmd_fd, "split v");
                }
        }
// Closing sequence
closing:
// Code for closing sequence
}
```
Look in line number 13 of the sample above. It makes the logic easy to follow.
### 3. Payloads
As of now we have only one payload defined here.
```c
#define SUBSCRIBE_WINDOW_EVENT_PAYLOAD     "[\"window\"]"
#define SUBSCRIBE_WINDOW_EVENT_PAYLOAD_LEN 10
```
we use it when we need to send a payload to the i3 ipc.
**Next up we check out our functions**

---

# Functions
The goal of this part is to explain the technical details and why these choices were
made. Of course nothing is set in stone, this is the first version and everything is
subject to change if i have made a mistake.

These are our functions in the order we see them in the source code:
```c
ssize_t readn(int fd, void *ptr, size_t n) {}
int simple_atoi(const char *str) {}
const char *get_i3_socket_path(void) {}
void flush_reply(int fd, uint32_t size) {}
int connect_to_i3(const char *socket_path) {}
int window_events_subscribe(int i3_fd_event) {}
int read_single_window_event(int i3_fd_event, int *out_width, int *out_height) {}
int send_i3_split_command(int i3_cmd_fd, const char *split_x) {}
```

### 1. Function readn
#### Source code
```c
ssize_t readn(int fd, void *ptr, size_t n)
{
        size_t  nleft;
        ssize_t nread;

        nleft = n;
        while (nleft > 0)
        {
                if ((nread = read(fd, ptr, nleft)) < 0)
                {
                        if (nleft == n)
                                return -1;
                        else break;
                } else if (nread == 0) {
                        break;
                }
                nleft -= nread;
                ptr   += nread;
        }
        return (n - nleft);
}
```
#### Explanation
This is the easiest to explain, is it copied from the book:
[Advanced Programming in the Unix Environment 3rd Edition](https://raw.githubusercontent.com/zwan074/technical-books/master/Advanced.Programming.in.the.UNIX.Environment.3rd.Edition.0321637739.pdf)
Basically a wrapper for the `read()` function.
It is chosen as a very safe and minimal way to ensure that we read everything that we
specify, this is because if we miss anything our future reads will not be synchronized,
and the program seizes to function.
For more details click on the link and read from the book.

---
### 2. Function simple_atoi()
#### Source code
```c
int simple_atoi(const char *str)
{
        int res = 0;
        int i = 0;

        if (str[i] == ' ')
                i = 1;

        for (; str[i] >= '0' && str[i] <= '9'; ++i)
                res = res * 10 + (str[i] - '0');

        return res;
}
```
#### Explanation
`atoi()` means ASCII to integer.
Inspired by the book:
The C Programming Language - 2nd Edition, by Brian W.Kernighan and Dennis D.Ritchie
The only changes are the name of the function the change:
`char[]` -> `const char *str`
and the we defined res instead of n. Another important change is shifting the start
position by one if the first character is a space. This is necessary since some
json payloads, such as used by sway, include a space after every colon.

---
### 3. Function get_i3_socket_path()
#### Source
```c
const char *get_i3_socket_path(void)
{
        char *path = getenv("I3SOCK");
        if (path)
                return path;

        const char *fallback = "/tmp/i3-ipc.sock";
        if (access(fallback, F_OK) == 0)
                return fallback;

        return NULL;
}
```
#### Explanation
Almost fully transparent by the comments on the `autotiling.c` file.
2 checks are happening.
First we try to retrieve the environment variable `I3SOCK`. It should not
fail if i3 is running on the system. In the case the environment variable
is not set we also check the file `/tmp/i3-ipc.sock`.
If both fail then we assume that i3 is not running on the system and return `NULL`.
Else we return the file descriptor of the socket.

In our program we do this two times, to have one connection for reading window events,
and the second time to have a channel for sending the split commands.

---
### 4. Function flush_reply()
#### Source code
```c
void flush_reply(int fd, uint32_t size)
{
        char buf[512] = {0};
        uint32_t remaining = size;
        uint32_t to_read = 0;

        while (remaining > 0)
        {
                to_read = remaining;

                if (remaining > sizeof(buf))
                        to_read = sizeof(buf);

                ssize_t n = readn(fd, buf, to_read);

                if (n <= 0)
                         break;

                remaining -= n;
        }
}
```
#### Explanation
Many times we get a reply in the buffer that we want to ignore, it is important
that we get rid of these messages, that's why this function exists.
It uses a static buffer of sufficient size to read it and discard it.
It goes without saying that it is a helper function

---
### 5. Function connect_to_i3()
#### Source code
```c
int connect_to_i3(const char *socket_path)
{
        int socket_fd = socket(AF_UNIX, SOCK_STREAM, 0);
        struct sockaddr_un addr = { .sun_family = AF_UNIX };

        if (!socket_path) {
                write(STDOUT_FILENO, ERR_SOCKET_PATH_NOT_FOUND, LEN_SOCKET_PATH_NOT_FOUND);
                goto out;
        }

        if (socket_fd == -1) {
                write(STDOUT_FILENO, ERR_FAILED_TO_CREATE_SOCKET, LEN_FAILED_TO_CREATE_SOCKET);
                goto out;
        }

        strncpy(addr.sun_path, socket_path, sizeof(addr.sun_path) - 1);
        addr.sun_path[ sizeof(addr.sun_path) - 1 ] = '\0';

        if (connect(socket_fd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
                write(STDOUT_FILENO, ERR_FAILED_TO_CONNECT_TO_SOCKET, LEN_FAILED_TO_CONNECT_TO_SOCKET);
                close(socket_fd);
                goto out;
        }

        return socket_fd;

out:
    return -1;
}
```
#### Explanation
This function is used after we have saved the path to the socket.
Its goal is to return the file descriptor `socket_fd` so we can communicate with i3.
As we saw in [get_i3_socket_path() function](#3-Function-get_i3_socket_path())
if we don't get the path we have `NULL` stored. If this function sees `NULL` in
the `socket_path` it throws an error. Then we check for the return of the `socket()`
command.
**Buffer Safety:** Because we are avoiding dynamic memory allocation (`malloc()`)
we copy the socket path into a fixed-size static struct (`addr.sun_path`).
We use `strncpy()` with `sizeof(...) - 1` and manually add the `\0` terminator.
This guarantees we will never have a buffer overflow, even if the environment
variable gives us an unexpectedly long path.

**Minimalistic Error Handling:** Notice the use of `goto out;` for error cleanup.
This is a classic C pattern (heavily used in the Linux kernel) that keeps the code
clean and prevents us from writing `return -1;` repeatedly. Also, when an error occurs,
we use `write(STDOUT_FILENO, ...)` paired with the `ERR_` macros we defined in the 
[Definitions section](#1-Error-Messages). This avoids the need to include `<stdio.h>`
for `printf()`, strictly following our rule to minimize includes.

If everything succeeds, the `connect()` system call links our empty socket to the
i3 window manager, and we return the file descriptor so the rest of the program can
start reading or writing.

---
### 6. Function window_events_subscribe()
#### Source code
```c
int window_events_subscribe(int i3_fd_event)
{
        struct i3_ipc_header reply_header = {0};
        struct i3_ipc_header header = {
                .magic = I3_IPC_MAGIC,
                .size  = SUBSCRIBE_WINDOW_EVENT_PAYLOAD_LEN,
                .type  = I3_IPC_MESSAGE_TYPE_SUBSCRIBE
        };

        write(i3_fd_event, &header, sizeof(struct i3_ipc_header));
        write(i3_fd_event, SUBSCRIBE_WINDOW_EVENT_PAYLOAD, SUBSCRIBE_WINDOW_EVENT_PAYLOAD_LEN);

        if (readn(i3_fd_event, &reply_header, sizeof(struct i3_ipc_header)) <= 0)
                return -1;

        flush_reply(i3_fd_event, reply_header.size);

        return 0;
}
```
#### Explanation
In order for the i3 ipc to transmit the correct messages for our program through the
socket we need to request them by sending the correct message. We have it defined
[in the definitions](#3-Payloads) for readability. We use the `i3_ipc_header` that we
include from `<i3/ipc.h>`. We get it ready and then send it to the ipc and we flush the
reply since it is really improbable that we can't subscribe at this point.

Despite this, if our read for the reply by the ipc fails we return `-1`, indicating that
our connection broke.
A return value of `0` means a successful subscription to the window events.

---
### 7. Function read_single_window_event()
Here the source code is not given all at once since it is a very big code block
we will examine the code in snippets. The whole code is in the file `autotiling.c`.
```c
int read_single_window_event(int i3_fd_event, int *out_width, int *out_height)
{
        int result = EVENT_ERROR;

        char *is_focus = NULL;
        char *is_new   = NULL;
        char *rect_ptr = NULL;

        struct i3_ipc_header event_header = {0};
        char json_payload[4096] = {0};

        ssize_t n = readn(i3_fd_event, &event_header, sizeof(struct i3_ipc_header));
```
The first part are the variables the function needs and the initialization of
the local variables.
We need the file descriptor for the socket we use to read. the pointer to the width we
will output and the pointer to the height we will output.
Our initializations are a bit tricky.
We start by thinking that the function will fail. So result is set to `EVENT_ERROR`.
Then we have three `char` pointers, two that are checks and one that helps us navigate the
json string we are expecting.
The struct is self-explanatory as we saw in the previous function we use headers in the
i3 ipc.
We use a decently sized buffer to store the payload, and inside n we store the number
of bytes read while storing the header in our struct.
The next part are the conditions for the program to exit without parsing the payload.
```c
    if (n <= 0)
            goto out;

    result = EVENT_IGNORE;
    if (event_header.type != I3_IPC_EVENT_WINDOW) {
            flush_reply(i3_fd_event, event_header.size);
            goto out;
    }

    if (event_header.size >= sizeof(json_payload)) {
            flush_reply(i3_fd_event, event_header.size);
            goto out;
    }

    n = readn(i3_fd_event, json_payload, event_header.size);
    if (n <= 0)
            goto out;
```
Here by checking if `n <= 0`, we make sure that we read the header without errors.
Next we set result to `EVENT_IGNORE`, since we read without errors whatever happens next
is either the payload is relevant or not.
The check after that is a bit unnecessary but is there for extra safety.
The third if statement seems weird, if the size of the payload is bigger or equal to our
buffer we ignore the event. For reference the readme file in this repo is 3.6Kb, our
buffer has a 4Kb capacity, so such a big payload is very improbable and would cause an
overflow.(The reason we flush even if the payload is 4Kb is because we need to add the
null terminator too).
Because n stores the return value of a `readn()` call we reuse it this time for reading
the payload.
```c
    json_payload[event_header.size] = '\0';

    is_focus = strstr(json_payload, "\"change\":\"focus\"");
    if (!is_focus)
            is_focus = strstr(json_payload, "\"change\": \"focus\"");

    is_new = strstr(json_payload, "\"change\":\"new\"");
    if (!is_new)
            is_new = strstr(json_payload, "\"change\": \"new\"");

    if (!is_focus && !is_new)
            goto out;
```
In this part is the first check to see if the payload is relevant.
The only cases where the height, and the width of a window changes is if
we focus on a new window or if we focus on an existing window, that's what we
are looking for here. We check this in two attempts, first for json variants,
such as i3, that do not include a space after the colon. If this fails, we try
with a space after the colon.
Now it's time to extract the height and the width.
```c
    result = EVENT_FOCUS;
    rect_ptr = strstr(json_payload, "\"rect\":");
    if (rect_ptr != NULL) {
            char *width_ptr = strstr(rect_ptr, "\"width\":");
            char *height_ptr = strstr(rect_ptr, "\"height\":");

            if (width_ptr && height_ptr) {
                    *out_width = simple_atoi(width_ptr + 8);
                    *out_height = simple_atoi(height_ptr + 9);
            }
    }

out:
    return result;
```
The correct dimensions are inside `rect`, so we first point the `rect_ptr` to `rect`,
if it is not found we exit, if found then from there we search for the first instance of
`height` and `width`, then if both are not null with the correct offset we extract their
values.

---
### 8. Function send_i3_split_command()
#### Source code
```c
int send_i3_split_command(int i3_cmd_fd, const char *split_x)
{
        struct i3_ipc_header header = {
                .magic = I3_IPC_MAGIC,
                .size  = strlen(split_x),
                .type  = I3_IPC_MESSAGE_TYPE_COMMAND
        };
        struct i3_ipc_header reply_header;

        write(i3_cmd_fd, &header, sizeof(struct i3_ipc_header));
        write(i3_cmd_fd, split_x, header.size);

        if (readn(i3_cmd_fd, &reply_header, sizeof(struct i3_ipc_header)) <= 0)
                return -1;

        flush_reply(i3_cmd_fd, reply_header.size);

        return 0;
}
```
#### Explanation
In this function in the same way, we get a header ready to let i3 know that
the incoming message is a command. Then we have a choice between sending
a split horizontally or a split vertically command. After that we clean up
by flushing the reply.

---
### 9. Function handle_signal()
#### Source code
```c
volatile sig_atomic_t keep_running = 1;
void handle_signal(int sig)
{
        (void)sig;
        keep_running = 0;
}
```
#### Explanation
This is our signal handler, designed to catch termination signals
(like `SIGINT` when you press Ctrl+C, or `SIGTERM` from the system) so the program
can shut down gracefully instead of just crashing.
The variable `keep_running` is what keeps our main loop alive. We use `sig_atomic_t`
to guarantee that reading and writing this variable happens in a single,
uninterruptible instruction. We add `volatile` to force the compiler to read it directly
from memory every single time the `while` loop checks it, preventing aggressive
optimization from caching it in a CPU register.

Because all signal handlers require an integer parameter (the signal number),
but we don't actually need to know *which* signal we received to shut down, we
cast it to `void`. This is a trick to silence the compiler's "unused parameter"
warning, keeping our compilation clean without disabling safety flags.
Inside a signal handler, you are severely restricted in what functions you can safely
call (most standard library functions are not "async-signal-safe").
By doing nothing but flipping a single integer flag, we guarantee the handler will
never cause a deadlock or crash. It simply tells the `main()` function to break the
loop and handle the socket cleanup securely in the `closing:` block.

---
### 10. Function main()
#### Source code
*Look inside the file and see the code, all the functions used are showcased*
#### Explanation
The `main` function acts as the orchestrator, divided into four clear phases:
*1. Two-Socket Architecture*
We connect to i3 twice to keep the program single-threaded without needing
complex async I/O. `i3_fd_event` permanently blocks to listen for window events,
while `i3_cmd_fd` is a dedicated channel to send commands instantly.
*2. Signal Registration*
We use POSIX `sigaction()` to catch `SIGINT` and `SIGTERM`.
We use `memset()` to zero out the struct, preventing garbage memory from
corrupting our flags.
*3. The Autotiling Loop*
The `while` loop blocks until i3 reports an event. When a new window gets focus,
the core autotiling math applies: if the window is wider than it is tall
(`width > height`), we split horizontally (`split h`). Otherwise, we split vertically
(`split v`).
*4. Cleanup*
The `closing:` label provides a single, safe exit point. Whether stopped by a user
signal or a broken socket, the program always funnels here to close file descriptors
and return resources to the kernel.
