---
title: Implementing a mini-mini Flask
summary: A small http server from scratch
date: "2025-09-18"
tags:
    - Python
---

There is a unique energy and satisfaction that comes from peeling back the layers of abstraction to understand how things really work. It is not just about building a tool; it is about the journey of discovery it.

I recently subscribed to CodeCrafters as I was quite hooked by their service. This platform offers a different kind of challenge. Instead of the classical gamified algorithmic puzzles, it invites you to build foundational technologies like an HTTP server, Git, Redis, or Kafka, from the ground up. The cherry on the top of the cake? You can tackle these in a wide array of programming languages.

The process is very straightforward yet very engaging. You are given a set of feature requirements ("here's the input," "we expect this output") and set free to implement them. Then, you run your code against a comprehensive test suite. It is a practical and empowering experience that deserves genuine kudos.

>it is still very expensive though.

## A HTTP server from scratch

My journey began with the HTTP server challenge, tasked with implementing one according to HTTP 1.1 specifications. The starting point was a simple `main.py` file. I started with a straightforward approach: a `socket` server that parsed incoming request `bytes` and used a long `if...else` block to determine the response.


```python
# app/main.py
import socket

from dataclasses import dataclass

CLRF = "\r\n"

@dataclass
class RequestStatus:
    method: str
    target: str
    version: str


@dataclass
class ResponseStatus:
    version: str
    code: int
    msg: str


@dataclass
class Response:
        status: ResponseStatus
        headers: dict
        body: str


def format_response(resp: Response):
    status = f"{resp.status.version} {resp.status.code} {resp.status.msg}{CLRF}"
    headers = "".join([f"{k}: {v}" for k, v in resp.headers.items()])
    body = f"{CLRF}{resp.body}"
    return status + headers + body


def process_request_line(data: bytes):
    return data.decode().split(CLRF)[0].split()


def main():
    server_socket = socket.create_server(("localhost", 4221), reuse_port=True)
    connexion, addr = server_socket.accept()  # wait for client

    with connexion as c:
        data = c.recv(1024)

        if not data:
            raise ValueError

        status_line = RequestStatus(*process_request_line(data=data))

        if status_line.target == "/":
            c.send(f"HTTP/1.1 200 OK{CLRF}{CLRF}".encode())
        elif status_line.target.startswith("/echo/"):
            content_length = len(status_line.target) - len("/echo/")
            status = ResponseStatus(version="HTTP/1.1", code=200, msg="OK")
            headers = {
                "Content-Type": f"text/plain{CLRF}",
                "Content-Length": f"{content_length}{CLRF}"
            }
            body = status_line.target[-content_length:]
            resp = Response(status=status, headers=headers, body=body)
            c.send(format_response(resp).encode())
        else:
            c.send(f"HTTP/1.1 404 Not Found{CLRF}{CLRF}".encode())


if __name__ == "__main__":
    main()
```

It worked, but clearly something felt off. The code was not elegant, and the branching logic was a clear sign that this approach would not scale gracefully with additional features. This is where I find myself thinking like battle between the "ok it works" and "Now it works well." Such tension is usually an invitation to learn, to refactor, and to find a path that honors both functionality and code craft.

## A brief overview of the `socket` library

But before we dive into the refactor, let's gently demystify the core technology enabling this: the Python `socket` library. Socket programming is the art of handling interprocess communication (IPC), allowing data to flow between processes, whether on the same machine or across the full network.

In Python the API is elegantly simple. Key functions like `socket()`, `.bind()`, `.listen()`, `.accept()`, `.connect()`, `.connect_ex()`, `.send()`, `.recv()` and `.close()` permits you to handle the network. Creating a server that listens for a client connection is very straightforward and not complicated:

```python
socket.create_server(("localhost", <your_port: int>), reuse_port=True)
connexion, addr = server_socket.accept()  # wait for client
```

The `create_sever()` method returns the `socket` and `_RetAddress` objects. You can next interact with the `socket` object in a context manager.

```python
connexion, addr = server_socket.accept()  # wait for client

with connexion as c:
    # collect data sent by the client
    data = c.recv(1024)

    # if no data you can raise or do whatever suits your business logic.
    if not data:
        raise ValueError
```

The client's request, a stream of bytes, follows the HTTP specification, in our case [HTTP1.1 specifications](https://datatracker.ietf.org/doc/html/rfc9112#name-message). Parsing the initial status line, containing the method, target endpoint, and version, is our first step. My initial implementation did this directly in the main loop:

```python
if status_line.target == "/":
    c.send(f"HTTP/1.1 200 OK{CLRF}{CLRF}".encode())
elif status_line.target.startswith("/echo/"):
    content_length = len(status_line.target) - len("/echo/")
    status = ResponseStatus(version="HTTP/1.1", code=200, msg="OK")
    headers = {
        "Content-Type": f"text/plain{CLRF}",
        "Content-Length": f"{content_length}{CLRF}"
    }
    body = status_line.target[-content_length:]
    resp = Response(status=status, headers=headers, body=body)
    c.send(format_response(resp).encode())
else:
    c.send(f"HTTP/1.1 404 Not Found{CLRF}{CLRF}".encode())
```

The initial `if...else` block was the trigger for change. It was functional but fragile to say the least. This is where the elegance of frameworks like Flask shines. They allow you to declare intent simply:

```python
@app.route("/my_endpoint")
def my_function():
    # business logic if any
    return your_response
```

I started thinking how I could implement something a bit similar. The goal was not to replicate Flask, I do not have the skills for that, but to include a bit of its philosophy and bring clarity and maintainability to my code.

## Turning raw bytes into a `Request` object

