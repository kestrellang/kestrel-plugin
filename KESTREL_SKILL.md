# Kestrel Skill

Use this guide when writing, editing, reviewing, or explaining Kestrel source
code. Kestrel is a statically typed, expression-oriented systems language with
Swift-like syntax, copy-by-default value semantics, explicit
borrowing/mutating/consuming parameter modes, RAII cleanup, nominal structs and
enums, protocol-constrained generics, and pattern matching.

Prefer these rules over assumptions from Swift, Rust, Kotlin, TypeScript, or C.

# Install And Tools

Install Kestrel with the official install script:

```sh
curl -fsSL https://kestrel-lang.com/install.sh | sh
jessup install stable
kestrel --version
flock --help
```

Jessup manages compiler toolchains:

```sh
jessup install stable
jessup install nightly
jessup install 0.15.0
jessup list
jessup show
jessup default stable-0.15.0
jessup update
```

Flock is Kestrel's package manager:

```sh
flock init
flock check
flock build
flock run
flock update
flock publish
```

Install the official VS Code extension by searching for **Kestrel** in the
Extensions sidebar, or with:

```sh
code --install-extension kestrel-lang.kestrel
```

If installing from a downloaded release artifact, use:

```sh
code --install-extension path/to/kestrel-<target>.vsix
```

To install from the latest GitHub release:

```sh
target="$(case "$(uname -s)-$(uname -m)" in
  Darwin-arm64) echo darwin-arm64 ;;
  Darwin-x86_64) echo darwin-x64 ;;
  Linux-x86_64) echo linux-x64 ;;
  *) echo unsupported; exit 1 ;;
esac)"
gh release download --repo kestrellang/kestrel-vscode --pattern "kestrel-${target}.vsix" --clobber
code --install-extension "kestrel-${target}.vsix"
```

Open a `.ks` file or a folder containing `flock.toml`. The extension uses
`kestrel-lsp` from `PATH`; Jessup installs and links it with the active
toolchain.

Minimal `flock.toml`:

```toml
[package]
name = "my-package"
version = "0.1.0"
description = ""
author = ""
license = ""
repository = ""

[dependencies]
"kestrel/quill" = "^0.1.0"
```

# Core Syntax

Use `let` for immutable bindings and `var` for mutable bindings. Semicolons are
required after statements.

```kestrel
let answer: Int64 = 42;
let message = "hello";
var count: Int64 = 0;
count = count + 1;
```

Functions use `func`. A block's final expression can be the return value.
Single-expression functions can use `=`.

```kestrel
func add(x: Int64, y: Int64) -> Int64 {
    x + y
}

func multiply(x: Int64, y: Int64) -> Int64 = x * y
func identity[T](value: T) -> T = value
```

Generic arguments use square brackets:

```kestrel
struct Box[T] {
    var value: T;
}

let box: Box[Int64] = Box(value: 42);
```

# Labels And Access Modes

Single-name parameters are positional at the call site. This is unlike Swift.

```kestrel
func add(x: Int64, y: Int64) -> Int64 { x + y }
add(1, 2);
```

To require a call-site label, write an external label before the internal name:

```kestrel
func send(to recipient: String) { }
send(to: "alice@example.com");
```

Access modes come before the external label and internal name:

```kestrel
func read(p: Point) -> Int64 { p.x }          // borrowing, default, read-only
func reset(mutating p: Point) { p.x = 0; }    // caller must pass mutable storage
func close(consuming f: File) { }             // takes ownership

func offset(mutating point p: Point, by delta: Int64) {
    p.x = p.x + delta;
}

var origin = Point(x: 0, y: 0);
offset(point: origin, by: 5);
```

`consuming` moves non-copyable values and copies copyable values.

# Methods And Self

Methods do not declare `self` as a parameter. `self` is implicit inside instance
methods.

```kestrel
struct Counter {
    var value: Int64;

    func current() -> Int64 { self.value }

    mutating func increment() {
        self.value = self.value + 1;
    }

    static func zero() -> Counter {
        Counter(value: 0)
    }
}
```

Do not write Rust-style receivers:

```kestrel
// Wrong:
func increment(self) { }
func increment(&self) { }
```

Receiver behavior follows the method modifier: no modifier borrows `self`,
`mutating` gives read-write `self`, and `consuming` takes ownership of `self`.

# Values, Ownership, And RAII

Most Kestrel values are copyable by default, including primitives, structs, and
enums. Types can opt out with `not Copyable`; those values use move semantics.

```kestrel
struct Point {
    var x: Int64;
    var y: Int64;
}

let p2 = p1;  // copied; both values remain valid

struct File: not Copyable {
    var handle: Int64;

    deinit {
        self.close();
    }
}

let f2 = f1;  // moved; f1 is no longer valid
```

Use `deinit` for RAII cleanup. Non-copyable fields make the containing type
non-copyable unless the type explicitly handles that behavior.

# Types

Common surface types:

- `Bool`, `String`
- `Int8`, `Int16`, `Int32`, `Int64`
- `UInt8`, `UInt16`, `UInt32`, `UInt64`
- `Float16`, `Float32`, `Float64`
- `()`, unit
- `!`, never
- `T?`, optional
- `[T]`, array
- `[K: V]`, dictionary
- `(A, B)`, tuple
- `(A, B) -> C`, function type

Examples:

```kestrel
let maybeName: String? = .None;
let numbers: [Int64] = [1, 2, 3];
let ages: [String: Int64] = [:];
let point: (Int64, Int64) = (10, 20);
let callback: (String) -> Bool = { it == "ok" };
```

Do not invent implicit conversions. Prefer explicit types when inference is
ambiguous, especially for empty arrays, `.None`, public APIs, and numeric
literals.

