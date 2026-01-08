# TapLang: The Android Micro-Kernel

**TapLang** is an embedded, concurrent scripting language designed to fill the "Power User Vacuum" on Android. It acts as a secure user-space operating system, bridging the gap between raw shell scripts (too dangerous) and rigid automation apps (too limited).

It is built on a strict **Object-Capability (OCAP)** security model where authority is encapsulated within functions rather than tied to the global user identity, effectively solving the software supply chain problem for mobile automation.

---

## 1. Core Philosophy: Trust No One & "Personal OS"

The central tenant of TapLang is that **knowledge is power**. In this system, holding a reference to a function is synonymous with holding the permission to perform that function's action.

* **Encapsulated Authority:** Permissions are not checked against a global "user" but against the `SecurityManager` attached to the specific `CompiledFunction` being executed.
* **The "Tenant" Model (Home vs. Abroad):**
* **Home:** Code running in its origin context (`origin === host`) operates under its native grants.
* **Abroad:** When code is passed to a foreign context (e.g., a callback passed to a library), it loses all high-privilege capabilities unless explicitly granted the `:share` permission.


* **The "Personal OS" Vision:** TapLang transforms the smartphone from a consumption device into a programmable environment. It allows users to script their own OS-level behaviors (e.g., "Silence notifications during calendar meetings") via secure, sandboxed "mini-apps."

---

## 2. Syntax: The "Thumb-First" Philosophy

TapLang is designed for mobile entry. It prioritizes "British" prose flow over symbol density, eliminating the need to constantly switch keyboard layers on a phone.

### Resilience & Control Flow (`assuming`, `is`)

We replace complex null-checks with natural language sentinels.

```taplang
// "assuming" handles Optional Binding (If-Let) safely
assuming (web:get the "https://api.weather.gov"!) as response do
    if response is empty do
        toast some "No data received" !
    otherwise
        log the response's body !
    end
end

```

### Context & Scoping (`do` vs `do to`)

TapLang utilizes a hybrid scoping model to balance convenience with power:

* **Shared Scope (`do`)**: Standard blocks (like `if`, `while`, or bare `do`) share the parent's environment. Variables defined inside leak out, similar to Python. This reduces boilerplate in configuration scripts.
* **Context Injection (`do to`)**: Explicitly pushes a new **Runtime Environment** onto the stack. This allows for dynamic context injection, where a block of code inherits "magic" variables (like `this` or configuration overrides) from the object being acted upon.

```taplang
set theme to "light"

// Injects { theme: "dark" } into the lookup chain for this block only
do to new some theme  "dark" end do
    println theme! // Prints "dark"
end

println theme! // Prints "light"

```


### The Sentinel System

TapLang distinguishes between **Intentional Absence** and **Structural Absence** to prevent "Null Pointer" ambiguity:

* **`none`**: The value is explicitly null (e.g., cleared by user). Used as a first-class value in the VM.
* **`empty`**: The property does not exist in the structure (missing data).

---

## 3. The Object Model: Dynamic Composition

TapLang rejects rigid Class Inheritance (and C3 linearization) in favor of a **Dynamic Blueprint System**. This allows automation scripts to adapt "magically" to the device hardware at runtime.

* **The Factory Pattern (`Start`):** Objects are not born from classes but from standalone `Start` functions. These functions act as intelligent routers, deciding which capabilities to equip based on arguments or hardware availability.
* **Blueprints as Mixins:** Objects maintain a list of `methodProviders`. A `Start` function can dynamically inject a `CameraBlueprint` or `MockCameraBlueprint` into the same object instance.
* **Reverse Resolution:** Method lookups scan the provider list in reverse order. This allows new Blueprints to "shadow" or override behaviors of previous ones dynamically, enabling a flexible "Plugin" architecture for object behavior.

---

## 4. Native Function Architecture: The Guarded Bridge

TapLang defines a strict hierarchy for native code to ensure "Security by Default" when wrapping the Android SDK.

### The Interface Split

