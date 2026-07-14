# Multi-Threaded HTTP/1.0 Web Server (Course Project)

## YOUTUBE LINK: https://youtu.be/MBcEc7KUmJk

## Project Structure

```
webserver/
├── server.py          # the entire server code (~250 lines with comments)
└── www/               # static file root
    ├── index.html     # main page
    ├── pages/         # allowed subdirectory
    │   ├── about.html
    │   └── contact.html
    ├── css/           # allowed subdirectory
    │   └── style.css
    ├── img/           # allowed subdirectory
    │   └── logo.png
    └── private/       # FORBIDDEN subdirectory (returns 403)
        └── secret.txt
```

## Running the Server

```bash
python3 server.py          # port 8080 by default
python3 server.py 9000     # or any port of your choice
```

Open in a browser: http://localhost:8080/

## Assignment Requirements

| Check | Command | Expected Result |
|---|---|---|
| Main page | `curl.exe -v http://localhost:8080/` | 200 OK, HTML |
| File in a subdirectory | `curl.exe -v http://localhost:8080/pages/about.html` | 200 OK |
| CSS with correct MIME type | `curl.exe -v http://localhost:8080/css/style.css` | 200, `text/css` |
| Image | `curl.exe -v http://localhost:8080/img/logo.png` | 200, `image/png` |
| Directory traversal | `curl.exe -v --path-as-is http://localhost:8080/../../etc/passwd` | **403 Forbidden** |
| Forbidden subdirectory | `curl.exe -v http://localhost:8080/private/secret.txt` | **403 Forbidden** |
| Non-existent file | `curl.exe -v http://localhost:8080/nope.html` | **404 Not Found** |
| Non-GET method | `curl.exe -v -X POST http://localhost:8080/index.html` | **405 Method Not Allowed** |
| Bad request | `python -c "import socket; s=socket.create_connection(('localhost',8080)); s.sendall(b'GARBAGE\r\n\r\n'); print(s.recv(200).decode()); s.close()"` | **400 Bad Request** |
| Concurrency | open the page in 2+ tabs simultaneously | both load |

## How It Works

**1. Socket lifecycle.** `socket()` → `bind((host, port))` → `listen()` →
an `accept()` loop. Each accepted connection is a new client socket object.

**2. Buffered stream reading.** TCP does not preserve message boundaries: a
request may arrive one byte at a time. The `read_request_head()` function calls
`recv()` in a loop and accumulates bytes until the buffer contains `\r\n\r\n` —
the end-of-headers marker. This is a key requirement of the assignment, and it
is verified by a test that sends the request in three chunks with pauses.

**3. Request parsing.** The first line (`GET /path HTTP/1.0`) is split on
spaces into exactly 3 parts: method, path, and version. Any deviation → 400.

**4. Security (two layers).**
   - An explicit check for `..` in the path → 403 (assignment requirement).
   - A safety net: `os.path.realpath()` normalizes the path, and if the
     resulting path ends up outside `www/`, the server also returns 403. This
     catches trickier escape attempts that don't contain a literal `..`.
   - A whitelist `ALLOWED_SUBDIRS = {"pages", "css", "img"}`: files in the
     `www/` root and in these subdirectories are accessible; everything else
     (e.g. `/private/`) → 403.

**5. Response format.** Strictly per the protocol: status line
(`HTTP/1.0 200 OK\r\n`), the `Content-Type` and `Content-Length` headers,
`Connection: close`, a blank line `\r\n`, then the raw bytes of the file.

**6. Stateless.** The connection is closed after every response
(`Connection: close` + `conn.close()`). No cookies, no sessions.

**7. Concurrency.** The main thread only accepts connections; each client is
served in its own `threading.Thread(daemon=True)`. A slow client occupies only
its own thread and does not block the others (no head-of-line blocking).
Additionally, a 10-second timeout is set on the client socket so that a hung
client cannot hold a thread forever.

## Likely Questions During the Defense

- **Why a loop around recv()?** TCP is a byte stream, not a sequence of
  messages; one send by the client may turn into several `recv()` calls and
  vice versa.
- **Why daemon threads?** So the server can shut down on Ctrl+C without
  waiting for all clients to finish.
- **Why SO_REUSEADDR?** After the server stops, the port lingers in the
  TIME_WAIT state for a while; without this option, an immediate restart fails
  with "Address already in use".
- **How does threading compare to a thread pool or asyncio?** Thread-per-client
  is simple, but expensive with thousands of clients (each thread's stack costs
  memory). A pool caps resource usage; asyncio scales best of all, but the code
  is more complex.
