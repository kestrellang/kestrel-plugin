---
name: kestrel
description: Use when writing, editing, reviewing, or explaining Kestrel source code, examples, packages, syntax, semantics, or tooling.
---

# Kestrel Skill

Use this guide when writing, editing, reviewing, or explaining Kestrel source
code. Kestrel is a statically typed, expression-oriented systems language with
Swift-like syntax, copy-by-default value semantics, explicit
borrowing/mutating/consuming parameter modes, RAII cleanup, nominal structs and
enums, protocol-constrained generics, opaque (`some`) types, and pattern
matching.

Prefer these rules over assumptions from Swift, Rust, Kotlin, TypeScript, or C.

# Install And Tools

Install Kestrel with the official install script:

```sh
curl -fsSL https://kestrel-lang.com/install | sh
jessup install preview
kestrel --version
flock --help
```

Jessup manages compiler toolchains:

```sh
jessup install preview
jessup install nightly
jessup install 6.0
jessup list
jessup show
jessup default preview-0.16.0
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
var count = 0; // Int64
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

# Strings

Kestrel has four string forms. Double-quoted strings are "cooked" — they process
escapes and interpolation. A `#`-delimited form is raw: no escapes, no
interpolation. Triple-quoted forms are multi-line.

```kestrel
let cooked = "tab\tand newline\n";        // escapes processed
let raw = #"C:\path\with\no\escapes"#;    // literal backslashes
let multi = """
    line one
    line two
    """;
let rawMulti = #"""
    raw \n stays literal
    """#;
```

Interpolate with `\(expr)` inside a cooked string. The brace form `\{expr}` is
not valid.

```kestrel
let name = "Ada";
let count = 3;
print("Hello \(name), you have \(count) messages");
print("sum = \(1 + 2)");
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

`Cloneable` is for types that are still copyable but need custom copy logic
instead of a bit-copy. The compiler calls `clone()` automatically wherever the
value would otherwise be implicitly copied — assignment, argument pass, return —
and `.clone()` can also be called explicitly. The conformer decides how deep the
copy goes: a refcounted box just bumps its count, a struct with heap fields
deep-copies them.

```kestrel
struct Buffer: Cloneable {
    var bytes: [UInt8];

    func clone() -> Buffer {
        Buffer(bytes: self.bytes.clone())
    }
}

let a = Buffer(bytes: [1, 2, 3]);
let b = a;          // implicit clone()
let c = a.clone();  // explicit
```

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

Index collections with parentheses, not brackets — brackets are reserved for
generic arguments. Tuples use positional `.0` / `.1` access.

```kestrel
let first = numbers(0);     // not numbers[0]
let score = ages("ada");
let x = point.0;
```

Do not invent implicit conversions. Prefer explicit types when inference is
ambiguous, especially for empty arrays, `.None`, public APIs, and numeric
literals.

# Operators

Boolean logic uses the keywords `and`, `or`, and `not` — never `&&`, `||`, or a
boolean `!`.

```kestrel
if ready and not waiting { start(); }
if x == 4 or x == 7 { skip(); }
let blocked = not allowed;
```

`!` is never boolean negation. It is the never type, postfix force-unwrap
(`value!`), and bitwise NOT on integers. Use `not` for booleans.

Other operators are conventional: comparison `==`, `!=`, `<`, `<=`, `>`, `>=`;
arithmetic `+`, `-`, `*`, `/`, `%`; bitwise `&`, `|`, `^`, `<<`, `>>`; and
compound assignment `+=`, `-=`, `*=`, `/=`.

# Optionals

`T?` is the optional type. Its values are `.Some(x)` and `.None`; `null` is an
alias for `.None`.

```kestrel
var maybe: Int64? = null;       // same as .None
maybe = .Some(42);

let name: String? = .None;
```

Unwrap with pattern matching (`if let` / `guard let`), nil-coalescing `??`, or a
postfix `!` that force-unwraps and traps on `.None`. The `??` default is lazy —
evaluated only when the value is `.None`. There is no `?.` optional-chaining
operator.

```kestrel
let value = maybe ?? 0;               // default when None
let label = lookup() ?? "anonymous";  // RHS runs only if None
let forced = maybe!;                  // traps if None
```

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

`if`, `while`, and `guard` accept several comma-separated clauses — each a
binding or a boolean condition. All must hold to enter the body (or, for
`guard`, the `else` is taken).

```kestrel
if let .Some(x) = a, let .Some(y) = b, x < y {
    use(x, y);
}

guard flag, let .Some(token) = session else {
    return;
}

while let .Some(x) = next(), x > 0 {
    process(x);
}
```

`Optional` also has dedicated patterns: `some <binding>` matches `.Some` and
binds the value, `null` matches `.None`.

```kestrel
match maybe {
    some x => use(x),
    null => fallback()
}
```

# Loops And Ranges

`for … in …` iterates anything iterable — ranges, arrays, dictionaries. `while`
loops on a condition.

```kestrel
for i in 0..<10 {
    print(i);
}

for item in numbers {
    process(item);
}

