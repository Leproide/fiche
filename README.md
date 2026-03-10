# Fiche — Bug Fixes

This document describes the bugs found and fixed in the original
[fiche](https://github.com/solusipse/fiche/) source code (`fiche.c`).

---

## Bug 1 — Stack overflow with large buffer sizes

### Location
`handle_connection()` — `fiche.c`

### Root cause
The receive buffer was declared as a **Variable Length Array (VLA) on the thread stack**:

```c
// Original — stack allocation
uint8_t buffer[c->settings->buffer_len];
```

The default value of `buffer_len` is already 32 768 bytes (32 KB).
When a larger value is passed via `-B` (e.g. 10 MB), the thread stack overflows
immediately on entry, before any data is received. The crash is silent: the OS
kills the thread with no error message, no URL is returned, and no file is
created.

VLAs on the stack are also flagged as undefined behaviour by `-Wpedantic` when
the size is not a compile-time constant.

### Fix
Replaced the VLA with a **heap allocation** via `calloc`, with proper cleanup on
every exit path:

```c
// Fixed — heap allocation
uint8_t *buffer = calloc(c->settings->buffer_len, sizeof(uint8_t));
if (!buffer) {
    print_error("Couldn't allocate buffer!");
    close(c->socket);
    free(c);
    pthread_exit(NULL);
    return NULL;
}
// ... use buffer ...
free(buffer);   // released on every exit path
```

---

## Bug 2 — Connections silently dropped with partial data

### Location
`handle_connection()` — `fiche.c`

### Root cause
The original code combined `MSG_WAITALL` with `SO_RCVTIMEO`:

```c
// Original
const struct timeval timeout = { 5, 0 };
setsockopt(s, SOL_SOCKET, SO_RCVTIMEO, &timeout, ...);
// ...
const int r = recv(c->socket, buffer, sizeof(buffer), MSG_WAITALL);
if (r <= 0) { /* discard everything */ }
```

`MSG_WAITALL` instructs the kernel to block until **exactly** `buffer_len` bytes
are received. The typical fiche client (`cat file | nc host 9999`) sends N bytes
where N < `buffer_len` and then **closes the connection**.

On Linux, when `SO_RCVTIMEO` is set and the client closes before filling the
buffer, `recv` may return either:

- `-1` with `errno = EAGAIN` (timeout fired with partial data pending), or
- `0` (client closed the connection cleanly before the timeout).

Both cases satisfy `r <= 0`, causing the thread to **discard all received data,
close the socket without responding, and exit** — as if nothing had been
received. The paste is never saved and the client gets no URL back.

### Fix
Removed `MSG_WAITALL` and replaced the single `recv` call with a **read loop**
that accumulates data until the client closes the connection or the buffer is
full:

```c
// Fixed — read loop, no MSG_WAITALL
ssize_t r = 0, chunk;
while ((chunk = recv(c->socket, buffer + r,
                     c->settings->buffer_len - (size_t)r, 0)) > 0) {
    r += chunk;
    if ((size_t)r >= c->settings->buffer_len) break;
}
if (r <= 0) { /* truly no data received */ }
```

The loop exits naturally when the client closes the TCP connection (EOF →
`recv` returns 0), regardless of whether the buffer was filled. The 5-second
`SO_RCVTIMEO` is still in place and still works correctly as a stall guard.

The return type was also changed from `int` to `ssize_t` to match the return
type of `recv`, and the corresponding `printf` format specifier was updated from
`%d` to `%zd` for correctness under `-Wpedantic`.

---

## Bug 3 — Use-after-free on slug generation failure

### Location
`handle_connection()` — `fiche.c`, inside the slug generation loop

### Root cause
In the error path triggered when the slug generation loop exceeds 128 attempts,
the original code called `free(c)` before `close(c->socket)`:

```c
// Original — use-after-free
free(c);
free(slug);
close(c->socket);   // c->socket read after c was freed
```

`c->socket` is a field of the struct pointed to by `c`. Reading it after
`free(c)` is undefined behaviour, detected by GCC with `-Wuse-after-free`.

### Fix
Reordered the cleanup so the socket is closed before the struct is freed:

```c
// Fixed
free(buffer);
free(slug);
close(c->socket);   // socket closed while c is still valid
free(c);
```

---

## Summary

| # | Location | Type | Effect |
|---|----------|------|--------|
| 1 | `handle_connection` | Stack overflow (VLA) | Crash / silent thread death with large `-B` values |
| 2 | `handle_connection` | `MSG_WAITALL` + `SO_RCVTIMEO` | Data discarded, no URL returned, no file saved |
| 3 | `handle_connection` | Use-after-free | Undefined behaviour on slug generation failure |

# Delete a paste
Use case: remove a paste you uploaded — requires the delete token returned at upload time

# Linux / macOS
echo -e "slug\ntoken" | nc termbin.huginn.ovh 9998

Change the delete port with -P parameter

=====


fiche [![Build Status](https://travis-ci.org/solusipse/fiche.svg?branch=master)](https://travis-ci.org/solusipse/fiche)
=====

Command line pastebin for sharing terminal output.

# Client-side usage

Self-explanatory live examples (using public server):

```
echo just testing! | nc termbin.com 9999
```

```
cat file.txt | nc termbin.com 9999
```

In case you installed and started fiche on localhost:

```
ls -la | nc localhost 9999
```

You will get an url to your paste as a response, e.g.:

```
http://termbin.com/ydxh
```

You can use our beautification service to get any paste colored and numbered. Just ask for it using `l.termbin.com` subdomain, e.g.:

```
http://l.termbin.com/ydxh
```

-------------------------------------------------------------------------------

## Useful aliases

You can make your life easier by adding a termbin alias to your rc file. We list some of them here:

-------------------------------------------------------------------------------

### Pure-bash alternative to netcat

__Linux/macOS:__
```
alias tb="(exec 3<>/dev/tcp/termbin.com/9999; cat >&3; cat <&3; exec 3<&-)"
```

```
echo less typing now! | tb
```

_See [#42](https://github.com/solusipse/fiche/issues/42), [#43](https://github.com/solusipse/fiche/issues/43) for more info._

-------------------------------------------------------------------------------

### `tb` alias

__Linux (Bash):__
```
echo 'alias tb="nc termbin.com 9999"' >> .bashrc
```

```
echo less typing now! | tb
```

__macOS:__

```
echo 'alias tb="nc termbin.com 9999"' >> .bash_profile
```

```
echo less typing now! | tb
```

-------------------------------------------------------------------------------

### Copy output to clipboard

__Linux (Bash):__
```
echo 'alias tbc="netcat termbin.com 9999 | xclip -selection c"' >> .bashrc
```

```
echo less typing now! | tbc
```

__macOS:__

```
echo 'alias tbc="nc termbin.com 9999 | pbcopy"' >> .bash_profile
```

```
echo less typing now! | tbc
```

__Remember__ to reload the shell with `source ~/.bashrc` or `source ~/.bash_profile` after adding any of provided above!

-------------------------------------------------------------------------------

## Requirements
To use fiche you have to have netcat installed. You probably already have it - try typing `nc` or `netcat` into your terminal!

-------------------------------------------------------------------------------

# Server-side usage

## Installation

1. Clone:

    ```
    git clone https://github.com/solusipse/fiche.git
    ```

2. Build:

    ```
    make
    ```
    
3. Install:

    ```
    sudo make install
    ```

### Using Ports on FreeBSD

To install the port: `cd /usr/ports/net/fiche/ && make install clean`. To add the package: `pkg install fiche`.

_See [#86](https://github.com/solusipse/fiche/issues/86) for more info._

-------------------------------------------------------------------------------

## Usage

```
usage: fiche [-D6epbsdSolBuw].
             [-d domain] [-L listen_addr ] [-p port] [-s slug size]
             [-o output directory] [-B buffer size] [-u user name]
             [-l log file] [-b banlist] [-w whitelist] [-S]
```

These are command line arguments. You don't have to provide any of them to run the application. Default settings will be used in such case. See section below for more info.

### Settings

-------------------------------------------------------------------------------

#### Output directory `-o`

Relative or absolute path to the directory where you want to store user-posted pastes.

```
fiche -o ./code
```

```
fiche -o /home/www/code/
```

__Default value:__ `./code`

-------------------------------------------------------------------------------

#### Domain `-d`

This will be used as a prefix for an output received by the client.
Value will be prepended with `http`.

```
fiche -d domain.com
```

```
fiche -d subdomain.domain.com
```

```
fiche -d subdomain.domain.com/some_directory
```

__Default value:__ `localhost`

-------------------------------------------------------------------------------

#### Slug size `-s`

This will force slugs to be of required length:

```
fiche -s 6
```

__Output url with default value__: `http://localhost/xxxx`,
where x is a randomized character

__Output url with example value 6__: `http://localhost/xxxxxx`,
where x is a randomized character

__Default value:__ 4

-------------------------------------------------------------------------------

#### HTTPS `-S`

If set, fiche returns url with https prefix instead of http

```
fiche -S
```

__Output url with this parameter__: `https://localhost/xxxx`,
where x is a randomized character

-------------------------------------------------------------------------------

#### User name `-u`

Fiche will try to switch to the requested user on startup if any is provided.

```
fiche -u _fiche
```

__Default value:__ not set

__WARNING:__ This requires that fiche is started as a root.

-------------------------------------------------------------------------------

#### Buffer size `-B`

This parameter defines size of the buffer used for getting data from the user.
Maximum size (in bytes) of all input files is defined by this value.

```
fiche -B 2048
```

__Default value:__ 32768

-------------------------------------------------------------------------------

#### Log file `-l`

```
fiche -l /home/www/fiche-log.txt
```

__Default value:__ not set

__WARNING:__ this file has to be user-writable

-------------------------------------------------------------------------------

#### Ban list `-b`

Relative or absolute path to a file containing IP addresses of banned users.

```
fiche -b fiche-bans.txt
```

__Format of the file:__ this file should contain only addresses, one per line.

__Default value:__ not set

__WARNING:__ not implemented yet

-------------------------------------------------------------------------------

#### White list `-w`

If whitelist mode is enabled, only addresses from the list will be able
to upload files.

```
fiche -w fiche-whitelist.txt
```

__Format of the file:__ this file should contain only addresses, one per line.

__Default value:__ not set

__WARNING:__ not implemented yet

-------------------------------------------------------------------------------

### Running as a service

There's a simple systemd example:
```
[Unit]
Description=FICHE-SERVER

[Service]
ExecStart=/usr/local/bin/fiche -d yourdomain.com -o /path/to/output -l /path/to/log -u youruser

[Install]
WantedBy=multi-user.target
```

__WARNING:__ In service mode you have to set output directory with `-o` parameter.

-------------------------------------------------------------------------------

### Example nginx config

Fiche has no http server built-in, thus you need to setup one if you want to make files available through http.

There's a sample configuration for nginx:

```
server {
    listen 80;
    server_name mysite.com www.mysite.com;
    charset utf-8;

    location / {
            root /home/www/code/;
            index index.txt index.html;
    }
}
```

## License

Fiche is MIT licensed.
