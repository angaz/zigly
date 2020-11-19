![Zigly](logo.png)
========

This is Zigly. A library to write Compute@Edge services in pure Zig.

## What is Compute@Edge?

[Compute@Edge](https://www.fastly.com/products/edge-compute/serverless/) is [Fastly](https://fastly.com)'s service to run custom code directly on CDN nodes.

The service runs anything that can be compiled to WebAssembly, and exports a convenient set of functions to interact with the platform.

## What is Zig?

[Zig](https://ziglang.org) defines itself as "a general-purpose programming language and toolchain for maintaining robust, optimal, and reusable software".


Zig:

- Is very simple to learn
- Compiles quickly, providing a great developer experience
- Can compile and use existing C and C++ code at no cost
- Is way safer than C and C++ by design
- Prints nice and useful error traces, on all platfors, including WebAssembly.
- Comes with a rich standard library, avoiding the need for many external dependencies
- Has excellent support for WebAssembly
- Creates small, fast, standalone executables and WebAssembly modules

## What is Zigly?

Zigly is a library that makes it easy to write Compute@Edge modules in Zig.

It is a work in progress, but the entire set of exported functions is going to be supported soon.

## Usage

### A minimal WebAssembly program

```zig
const std = @import("std");

fn start() !void {
    std.debug.print("Hello from WebAssembly and Zig!\n", .{});
}

pub export fn _start() callconv(.C) void {
    start() catch unreachable;
}
```

The `_start()` function is important, and must have that exact type. No `main()` function is required in our case.
We simply use a distinct `start()` function in order to catch errors, since the `_start()` function, as expected by the WebAssembly interface, cannot handle Zig errors.

The program can be compiled with:

```sh
zig build-exe -Dtarget=wasm32-wasi
```

or by defining the target as

```zig
    const target = try std.zig.CrossTarget.parse(.{ .arch_os_abi = "wasm32-wasi" });
```

in the `build.zig` file, compile to WebAssembly by default.

Once testing has been done, compile with `-Drelease-small` or `-Drelease-fast` to get small, optimized modules.

### Testing Compute@Edge modules

The easiest way to test the resulting modules is currently to use [Fastlike](https://github.com/avidal/fastlike), a partial reimplementation of the Fastly API that runs locally, on any operating system supported by the Go language.

### Using Zigly

#### Hello world!

```zig
var downstream = try zigly.downstream();
var response = downstream.response;
try response.body.writeAll("Hello world!");
try response.finish();
```        

`downstream()` returns a type representing the initial connection, from a client to the proxy.

That type includes `response`, that can be used to send a response, as well as `request`, that can be used to inspect the incoming request.

Every function call may fail with an error from the `FastlyError` set.

#### Inspecting incoming requests

Applications can read the body of an incoming requests as well as other informations such as the headers:

```zig
const request = downstream.request;
const user_agent = try request.headers.get(&allocator, "user-agent");
if (request.is_post()) {
    // method is POST, read the body until the end
    const body = try request.body.readAll(&allocator);   
}
```

As usual in Zig, memory allocations are never hidden, and applications can choose the allocator they want to use for individual function calls.

#### Making HTTP queries

Making HTTP queries is easy:

```zig
var query = try Request.new("GET", "https://example.com");
var response = try query.send("backend");
const body = try response.body.readAll(&allocator);
```

Arbitrary headers can be added onto the outgoing `query`:

```zig
try query.headers.set("X-Custom-Header", "Custom value");
```

Body content can also be pushed, even as chunks:

```zig
try query.body.write("X");
try query.body.write("Y");
try query.body.close();
```

And the resulting `response` contains `headers` and `body` properties, that can be inspected the same way as a downstream query.

...

** Documentation in progress! **