# Structs And Enums

Structs are nominal value types with stored properties, computed properties,
initializers, methods, static members, and `deinit`.

```kestrel
struct Size {
    var width: Int64;
    var height: Int64;
}

let size = Size(width: 80, height: 24);  // memberwise initializer
```

Custom initializer labels follow normal parameter-label rules:

```kestrel
init(width: Int64, height: Int64) { }           // Size(80, 24)
init(withWidth width: Int64, height: Int64) { } // Size(withWidth: 80, 24)
```

Enums are sum types. Cases can be plain, labeled, or positional.

```kestrel
enum Shape {
    case Circle(radius: Float64)
    case Rectangle(width: Float64, height: Float64)
    case Point
}

let shape = Shape.Circle(radius: 5.0);

enum Option[T] {
    case Some(T)
    case None
}

let value = Option.Some(42);
```

Recursive enums require `indirect`:

```kestrel
indirect enum Tree[T] {
    case Leaf(value: T)
    case Node(left: Tree[T], right: Tree[T])
}
```

# Pattern Matching And Control Flow

`match` is an expression. Arms use `=>`, not `case`, and all arms must produce
compatible types.

```kestrel
match shape {
    .Circle(radius: r) if r > 10.0 => "large circle",
    .Circle(radius: _) => "circle",
    .Rectangle(width: w, height: h) => "rectangle",
    .Point => "point"
}
```

Pattern matching supports enum, tuple, and struct destructuring, literals,
ranges, wildcards, bindings, guards, and `or` patterns.

```kestrel
if let .Some(value) = maybeValue {
    print(value);
}

guard let .Some(value) = maybeValue else {
    return;
}

while let .Some(item) = iterator.next() {
    process(item);
}

let sign = if x > 0 { 1 } else if x < 0 { -1 } else { 0 };
```

# Closures

Closures are first-class values and capture by value. Captured variables are
read-only inside the closure by default.

```kestrel
let add = { (a: Int64, b: Int64) in a + b };
let double: (Int64) -> Int64 = { it * 2 };
numbers.map { it * 2 };
```

Use `it` only for single-argument closures when the expected function type is
known.

# Protocols, Generics, And Extensions

Protocols define requirements for methods, properties, static methods,
initializers, and associated types. Conformance is explicit.

```kestrel
protocol Drawable {
    func draw();
}

struct Circle {
    var radius: Float64;
}

extend Circle: Drawable {
    func draw() { }
}
```

Generic declarations use square brackets. Constraints are expressed with
protocol bounds and `where` clauses. Generics are monomorphized.

```kestrel
struct Box[T] {
    var value: T;
}

func max[T](a: T, b: T) -> T where T: Comparable {
    if a.greaterThan(other: b) { a } else { b }
}

extend Box[T] where T: Equatable {
    func equals(other: Box[T]) -> Bool {
        self.value == other.value
    }
}
```

Prefer generic, protocol-constrained code over untyped helpers or runtime type
checks.

# Errors

Kestrel uses typed errors through `Result[T, E]` and `throws` sugar. Recoverable
errors should be values, not unchecked exceptions.

```kestrel
enum FileError {
    case NotFound
    case PermissionDenied
}

func readFile(path: String) -> String throws FileError {
    throw FileError.NotFound;
}

func loadConfig() -> String throws FileError {
    try readFile(path: "config.toml")
}
```

Handle errors explicitly with `match` or propagate them with `try`.

# Modules And Imports

Kestrel files can declare a module and import other modules. Imports may be
renamed or grouped.

```kestrel
module MyApp.Utils

import std.collections.Array
import std.io as IO
import Library.(Item1, Item2)
import ModuleA.(Widget as WidgetA)
public import internal.types.Core
```

The standard library is auto-imported for common language and runtime
functionality. Do not add broad `std.*` imports unless the project or docs
explicitly use them.

# Libraries

Prefer existing first-party libraries before writing new helpers for common
tasks.

- `clutch` - CLI argument parsing and command definitions.
- `quill` - serialization/deserialization value model.
- `quill-json` - JSON parsing/emitting through Quill values.
- `quill-toml` - TOML parsing/emitting through Quill values.
- `http` - HTTP methods, status codes, headers, URLs, cookies, and wire helpers.
- `swoop` - HTTP client.
- `perch` - HTTP server/router framework.
- `flock` - package manager tooling.
- `jessup` - toolchain version manager.

# Style

- Use `PascalCase` for types, protocols, enum names, and enum cases.
- Use `camelCase` for functions, methods, fields, variables, and local
  constants.
- Prefer `let`; use `var` only when the binding mutates.
- Use explicit integer widths in public APIs.
- Prefer labeled parameters when positional arguments would be unclear.
- Prefer methods/extensions when behavior naturally belongs to a type.
- Prefer protocols and generic constraints over untyped or stringly typed APIs.
- Keep ownership visible: use `mutating` and `consuming` intentionally.

# Gotchas

- Single-name parameters have no external label: `foo(42)`, not `foo(x: 42)`.
- Memberwise initializers use field labels: `Point(x: 1, y: 2)`.
- Custom initializers follow normal parameter-label rules.
- Methods never declare `self` as a parameter.
- Generic arguments use square brackets: `Array[Int64]`, not `Array<Int64>`.
- `match` arms use `=>`, not `case`.
- Labeled enum associated values require labels in construction and patterns.
- Recursive enums require `indirect`.
- Closures capture by value.
- `it` only exists in single-argument closures with known expected type.
- `!` is the never type.
- Avoid broad `std.*` imports; common standard-library APIs are auto-imported.
- Do not rely on implicit numeric or pointer conversions.