while count > 0 {
    count -= 1;
}
```

Ranges are values. `..<` excludes the upper bound; `..=` includes it. One-sided
ranges omit a bound.

```kestrel
let upTo = 0..<10;       // 0,1,…,9
let through = 0..=10;    // 0,1,…,10
let fromFive = 5..;      // 5,6,7,… (unbounded upper)
let belowTen = ..<10;    // unbounded lower, exclusive
let throughTen = ..=10;  // unbounded lower, inclusive
```

One-sided ranges also serve as slice indices and `match` patterns. There is no
bare `..5`; write `..<5` or `..=5`.

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

Opaque types use `some P`. As a parameter, `some P` is sugar for a generic
(`func f(x: some P)` ≡ `func f[T: P](x: T)`, statically dispatched and
monomorphized). In return or field position, `some P` hides one concrete type
from callers while the callee still knows it.

```kestrel
func makeClock() -> some Clock { SystemClock() }  // caller sees `some Clock`
func run(db: some SqliteExecutor) { }             // generic sugar
```

Existential `any P` types are not in this language version yet — use generics or
`some P`.

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
module myapp.utils

import std.collections.Array
import std.io as IO
import quill.(Encoder, Decoder)
import graphics.(Widget as WidgetA)
public import internal.types.Core
```

Package and module names are lowercase (`http`, `quill`, `myapp.utils`); the
types and symbols imported from them stay `PascalCase`.

The standard library is auto-imported for common language and runtime
functionality. Do not add broad `std.*` imports unless the project or docs
explicitly use them.

# Entry Point

An executable's entry point is the function annotated with `@main`. This is
explicit — a function is not special just because it is named `main`. It may
return `()` or an integer exit code.

```kestrel
@main
func main() {
    print("hello");
}
```

# Libraries

Prefer existing first-party libraries before writing new helpers for common
tasks. Package names are lowercase; each has reference docs at
`kestrel-lang.com/flock/kestrel/<name>` and is added under `[dependencies]` as
`"kestrel/<name>"`.

- `clutch` — CLI argument parsing and command definitions.
  (`kestrel-lang.com/flock/kestrel/clutch`)
- `quill` — serialization/deserialization value model.
  (`kestrel-lang.com/flock/kestrel/quill`)
- `quill-json` — JSON parsing/emitting through Quill values.
  (`kestrel-lang.com/flock/kestrel/quill-json`)
- `quill-toml` — TOML parsing/emitting through Quill values.
  (`kestrel-lang.com/flock/kestrel/quill-toml`)
- `http` — HTTP methods, status codes, headers, URLs, cookies, wire helpers.
  (`kestrel-lang.com/flock/kestrel/http`)
- `swoop` — HTTP client. (`kestrel-lang.com/flock/kestrel/swoop`)
- `perch` — HTTP server/router framework.
  (`kestrel-lang.com/flock/kestrel/perch`)
- `html-builder` — HTML construction.
  (`kestrel-lang.com/flock/kestrel/html-builder`)
- `datetime` — dates, times, instants, and time zones.
  (`kestrel-lang.com/flock/kestrel/datetime`)
- `crypto` — hashing and cryptographic primitives.
  (`kestrel-lang.com/flock/kestrel/crypto`)
- `uuid` — UUID generation and parsing.
  (`kestrel-lang.com/flock/kestrel/uuid`)
- `talon-sqlite` — SQLite bindings.
  (`kestrel-lang.com/flock/kestrel/talon-sqlite`)

`flock` (package manager) and `jessup` (toolchain manager) are CLI tools, not
libraries — see Install And Tools.

# Packages And Documentation

Published packages live in the Flock registry at
`https://registry.kestrel-lang.com` and are browsable at `kestrel-lang.com/flock`
— each package has a page at `kestrel-lang.com/flock/<org>/<package>` (for
example `kestrel-lang.com/flock/kestrel/quill`). To use one, add it under
`[dependencies]` in `flock.toml` (as `"<org>/<package>" = "<version>"`) and run
`flock update`; there is no `flock add` subcommand.

```toml
[dependencies]
"kestrel/quill" = "^0.1.0"
"kestrel/http" = "^0.1.0"
```

For authoritative, up-to-date APIs, prefer published references over guessing
signatures:

- Standard-library reference: `kestrel-lang.com/reference/stdlib`.
- Context7: the language and stdlib docs are published there. Resolve the
  `kestrellang/kestrel` library, then query it for stdlib/language docs. The
  source repo ships a `context7.json` that indexes `docs/stdlib` and
  `docs/language`.
- Source, issues, and the `context7.json` config:
  `github.com/kestrellang/kestrel`.

# Style

- Use `PascalCase` for types, protocols, enum names, and enum cases.
- Use `camelCase` for functions, methods, fields, variables, and local
  constants.
- Use lowercase for package and module names (`http`, `myapp.utils`).
- Prefer `let`; use `var` only when the binding mutates.
- Use explicit integer widths in public APIs.
- Prefer labeled parameters when positional arguments would be unclear.
- Prefer methods/extensions when behavior naturally belongs to a type.
- Prefer protocols and generic constraints over untyped or stringly typed APIs.
- Keep ownership visible: use `mutating` and `consuming` intentionally.

# Gotchas

- Boolean logic is `and` / `or` / `not`, not `&&` / `||` / boolean `!`.
- `!` is the never type, postfix force-unwrap, and bitwise NOT — never boolean negation.
- Interpolate with `"\(expr)"`, not `"\{expr}"`; raw strings are `#"..."#`.
- Index collections with parentheses: `array(i)`, not `array[i]` (brackets are generics).
- Ranges: `a..<b` exclusive, `a..=b` inclusive; one-sided `a..`, `..<b`, `..=b` (no bare `..b`).
- Iterate with `for x in seq { }`; the entry point is `@main func main()`.
- Optionals: `null` is `.None`; unwrap with `??`, `if let`, or postfix `!`. No `?.` chaining.
- Match `Optional` with `some x` / `null`, or with `.Some(x)` / `.None`.
- `if` / `while` / `guard` chain comma-separated bindings and boolean conditions.
- `Cloneable` runs a custom `clone()` on every implicit copy (e.g. refcount bump).
- Package and module names are lowercase; existentials (`any P`) are not in this version.
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
