---
title: Sanic Custom Route Parameters
date: 2023-01-23T12:11:33Z
draft: false
categories:
- Sanic 
- til
---

Sanic supports path parameters like the majority of web frameworks. It supports a number of [built-in types](https://sanic.dev/en/guide/basics/routing.html#path-parameters) but you can define your own. 

Something like this from Adam on Discord:

```python
import re
from typing import NamedTuple


DIMENSIONS_PATTERN = re.compile(r"(?P<width>\d+)x(?P<height>\d+)")


class Dimensions(NamedTuple):
    width: int
    height: int


def _extract_dimensions(value: str) -> Dimensions:
    if match := DIMENSIONS_PATTERN.match(value):
        return Dimensions(**{k: int(v) for k, v in match.groupdict().items()})
    raise ValueError("Not proper dimensions format")


app.router.register_pattern(
    "dimensions", _extract_dimensions, DIMENSIONS_PATTERN
)


@app.get("/<dimensions:dimensions>/colors/<hex:ext>")
async def handler(
    request: Request, dimensions: Dimensions, hex: str, ext: str
):
    return json({"dimensions": dimensions, "hex": hex, "ext": ext})
```
