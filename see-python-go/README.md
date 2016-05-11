# See Python, See Python Go, Go Python Go

Versions:
* Go 1.6
* Python 3.5


## Running a webserver in Go

```go
package main

import (
    "fmt"
    "net/http"
)

func index(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "Hello, world.\n")
}

func main() {
    http.HandleFunc("/", index)
    http.ListenAndServe("127.0.0.1:5000", nil)
}
```

## Running a webserver in Python

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return 'Hello, world!\n'

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

## Running a Go webserver in Python??

```python
from gohttp import route, run

@route('/')
def index(w, req):
    w.write("Hello, world.\n")

if __name__ == '__main__':
    run(host='127.0.0.1', port=5000)
```

## Yo--whaa???

That's right.


## How?

At first, Go was created to be run as a single statically linked binary
process. Later, more *execution modes* were added to let us compile Go as
dynamically linked binaries. In Go 1.5, additional modes were added to allow us
to build Go code into shared library that is runnable from other
runtimes.

Just like how all kinds of Python modules like lxml use C to run
super-optimized code, you can now run Go code just the same. More or less.


## Considerations

### Runtime Overhead

When a Go library is used from another runtime, it spins up the Go runtime in
parallel with the caller's runtime (if any). That is, it gets the goroutine
threads and the garbage collector and all that other nice stuff you'd normally
boot up when calling Go on its own.

This is different than calling vanilla C code because technically there is no
innate C runtime involved. There is no default worker pools, no default garbage
collector. You might call a C library which has its own equivalents of this,
then all bets are off, but in most simple cases you get very little overhead
from calling C. This is something to consider when calling out to code that
requires its own runtime to co-exist.


### Runtime Boundaries

Moving memory (or objects) between runtime can be tricky and dangerous,
especially when garbage collectors are involved. Both Go and Python have their
own garbage collector. If you share the same memory pointer between the two
runtimes, one garbage collector might decide that it's no longer used and clear
the memory while the other runtime would be all "WHY'D YOU DO THAT??" and
crash. Or worse, it could try to move it around, or change the memory's layout
in a way that the runtimes disagree, then we'd get weird hard-to-diagnose
heisenbugs.

The safest thing to do is to copy data across boundaries when possible, or
treat it as immutable read-only data when it's too big to be copied.


### Runtime Demilitarized Zone

When mediating calls between two runtimes like Go and Python, we use C land in
between them as a kind of demilitarized zone because C has no runtime and we
can trust it to not mess with our data all willynilly.


## The Plan

In this proof-of-concept, we're going to use the Go webserver, but we want to
provide a Python handler that will get called when our route gets hit.

1. Make a Go webserver, easy peasy.

2. Make it into a module that is exported into a C shared library.

3. Add a handler registry bridge in C.

4. Add some helpers for calling Go interface functions from C.

5. Add headers for importing the shared library in Python.

6. Write our handler to use the C registry bridge in Python, and the helpers to
   interact with the data to create a response.


[Diagram]


## Calling Go from C

Ref: https://golang.org/cmd/cgo/#hdr-Passing_pointers


## Calling C from Python

### Python C ABI

Ref: https://docs.python.org/3/c-api/index.html

Python 3.2+ has a "Stable ABI" which allows us to compile foreign module code 
into binary libraries that are forward-compatible with Python versions which 
support the Stable ABI.


### CFFI

Ref: http://cffi.readthedocs.io/

Generates glue code for calling C code from Python. More magic, but less code
to write by hand.


## Useful resources


* https://github.com/shazow/gohttplib/
* https://blog.filippo.io/building-python-modules-with-go-1-5/
* https://github.com/golang/go/wiki/cgo


* https://feiskyer.github.io/2016/04/19/cgo-in-go-1-6/

* https://github.com/go-python/gopy
