Excellent choice. These two projects sit at the intersection of **Linux kernel programming**, **systems engineering**, and **Rust** — one of the most demanding and rewarding areas in all of software. Let me build your complete entry map from zero.

---

# 🧠 PHASE 0 — Foundation Concepts (What You're Even Looking At)

Before touching a single line of code, you must understand the **conceptual terrain**. Think of this like learning the rules of chess before studying openings.

## What is eBPF?

**eBPF** stands for **extended Berkeley Packet Filter**. To understand it, let's build up from scratch:

```
NORMAL LINUX ARCHITECTURE:
┌─────────────────────────────────────────────────┐
│                 USER SPACE                       │
│   Your programs, bash, curl, your Rust app       │
│   (cannot directly touch hardware or kernel)     │
├─────────────────────────────────────────────────┤
│               KERNEL BOUNDARY                    │
│   (system calls = the only door between worlds)  │
├─────────────────────────────────────────────────┤
│                KERNEL SPACE                      │
│   Manages CPU, memory, network, filesystem       │
│   Linux kernel code lives here                   │
└─────────────────────────────────────────────────┘
```

Traditionally, if you wanted to hook into the kernel (e.g., intercept every network packet), you had to write a **kernel module** — dangerous, unstable, requires kernel recompile, one bug = system crash.

**eBPF solves this** by creating a **safe sandbox inside the kernel** where you can run YOUR programs:

```
WITH eBPF:
┌───────────────────────────────────────────────────┐
│                 USER SPACE                         │
│  Your loader program (Rust/C/Go)                  │
│  - Compiles eBPF bytecode                         │
│  - Loads it into kernel via syscall               │
│  - Reads results from shared maps                 │
├───────────────────────────────────────────────────┤
│                KERNEL SPACE                       │
│  ┌─────────────────────────────────────────────┐ │
│  │  eBPF VIRTUAL MACHINE (sandbox)             │ │
│  │  - Verifier checks your code before running  │ │
│  │  - JIT compiles to native machine code       │ │
│  │  - Hooks into kernel events (network, file,  │ │
│  │    syscalls, tracepoints...)                 │ │
│  └─────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

**Key terminology you'll encounter:**

| Term | What it means |
|---|---|
| **eBPF program** | Your code that runs *inside* the kernel sandbox |
| **eBPF map** | Shared memory between kernel-side eBPF program and user-space app |
| **Hook point** | The specific kernel event where your eBPF program attaches (e.g., incoming packet) |
| **XDP** | eXpress Data Path — hooks at the network driver level, earliest point possible |
| **TC** | Traffic Control — hooks at the kernel networking layer |
| **Tracepoint** | Named kernel event you can hook into (e.g., `sys_enter_read`) |
| **BTF** | BPF Type Format — debug info that lets eBPF programs work across kernel versions |
| **Verifier** | Kernel component that proves your eBPF program is safe before running it |
| **BPF syscall** | The system call that loads/attaches eBPF programs |
| **JIT** | Just-In-Time compiler — turns eBPF bytecode → native CPU instructions |

---

## What is Aya?

Aya is an eBPF library built with a focus on operability and developer experience. It does not rely on libbpf or bcc — it's built from the ground up purely in Rust, using only the libc crate to execute syscalls.

```
TRADITIONAL eBPF TOOLCHAIN:
  Your C Code
     ↓  (clang)
  eBPF bytecode (.o file)
     ↓  (libbpf / bcc — C library)
  Kernel loading

AYA TOOLCHAIN:
  Your Rust Code (eBPF side)
     ↓  (cargo + Rust eBPF target)
  eBPF bytecode (.o file)
     ↓  (aya — pure Rust, no libbpf!)
  Kernel loading
  Your Rust Code (userspace side)  ←→  eBPF Maps