The first step was to encapsulate the raw byte parsing logic. Instead of slicing strings in the main `loop`, I created a `Request` class to handle the intricate details, providing a clean, attribute-based interface.

```python
# server/request.py
from dataclasses import dataclass

CLRF = "\r\n"

@dataclass
class Status: # the status line
    method: str
    target: str
    version: str


class Headers:
    def __init__(self, **kwargs):
        for k, v in kwargs.items():
            # dynamic assignments of the attributes based
            # on received headers
            setattr(self, k, v)


class Request:
    def __init__(self, data: bytes):
        self.process(data=data)

    def process(self, data: bytes) -> None:
        content = data.decode().split(CLRF)
        _status = content[0].split()
        _headers = content[1].split()
        _body = content[-1].split()

        headers = {}
        for header in _headers:
            key = header[0]
            value = header[1]
            headers[key] = value

        self.status = Status(*_status)
        self.headers = Headers(**headers)
        self.body = _body
```

Once instantieted, the `Request` object gives me nice encapsulation of the data recevied from the client. I can now ask for `Request.status.target` or `Request.headers.Host` instead of slicing strings. Here we model the problem domain instead of just manipulating data.

Next, I applied the same thinking to the response. Hand-crafting HTTP payloads in every branch was error-prone. By creating a `Response` class with a `__str__` dunder method, I centralized the HTTP formatting logic into a single, responsible interface.

```python
# server/response.py
from dataclasses import dataclass

CLRF = "\r\n"

@dataclass
class Status:
    version: str
    code: int
    msg: str


class Response:
    def __init__(self, status: Status, headers: dict, body: str):
        self.status = status
        self.headers = headers
        self.body = body

    def __str__(self):
        status = f"{self.status.version} {self.status.code} {self.status.msg}{CLRF}"
        headers = "".join([f"{k}: {v}" for k, v in self.headers.items()])
        body = f"{CLRF}{self.body}"
        return status + headers + body
```

Returning the stringified response keeps handlers readable and centralizes the knowledge about HTTP formatting in one place. It ensures consistency and turns response generation from a chore into a simple, declarative act.

## Now let's decorate routes a bit like in Flask

The biggest ergonomics gain is the implementation of the routing system. A decorator registers the route and the function in a global table, keeping URL declarations close to the functions that serve them.

```python
# server/routes.py
from .request import Request
from .response import Status, Response


ROUTES = {}
CLRF = "\r\n"

def route(path):
    def decorator(func):
        ROUTES[path] = func
        return func
    return decorator
```

With this, defining a new endpoint became a moment of clarity:

```python
@route("/")
def index(req: Request):
    return f"HTTP/1.1 200 OK{CLRF}{CLRF}".encode()


@route("/echo/abc")
def echo(req: Request):
    length = len(req.status.target) - len("/echo/")
    status = Status(version="HTTP/1.1", code=200, msg="OK")
    headers = {
        "Content-Type": f"text/plain{CLRF}",
        "Content-Length": f"{length}{CLRF}"
    }
    body = req.status.target[-length:]
    return str(Response(status=status, headers=headers, body=body))
```

## A slimmer entry point

With parsing and routing abstracted away the main loop can be transformed from a set of conditionals into a clear dispatcher:

```python
# main.py
import socket  # noqa: F401

from server import routes, request

CLRF = "\r\n"

def main():
    server_socket = socket.create_server(("localhost", 4221), reuse_port=True)
    connexion, _ = server_socket.accept()  # wait for client

    with connexion as c:
        data = c.recv(1024)

        if not data:
            raise ValueError

        req = request.Request(data=data)
        handler = routes.ROUTES.get(req.status.target, None)

        if handler is None:
            c.send(f"HTTP/1.1 404 Not Found{CLRF}{CLRF}".encode())
        else:
            resp = handler(req)
            c.send(resp.encode())


if __name__ == "__main__":
    main()
```

The complex `if...else` block vanished, replaced by a simple dictionary lookup. The system became more powerful by becoming simpler.

## From static to dynamic routing

Of course, reality quickly presented a new lesson in humility. The challenge required a dynamic endpoint (`/echo/<some_string>`), and my simple dictionary could not handle that. My initial design, while cleaner, was still incomplete.

>A challenge is not a failure; it is information. It tells us how to adapt.

The solution I chose was the following: replace the dictionary with a list of tuples where I paired a compiled regex pattern with its handler function. I kept an edge case for the `"/"` endpoint.

```python
ROUTES = []
CLRF = "\r\n"

def router(path):
    pattern = (
        re.compile("^"+path+"$")
        if path == "/"
        else re.compile(path)
    )
    def decorator(func):
        ROUTES.append((pattern, func))
        return func
    return decorator
```

The dispatch logic evolved into a pattern-matching loop:

```python
# code before is unchanged
handler = None
for pattern, func in routes.ROUTES:
   if re.match(pattern, req.status.target):
       handler = func

if handler is None:
   c.send(f"HTTP/1.1 404 Not Found{CLRF}{CLRF}".encode())
else:
   resp = handler(req)
   c.send(resp.encode())
```

With this change I managed to somehow unlock dynamic routing and made my code a bit more flexible. In term of efficiency we unfortunately move from `O(1)` (dictonnary lookup) to `O(n)`. But it is fine usually the number of endpoint is not huge. So while the efficiency of looking up in routes will depend of the length of the list, it will stay pretty small.

## Wrap up

We moved from manipulating bytes very roughly to some modeling concepts, from procedural branching to declarative routing. But the journey is not over yet. The next steps— include the implentation of concurrency and compression. This promises new lessons. I am genuinely excited to continue and share what comes next. There is a clearly certain satisfaction in building something with your own hands, even small, and understanding the layers behind the tools we use every day.
