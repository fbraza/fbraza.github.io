---
title: Mocking in unittest
summary: An approach to mocking
date: "2025-07-17"
tags:
    - Python
---

>Can we make our tests fast, reliable, and still human-friendly?

In other words, how do we keep the craft while shaving off the friction? When our code reaches across the network, mocking is often the simplest way to reclaim speed, determinism, and focus without pretending the world is less complex than it is. I will try here to summarize and show some mocking approach I practiced and saw while working at Sunrise. For the sake of the demonstration we will use a practical guide to mocking HTTP clients with `unittest.mock`.

## Our use case

Let’s anchor everything around a tiny, real client. Something small enough to hold in my head, but realistic enough:

```python
# client.py
import requests

class Client:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.bootstrap = requests.get(f"{self.base_url}/bootstrap").json()

    def get_json(self, endpoint: str, **params):
        resp = requests.get(f"{self.base_url}/{endpoint}", params=params)
        return resp.json()
```

We want our tests to be: 

- Fast: not waiting for DNS, TLS handshakes, or remote servers.
- Deterministic: we control the responses, every time.
- Focused: when a test fails, it tells us about our logic not that the network is out or we are facing a rate limit.
     
To make a long story short, mocking lets us replace `requests.get` with a controllable piece of code. 

>That being said, we will also peek at dependency injection and fixture patterns, because sometimes the cleanest test is the one that does not need patching at all!

## Patch once, return `JSON` 

Most of the time, you just need: “When my code calls `requests.get(...).json()`, give me this.” 

Here’s how:

```python
# test_client.py
import pytest
from unittest.mock import patch

from client import Client

@patch("client.requests.get")
def test_client_bootstrap(mock_get):
    mock_get.return_value.json.return_value = {"ok": True}

    c = Client("http://api.test")
    assert c.bootstrap == {"ok": True}

    mock_get.assert_called_once_with("http://api.test/bootstrap")
```

I have to say that at first mocking felt like some dark magic. Reading at the code it is not really obvious why this works. Especially in Python where a lot of abstraction are at play here. Let’s try to break it down:

1. First `requests.get(...)` is replaced by `mock_get`
2. `.return_value` stands for the `Response` object
3. `.json.return_value` is our controlled `JSON` payload

It is a chain: mock → response → json → payload. Once you see it, it stops feeling magical and starts feeling bit more mechanical.

But let’s try to bring a bit more of clarity and expressiveness. We can actually spell out the response object. This may helps your future-you or your teammate at 2 AM.

```python
from unittest.mock import Mock, patch

@patch("client.requests.get")
def test_client_bootstrap_explicit(mock_get):
    resp = Mock()
    resp.json.return_value = {"ok": True}
    mock_get.return_value = resp

    c = Client("http://api.test")
    assert c.bootstrap == {"ok": True}
```

Let’s break it down:

1. `resp = Mock()`, we create the fake `Response` object
2. `resp.json.return_value = {"ok": True}`, we define what `.json()` returns
3. `mock_get.return_value = resp` , we Tell `mock_get` to return this object

This exercise may help understanding what is happening under the hood when reading this code:

```python
mock_get.return_value.json.return_value = {"ok": True}
```

## What if we have multiple `requests.get` calls?

Our client calls the `requests.get` method two times. At client instantiation with the `self.bootstrap` and if needed by using the method `get_json()`. Now if I want to test and mock the `get_json()` method I will have to be careful. We will need distinct mocked responses in order.

```python
from unittest.mock import Mock, patch

@patch("client.requests.get")
def test_get_json_two_calls(mock_get):
    mock_get.side_effect = [
        Mock(json=lambda: {"ok": True}),                    # for bootstrap
        Mock(json=lambda: {"id": 1, "name": "Ada Lovelace"}) # for endpoint call
    ]

    c = Client("http://api.test")
    result = c.get_json("users", id=1)

    assert result["name"] == "Ada Lovelace"
    assert mock_get.call_count == 2
```

Think of `side_effect` as a tiny queue of your responses. First call → first item. Second call → second item. It is important to understand your code and anticipate on any call that may occur at initialization or by other functions.

## Some common pitfalls avoid

### 1. Patching the wrong import path 

Patch where the function is looked up, not where it’s defined. 

- If `client.py` does `import requests` → patch `client.requests.get`
- If it does `from requests import get` → patch `client.get`
     
Python import always drove me crazy. With the mocking layer it drove me crazier. Knowing may save you some time.

### 2. Return-chain confusion

```python
# ❌ Won’t work — .json() is a method, not a property
mock_get.return_value = {"data": 42}

# ✅ Works — mock the method’s return value
mock_get.return_value.json.return_value = {"data": 42}
```
### 3. Wrong number or order with `side_effect`

If you make 2 calls but provide only 1 item? You will hit StopIteration. Keep your lists aligned with same length, same order as the calls you expect.

## A small detour with dependency injection

Mocking is great. But sometimes, the cleanest test is the one that does not need a patch at all. Injecting a tiny `requester` dependency can help. Let’s modify our client.

```python
from typing import Protocol, Any

class Requester(Protocol):
    def get_json(self, url: str, **kwargs) -> dict[str, Any]: ...

class RequestsRequester:
    def get_json(self, url: str, **kwargs):
        import requests
        return requests.get(url, **kwargs).json()

class Client:
    def __init__(self, base_url: str, requester: Requester = None):
        self.base_url = base_url
        self.req = requester or RequestsRequester()
        self.bootstrap = self.req.get_json(f"{self.base_url}/bootstrap")

    def get_json(self, endpoint: str, **params):
        return self.req.get_json(f"{self.base_url}/{endpoint}", params=params)
```

Now our test have predictable objects:

```python
class FakeRequester:
    def __init__(self, mapping):
        self.mapping = mapping
        self.calls = []

    def get_json(self, url: str, **kwargs):
        self.calls.append((url, kwargs))
        return self.mapping[url]

def test_di_client_without_patching():
    fake_data = {
        "http://api.test/bootstrap": {"ok": True},
        "http://api.test/users": {"id": 1}
    }
    c = Client("http://api.test", requester=FakeRequester(fake_data))
    assert c.bootstrap == {"ok": True}
    assert c.get_json("users") == {"id": 1}
```

That being said I would not abstract prematurely. Inject dependencies when they materially improve test ergonomics or production flexibility. Otherwise? A simple patch is perfectly fine. (And often, very elegant.)

## Warp up

Mocking is a tool not a dogma. So use real calls when: 

- Integration tests: you want to validate wiring, contracts, or server behavior.
- End-to-end tests: you’re exercising the full path, intentionally.
- Testing your own logic: mock the boundary, not the code under test.
     

Mocking the world does not necessarily make your code better. Mocking the right content does. 