```

The revolutionary thing: **you write both sides in Rust** — the kernel-side eBPF program AND the userspace loader. One language, end to end.

## What is bpfman?

bpfman operates as an eBPF manager, focusing on simplifying the deployment and administration of eBPF programs. Its notable features encompass: System Overview — provides insights into how eBPF is utilized in your system; a built-in program loader that supports program cooperation for XDP and TC programs, as well as deployment of eBPF programs from OCI images; and eBPF filesystem management.

Think of it this way:

```
WITHOUT bpfman:
  App A loads XDP program → owns the network hook
  App B tries to load XDP program → CONFLICT! Only one allowed!

WITH bpfman:
  bpfman daemon manages ALL eBPF programs
  App A registers with bpfman → bpfman multiplexes the hook
  App B registers with bpfman → both run, cooperatively
  Kubernetes operator → deploy eBPF across entire clusters via CRDs
```

bpfman is built ON TOP of aya. bpfman includes a Kubernetes operator, extending these capabilities to Kubernetes — allowing users to confidently deploy eBPF through custom resource definitions across nodes in a cluster.

---

# 🗺️ THE COMPLETE LEARNING + CONTRIBUTION ROADMAP

```
LEVEL 0: Prerequisites
    ↓
LEVEL 1: Understand eBPF deeply (theory)
    ↓
LEVEL 2: Build with Aya (hands-on)
    ↓
LEVEL 3: Read Aya internals
    ↓
LEVEL 4: First Aya contribution
    ↓
LEVEL 5: bpfman architecture
    ↓
LEVEL 6: First bpfman contribution
    ↓
LEVEL 7: Deep systems contributor
```

---

## LEVEL 0 — Prerequisites Checklist

Before diving in, you need solid footing in all of these:

```
MUST KNOW:
┌──────────────────────────────────────────────────┐
│  Rust                                            │
│  ├── Ownership, borrowing, lifetimes             │
│  ├── Traits, generics, async/await               │
│  ├── unsafe Rust (you WILL use this)             │
│  ├── Proc macros (aya uses them heavily)         │
│  └── cargo workspaces                            │
├──────────────────────────────────────────────────┤
│  Linux fundamentals                              │
│  ├── Processes, file descriptors                 │
│  ├── System calls (what they are, how they work) │
│  ├── Network stack basics (packets, sockets)     │
│  └── /proc and /sys filesystem                   │
├──────────────────────────────────────────────────┤
│  C basics (for reading kernel headers/eBPF C)    │
│  └── Pointers, structs, bitfields                │
└──────────────────────────────────────────────────┘