* **`TapNativeFunction` (Base):** For dynamic capabilities where the permission depends on the argument (e.g., `io:read` on a specific file path). These functions must perform their own `vmCtx.claimPermission` checks internally.
* **`GuardedNativeFunction` (Static):** For binary capabilities (e.g., `camera:capture`). Developers declare `requiredPermissions`, and the VM automatically validates and claims them before the function body ever executes.

### Argument Safety (`ArgUtils`)

To maintain the "Sentinel System" integrity, native functions never receive Kotlin `null`.

* **Strict Validation:** The `ArgUtils` helper enforces types and converts "missing" arguments into `TapNone`.
* **Variadic Packing:** Automatically collects tail arguments into Lists, matching the language's `...` syntax.

### Priority-Based Registry (Shadowing)

The VM utilizes a layered registry to route calls, ensuring the "Gold Standard" PC implementation remains available for debugging while the Android version takes precedence:

1. **The Script Calls:** `file:read("config.json")`
2. **The Registry Intercepts:** The VM checks the **Priority Stack**. It sees the Android implementation shadows the Core implementation.
3. **The Host Executes:** Instead of a raw I/O call, the Android Runtime executes a secure implementation routing the request through the **Android Storage Access Framework** or **ContentProvider**.

---

## 5. The Security Model

TapLang uses a unified, thread-safe `SecurityManager` to manage all authoritative grants.

### The "One-Time Ticket" & Atomic Claims

Authority is granted via the `Grant` object. Crucially, TapLang supports **Atomic Consumption**:

* **Ticket Grants:** Permissions marked `oneTimeUse` are "burned" upon a successful call.
* **Atomic Batching:** Native functions claim permissions in an atomic transaction. If a function requires 3 permissions and only 2 are valid, **zero** tickets are burned. This prevents "partial failure" states where a user loses a ticket but gets no action.

### The Standard Internal Protocol (The Colon Hierarchy)

To support a unified `SecurityManager`, all resources are canonicalized into colon-separated tokens:

* **Files:** `io:read:data:user:config_yaml` (Paths are flattened)
* **Networking:** `net:connect:com:google:mail` (Reverse domain logic)
* **Android:** `android:camera:0:capture`

### The "Iron Law" of Storage

To prevent internal sabotage, the VM enforces hardcoded boundaries at the Native I/O layer:

* **The Forbidden Zone:** Any path resolving to the VM's private system root (config/registry) results in an immediate Security Exception.
* **The Scratchpad:** Tapps have implicit Read/Write access to their own private working directory.
* **The External World:** Access to shared storage (`/Documents`, `/Photos`) requires explicit negotiation and permission grants.
* **Host Blacklisting:** The Host App can inject "Negative Permissions" (Blacklists) that override any user grant, preventing scripts from touching sensitive Android system directories.

### Optional Sandboxing ("Guest Mode")

Developers can strictly limit trusted libraries using the `sandbox` helper, wrapping the import in a `ProxyPermissionManager` that filters capabilities.

```taplang
// "Guest House" logic: Restrict the library to only read files
set library to sandbox the <maybe:library> <io:file:read> !

```

---

## 6. Performance: "Opt-Out of Magic"

Dynamic languages are historically slow due to dictionary lookups. TapLang adopts a **Dynamic-by-Default** architecture to favor flexibility, but provides a strictly optimized "Escape Hatch."

* **Default (Secure & Dynamic):** Variables are resolved via `LOAD_NAME` (Hash Map lookup).
* **`do fast` Blocks (The Optimization Island):** Critical sections can opt-out of dynamic resolution.

### The "AirLock" Mechanism

Fast blocks function as "Leaf Nodes" in the execution tree. They marshal data from the dynamic Map to a static Stack Array upon entry and sync it back upon exit.

```taplang
// 'takes': Copy-In only (Read-only optimization)
// 'with':  Copy-In and Copy-Back (Read-Write binding)
to fast with x y do 
    while x < 1000000 do
        set x to x + 1 // Compiles to LOAD_LOCAL (Native Array Access)
    end
end

```