HELPFUL BUT CAN LEARN ALONGSIDE:
┌──────────────────────────────────────────────────┐
│  Networking: TCP/IP, XDP, TC                     │
│  Kubernetes basics (for bpfman)                  │
│  gRPC / protobuf (bpfman's API layer)            │
│  Go (bpfman has Go client code)                  │
└──────────────────────────────────────────────────┘
```

---

## LEVEL 1 — Theory: Understand eBPF Architecture

**Read in this exact order:**

```
Step 1: ebpf.io/what-is-ebpf
        → The official conceptual overview (30 min)

Step 2: The Aya Book
        → https://aya-rs.dev/book
        → This is your primary textbook for aya

Step 3: Brendan Gregg's eBPF page
        → https://www.brendangregg.com/ebpf.html
        → Deep performance/observability angle

Step 4: Linux kernel docs on BPF
        → https://www.kernel.org/doc/html/latest/bpf/index.html
        → Heavy, but teaches you how the verifier thinks
```

**Mental model to build — the eBPF lifecycle:**

```
WRITE           COMPILE          VERIFY          LOAD
┌────────┐     ┌────────┐      ┌────────┐      ┌────────┐
│ Rust   │────▶│ cargo  │─────▶│ Kernel │─────▶│ Attach │
│ eBPF   │     │ →.o    │      │Verifier│      │to hook │
│ code   │     │ file   │      │(safety)│      │point   │
└────────┘     └────────┘      └────────┘      └────────┘
                                                    │
                                    ┌───────────────▼────────────┐
                                    │  RUNNING in kernel sandbox │
                                    │  Packet arrives → your     │
                                    │  eBPF program fires        │
                                    │  Writes to BPF Map         │
                                    └───────────────┬────────────┘
                                                    │
                                    ┌───────────────▼────────────┐
                                    │  Userspace reads BPF Map   │
                                    │  (your Rust app)           │
                                    └────────────────────────────┘
```

---

## LEVEL 2 — Hands-On: Build Things With Aya

### Step 1 — Set up your environment

```bash
# Install Rust nightly (eBPF target requires it)
rustup install nightly
rustup component add rust-src --toolchain nightly

# Install bpf-linker (links eBPF object files)
cargo install bpf-linker

# Install cargo-generate (scaffolds aya projects)
cargo install cargo-generate

# Your Linux kernel must be >= 5.15 ideally
uname -r
```

### Step 2 — Generate your first aya project

```bash
# This scaffolds a complete aya project for you
cargo generate https://github.com/aya-rs/aya-template

# You'll be prompted:
# - Program type? (XDP, TC, Tracepoint, etc.)
# - Project name?
```

**What gets generated:**

```
my-ebpf-project/
├── Cargo.toml                 ← workspace
├── my-project/                ← USERSPACE code (your Rust app)
│   ├── src/main.rs            ← loads & attaches eBPF program
│   └── Cargo.toml
├── my-project-ebpf/           ← KERNEL-SIDE eBPF code (Rust!)
│   ├── src/main.rs            ← the program running in kernel
│   └── Cargo.toml
└── my-project-common/         ← shared types (both sides use)
    └── src/lib.rs
```

### Step 3 — Build 3 progressively complex programs

```
Project 1: XDP packet counter
  → Count every incoming packet by protocol
  → Learn: XDP hook, BPF HashMap, reading maps

Project 2: syscall tracer
  → Hook sys_enter_openat, log every file opened
  → Learn: tracepoints, perf event arrays, async reading

Project 3: network firewall
  → Block traffic from specific IPs
  → Learn: XDP_DROP return code, LPM trie map, BTF
```

---

## LEVEL 3 — Read Aya's Internals

Now you understand the surface. Time to read the engine. Here is Aya's codebase, decoded:

```
aya/                    ← MAIN CRATE (userspace library)
├── src/
│   ├── ebpf.rs         ← The Ebpf struct: entry point, loads .o files
│   ├── programs/       ← All program types: XDP, TC, Tracepoint, etc.
│   │   ├── xdp.rs      ← How XDP programs are attached to interfaces
│   │   ├── trace_point.rs
│   │   └── ...
│   ├── maps/           ← All map types: HashMap, Array, PerfEventArray
│   │   ├── hash_map.rs
│   │   ├── perf.rs
│   │   └── ...
│   └── sys/            ← Raw BPF syscall wrappers (unsafe Rust)
│       └── bpf.rs      ← THIS IS THE GOLD: direct kernel interface

aya-ebpf/               ← KERNEL-SIDE crate (what runs in eBPF VM)
├── src/
│   ├── programs/       ← Macros and helpers for eBPF programs
│   ├── maps/           ← eBPF-side map access (different from userspace!)
│   └── helpers.rs      ← Kernel helper function bindings

aya-obj/                ← ELF object file parser
│                         (reads the .o file your eBPF compiles to)
│                         Key insight: aya doesn't use libbpf because
│                         it has its OWN ELF parser here!

aya-build/              ← Build scripts for eBPF compilation

aya-log/                ← Logging system bridging kernel↔userspace
```

**Reading strategy** — use the "outside-in" method:
1. Start with `aya/src/ebpf.rs` — trace how `Ebpf::load_file()` works
2. Follow it into `aya-obj` — see how the ELF is parsed
3. Then follow the syscall path in `aya/src/sys/bpf.rs`
4. Then read one program type end-to-end (start with `xdp.rs`)

---

## LEVEL 4 — First Aya Contribution

**The expert's entry strategy:** Don't look for random "good first issues." Instead, follow the **Curiosity-Driven Contribution** model:

```
STEP 1: While learning, write down EVERY question:
  "Why does this panic instead of returning Err?"
  "This error message is confusing"
  "This example is missing"
  "This map type isn't documented"

STEP 2: Check if there's an open issue for it
  github.com/aya-rs/aya/issues

STEP 3: If not, that's YOUR contribution
```

**Concrete first-contribution targets for Aya:**

```
EASIEST (documentation):
  → Add doc examples to map types in aya/src/maps/
  → Fix misleading error messages in programs/
  → Add missing rustdoc to public API items

MEDIUM (tests):
  → Add integration tests in test/ directory
  → The test infrastructure uses VMs — read test/README

HARDER (features):
  → Implement missing eBPF program types
    (check: does aya support BPF_PROG_TYPE_SK_MSG? etc.)
  → Add missing map types
  → Improve BTF handling in aya-obj
```

**Workflow:**

```bash
# Fork the repo, then:
git clone https://github.com/YOUR_USERNAME/aya
cd aya
git checkout -b my-contribution

# Read CONTRIBUTING.md FIRST — always
cat CONTRIBUTING.md

# Build
cargo build

# Run tests (requires Linux + root for kernel tests)
cargo test

# Check CI passes
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

---

## LEVEL 5 — bpfman Architecture

bpfman is architecturally more complex than aya — it's a **daemon + Kubernetes operator**:

```
bpfman SYSTEM ARCHITECTURE:

┌──────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                   │
│  ┌─────────────────────────────────────────────────┐ │
│  │         bpfman-operator (Go)                    │ │
│  │  Watches BpfProgram CRDs                        │ │
│  │  Reconciles desired state → calls bpfman API    │ │
│  └────────────────────┬────────────────────────────┘ │
└───────────────────────│──────────────────────────────┘
                        │ gRPC
┌───────────────────────▼──────────────────────────────┐
│                  LINUX HOST                           │
│  ┌─────────────────────────────────────────────────┐ │
│  │         bpfman daemon (Rust)                    │ │
│  │  ├── gRPC server (bpfman-api)                   │ │
│  │  ├── Program lifecycle manager                  │ │
│  │  ├── XDP/TC dispatcher (multiplexer!)           │ │
│  │  ├── OCI image loader (pulls eBPF from registry)│ │
│  │  └── Uses AYA internally to load programs       │ │
│  └─────────────────────────────────────────────────┘ │
│                        │                             │
│  ┌─────────────────────▼───────────────────────────┐ │
│  │         Linux Kernel                            │ │
│  │  eBPF programs loaded & attached                │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**bpfman codebase map:**

```
bpfman/             ← Core daemon logic
├── src/
│   ├── bin/bpfman.rs     ← Main entry point
│   ├── lib.rs            ← Core program management
│   ├── multiprog/        ← XDP/TC dispatcher (COMPLEX, FASCINATING)
│   └── ...

bpfman-api/         ← gRPC API definitions + server
├── proto/           ← .proto files defining the API
└── src/             ← Rust gRPC server implementation

bpf/                ← C eBPF programs (the dispatchers themselves!)
│                     These are the kernel-side C programs bpfman
│                     uses to multiplex multiple eBPF programs

clients/gobpfman/   ← Go client library

examples/           ← Example eBPF programs using bpfman
```

---

## LEVEL 6 — First bpfman Contribution

bpfman is a CNCF Sandbox project, which means it has structured community processes.

```
ENTRY POINTS:
1. Read: https://bpfman.io/main/getting-started/building-bpfman/
2. Join the community meeting (see MEETINGS.md)
3. Look at open issues on GitHub

GOOD FIRST CONTRIBUTION AREAS:
├── Documentation improvements (docs/ folder, mkdocs)
├── Example eBPF programs (examples/ folder)
│   → New examples showing bpfman features
├── Go client improvements (clients/gobpfman/)
│   → Good if you know Go
├── Integration test coverage (tests/integration-test/)
└── CLI UX improvements (bpfman/src/bin/)
```

---

## YOUR 90-DAY BATTLE PLAN

```
WEEK 1-2:  eBPF theory + Aya book
WEEK 3-4:  Build your 3 sample projects
WEEK 5-6:  Read Aya source code (ebpf.rs → sys/bpf.rs)
WEEK 7-8:  First Aya contribution (docs/tests)
WEEK 9-10: Read bpfman architecture + build from source
WEEK 11-12: First bpfman contribution + join community
```

---

## 🧘 Cognitive Principle for This Journey

This domain requires what psychologists call **schema building** — you can't rush it. Each concept (verifier, maps, hook points, BTF) is a node. The understanding comes from building **edges between nodes** through practice, not just reading.

The single most powerful practice: **after reading any code, close the tab and redraw the architecture from memory as ASCII.** If you can't, you don't own it yet. This forces **active recall**, which is 3x more effective than re-reading (Roediger & Karpicke, 2006 — the "Testing Effect").

The projects you've chosen (aya + bpfman) are genuinely difficult, genuinely important, and used in production at companies like Meta, Google, and Cloudflare. Every hour you invest here compounds for years.

Start with `https://aya-rs.dev/book` today. Build the XDP counter by end of week. That single program will teach you more than a month of reading.

Let me fetch the actual source files so I give you real, accurate entry points — not guesses.Now I have real, accurate data from the actual docs and source. Let me give you the precise, surgical entry point guide.

---

# 🎯 The Exact Entry Points for Code Investigation

The correct mental model before you open a single file:

```
THERE ARE TWO SIDES TO AYA — ALWAYS REMEMBER THIS:

  SIDE A: aya-ebpf/          ← Runs INSIDE the Linux kernel (eBPF VM)
  SIDE B: aya/               ← Runs in YOUR process (userspace)

  They COMMUNICATE via BPF Maps.
  You must understand BOTH sides to understand aya.
```

Do not mix them up. Different constraints, different APIs, different Rust targets.

---

## 🔴 ENTRY POINT #1 — The Userspace Side (`aya/`)

The single most important struct in the entire codebase is:

```
aya::Ebpf
```

For more loading options, see `EbpfLoader`. The `Ebpf` struct is used like: `let bpf = Ebpf::load_file("file.o")?` — it loads eBPF bytecode from a file, parses the object code, and initializes the maps defined in it. If the kernel supports BTF debug info, it is automatically loaded from `/sys/kernel/btf/vmlinux`.

Everything flows through this struct. Here is the **exact file path and call chain** to trace:

```
FILE:  aya/src/lib.rs          ← Public API definition, re-exports
         ↓
FILE:  aya/src/ebpf.rs         ← Ebpf struct implementation
         ↓
  pub fn load_file(path) → calls load(bytes)
         ↓
  pub fn load(data: &[u8]) → calls EbpfLoader::new().load(data)
         ↓
FILE:  aya/src/ebpf.rs         ← EbpfLoader struct
         ↓
  fn load(data) →
    1. aya_obj::Object::parse(data)    ← PARSES the .o ELF file
    2. object.relocate_maps()          ← Fixes map references
    3. object.relocate_btf()           ← Applies BTF relocations
    4. bpf_load_map() for each map     ← Kernel syscall per map
    5. bpf_load_program() per program  ← Kernel syscall per program
         ↓
FILE:  aya/src/sys/bpf.rs      ← RAW BPF SYSCALL WRAPPERS
         ↓
  bpf(BPF_PROG_LOAD, attr)  ← THIS IS THE ACTUAL LINUX SYSCALL
```

**Your investigation trail, step by step:**

```
STEP 1 — Open:  aya/src/lib.rs
  Purpose: See what is public API. What does aya EXPORT?
  Look for: pub use, pub mod
  Key question: "What can a user of this library actually touch?"

STEP 2 — Open:  aya/src/ebpf.rs
  Purpose: The Ebpf struct — the heart of userspace aya
  Look for: impl Ebpf, impl EbpfLoader
  Key question: "How does load() actually work, step by step?"

STEP 3 — Open:  aya-obj/src/obj.rs  (or mod.rs)
  Purpose: ELF object file parser — reads your .o file
  Key concept: ELF = Executable and Linkable Format.
               Your compiled eBPF code is stored as ELF.
               aya parses it WITHOUT libbpf — this is what makes aya unique.
  Look for: pub struct Object, fn parse()
  Key question: "How does aya read sections from the .o file?"

STEP 4 — Open:  aya/src/sys/bpf.rs
  Purpose: The LOWEST level — direct kernel BPF syscalls
  Look for: fn bpf(), BPF_PROG_LOAD, BPF_MAP_CREATE
  Key question: "What does the kernel actually receive?"
  WARNING: This is unsafe Rust. Study it carefully.
```

---

## 🔵 ENTRY POINT #2 — The Kernel Side (`aya-ebpf/`)

This is the code that actually runs **inside the kernel**. It has completely different constraints than normal Rust:

```
CONSTRAINTS OF eBPF RUST (aya-ebpf side):
┌──────────────────────────────────────────────┐
│  NO std library  → use core instead          │
│  NO heap         → no Vec, no String         │
│  NO panic        → must handle all errors    │
│  NO main()       → entry = your #[xdp] fn   │
│  512 bytes stack → be very careful           │
│  NO loops that   → verifier rejects infinite │
│  run forever       loops                     │
└──────────────────────────────────────────────┘
```

The entry point of a kernel-side eBPF program in aya looks like this (from the real Aya book):

```rust
#![no_std]   // ← NO standard library
#![no_main]  // ← NO main() function

use aya_ebpf::{
    bindings::xdp_action,
    macros::xdp,           // ← PROC MACRO: generates ELF section name
    programs::XdpContext,  // ← Context gives you access to the packet
};

#[xdp]   // ← This macro is the ENTRY POINT declaration
pub fn xdp_hello(ctx: XdpContext) -> u32 {
    match unsafe { try_xdp_hello(ctx) } {
        Ok(ret) => ret,
        Err(_) => xdp_action::XDP_ABORTED,
    }
}

unsafe fn try_xdp_hello(ctx: XdpContext) -> Result<u32, u32> {
    Ok(xdp_action::XDP_PASS)  // ← Let packet through
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}  // ← Verifier ensures this is never reached
}
```

**Your investigation trail for the kernel side:**

```
STEP 1 — Open: aya-ebpf-macros/src/lib.rs
  Purpose: The proc macros like #[xdp], #[kprobe], #[tracepoint]
  Key question: "What does #[xdp] actually DO to my function?"
  Answer: It places the function in the correct ELF section
          e.g. section name "xdp" → kernel knows it's an XDP program

STEP 2 — Open: aya-ebpf/src/programs/xdp.rs
  Purpose: XdpContext — the struct passed to your XDP function
  Key question: "How do I access packet data from context?"
  Look for: impl XdpContext, data(), data_end()

STEP 3 — Open: aya-ebpf/src/maps/hash_map.rs
  Purpose: HashMap on the eBPF (kernel) side
  Key question: "How does a map lookup work from inside the kernel?"
  Look for: fn get(), fn insert()
  Note: These call kernel BPF helper functions like bpf_map_lookup_elem
```

---

## 🟢 ENTRY POINT #3 — The Bridge: BPF Maps

Maps are the **communication channel** between your kernel eBPF code and your userspace Rust app. Understanding them is critical.

```
KERNEL SIDE                    USERSPACE SIDE
(aya-ebpf/src/maps/)           (aya/src/maps/)

HashMap::get(&key)    ←→    HashMap::get(&key, &mut value)
HashMap::insert()     ←→    HashMap::insert(&key, &value, flags)
PerfEventArray        ←→    AsyncPerfEventArray
                             (async! reads events from kernel)
```

```
FILE TO OPEN:  aya/src/maps/hash_map.rs    (userspace)
FILE TO OPEN:  aya-ebpf/src/maps/hash_map.rs  (kernel side)

COMPARE THEM SIDE BY SIDE.
They operate on the SAME kernel memory but through different APIs.
This comparison will teach you more than any tutorial.
```

---

## 🟡 ENTRY POINT #4 — bpfman's `main.rs`

For bpfman, the entry is the daemon binary:

```
FILE:  bpfman/src/bin/bpfman.rs   ← main() of the daemon
         ↓
  Sets up gRPC server
         ↓
FILE:  bpfman-api/src/lib.rs      ← gRPC API (protobuf-generated + handwritten)
         ↓
FILE:  bpfman/src/lib.rs          ← Core program management logic
         ↓
FILE:  bpfman/src/multiprog/      ← THE MOST INTERESTING PART
         xdp.rs                   ← XDP dispatcher (how multiple programs share one hook)
         tc.rs                    ← TC dispatcher
```

The **dispatcher** is bpfman's secret weapon. This is how it lets multiple eBPF programs share one XDP hook — it inserts its own eBPF program that acts as a jump table, calling each registered program in sequence.

```
WITHOUT bpfman:
  eth0 XDP hook → YOUR program (only one allowed)

WITH bpfman dispatcher:
  eth0 XDP hook → bpfman DISPATCHER program
                      ↓
                  calls program A  (returns XDP_PASS)
                      ↓
                  calls program B  (returns XDP_DROP)
                      ↓
                  final verdict = XDP_DROP
```

---

## The Investigation Protocol (How Experts Read a New Codebase)

Use this exact method — it's called **"breadth-first skeleton, then depth-first focus"**:

```
PHASE 1: SKELETON (2 hours)
  Do NOT read implementation details yet.
  Open every file, read only:
  - Struct names
  - Public fn signatures
  - Module names
  Draw the map on paper.

PHASE 2: TRACE ONE PATH (4 hours)
  Pick ONE operation: Ebpf::load_file("x.o")
  Follow it from the top call to the raw syscall.
  Read EVERY line on that path.
  Annotate what you don't understand.

PHASE 3: FOLLOW THE DATA (ongoing)
  Pick ONE data structure: e.g. HashMap map
  Trace it:
  - How is it DEFINED in eBPF? (aya-ebpf/src/maps/)
  - How is it CREATED? (bpf_map_create syscall in sys/bpf.rs)
  - How is it WRITTEN from kernel side? (bpf_map_update_elem helper)
  - How is it READ from userspace? (aya/src/maps/hash_map.rs)

PHASE 4: READ THE TESTS
  The integration tests in test/ are DOCUMENTATION.
  They show you exactly how the API is meant to be used.
  Treat every test as a guaranteed-correct usage example.
```

---

## Concrete First Session — Exactly What to Open Right Now

```bash
git clone https://github.com/aya-rs/aya
cd aya

# Open these files in this order, in your editor:
# 1. aya/src/lib.rs              ← 5 min: public API overview
# 2. aya/src/ebpf.rs             ← 30 min: Ebpf + EbpfLoader structs
# 3. aya-obj/src/obj.rs          ← 20 min: ELF parsing
# 4. aya/src/sys/bpf.rs          ← 20 min: raw syscalls
# 5. aya/src/programs/xdp.rs     ← 20 min: one program type end-to-end
# 6. aya-ebpf-macros/src/lib.rs  ← 15 min: how #[xdp] works
```

**After this session**, you should be able to answer:
- How does `Ebpf::load_file()` become a `bpf()` syscall?
- What does the `#[xdp]` macro actually emit?
- How is the `.o` ELF file parsed without libbpf?

That's your first milestone. Everything else builds on top of it.

---

**Cognitive note:** What you are doing here is called **"tracing"** — following a single vertical path through a system from the highest abstraction (`load_file`) to the lowest (`bpf() syscall`). This is the single most powerful technique for understanding any large codebase. Computer scientists call this building a **"mental execution model"** — once you can simulate the execution of one operation in your head, the whole system becomes navigable.