### Volatile Syntax (`?`)

In `--strict-magic` environments, developers can explicitly mark variables that must remain dynamic (e.g., host-injected sensor values) using the `?` suffix. This forces the compiler to emit `LOAD_NAME` for that specific identifier while keeping the rest of the block optimized.

### Compiler Diagnostics & Syntactic Sugar

To bridge the gap between scripting and engineering:

* **Parser Shortcuts:** The compiler supports `fast each item in list`, which automatically wraps the loop in a `do fast` AST node.
* **Advisory Mode (`[info]`):** The compiler performs static analysis to detect "Hot Paths"—loops that only access up-scope variables. It emits non-blocking `[info]` logs suggesting a conversion to `do fast` for battery optimization.
* **Strict Flags:** Developers can compile with `--strict-magic` to enforce `do fast` usage in performance-critical Tapps.

---

## 7. The "Tapp" Architecture & Lifecycle

A **Tapp** is a secure, signed bundle. The ecosystem uses a three-tier binary hierarchy to separate development from execution.

### The File Hierarchy

1. **`.tapc` (The Component):** A single compiled module (pure bytecode). Signed by the Developer for integrity.
2. **`.tappc` (The Package):** The distribution format found on GitHub. Contains the `manifest.tap` (Intent), `install.tap` (Negotiator), and all module binaries.
3. **`.tapp` (The Executable):** The installed format. Stripped of the installer/text manifest. Contains the **Finalized Manifest** (Immutable Contract) and is signed by the **Local Device Hardware**.

### The Virtual File System (VFS)

Tapps do not unpack into a directory structure. They remain as flat binary archives (`.tapp`) containing a Header Table.

* **Architecture:** `Header (Path String -> Offset) + Serialized IntArray Blobs`.
* **Resolution:** Imports like `use "utils:helpers"` perform an O(1) lookup in the Header Table to jump directly to the memory offset of the required module.

### The Lifecycle Stages

* **Stage 1: The Intent (`manifest.tap`)**
* A sandboxed declaration of "Active" and "Potential" permissions. The "Ceiling" of what the app can ever do.


* **Stage 2: The Negotiation (`install.tap`)**
* An interactive wizard script that runs in a restricted VM context. It negotiates with the user to turn "Potential" permissions into "Active" ones.


* **Stage 3: The Contract (`.tapp`)**
* The VM generates the **Finalized Manifest**, appends it to the bytecode, and signs the bundle with the Android Keystore (Ed25519). This ensures the agreed-upon permissions are mathematically locked to the device.



---

## 8. Tooling & Visual Identity

TapLang is designed as a full-stack platform, not just a language.

### Visual Identity

* **The Editor Icon:** A **"Tail-Dot" (Motion T)**. Represents the workflow (Stroke -> Press).

### Ecosystem Features

* **GitHub Integration:** The host app can download, verify, and install `.tappc` binaries directly from GitHub, enabling a decentralized "Tapp Store."
* **Lexer-Driven IDE:** The mobile editor utilizes the language's Lexer for real-time, "thumbs-first" syntax highlighting and keyword-based autocomplete, keeping the user on the primary keyboard.
* **Recursive Compilation:** The `ModuleIO` system supports compiling directories of scripts into single portable binaries, allowing complex tools to be shared as easily as a photo.
* **Library Conventions:** Standard libraries can include a `permissions.tap` file, allowing the Host App to introspect requirements and elevate them to the user during the install flow.

---

## 9. Licensing & Trust

To balance ecosystem growth with user trust, the project is split into two licenses:

* **TapLang Core (Apache 2.0):** The Compiler, VM, and Standard Library definitions are permissive. This ensures TapLang can be embedded in any application, fostering a standard scripting layer for the industry.
* **Tap Android Runtime (GPLv3):** The reference Android Host application—which performs the critical "Shadowing" and Permission Management—is copyleft. This guarantees that the *sandbox itself* remains open and auditable. Users must be able to verify that the Runtime is actually checking the permissions it claims to check.
