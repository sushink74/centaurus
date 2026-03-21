# 🧭 Complete Roadmap to Contributing to Cilium

This is one of the most technically demanding open-source projects in the world. It sits at the intersection of **Linux kernel internals**, **networking**, **Kubernetes**, and **systems programming**. Let me break this down with the precision and structure a world-class mentor would use.

---

## 🔬 What Is Cilium, Fundamentally?

Before touching any code, you must understand the **mental model**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CILIUM ARCHITECTURE                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    USER SPACE (Go ~88%)                      │   │
│  │                                                              │   │
│  │  ┌──────────┐   ┌──────────────┐   ┌───────────────────┐   │   │
│  │  │  cilium  │   │   operator   │   │   hubble-relay    │   │   │
│  │  │  daemon  │   │ (k8s events) │   │  (observability)  │   │   │
│  │  └────┬─────┘   └──────────────┘   └───────────────────┘   │   │
│  │       │                                                      │   │
│  │       │  generates & loads BPF programs into kernel         │   │
│  └───────┼──────────────────────────────────────────────────────┘  │
│          │                                                          │
│  ┌───────▼──────────────────────────────────────────────────────┐  │
│  │                  KERNEL SPACE (C ~10%)                        │  │
│  │                                                               │  │
│  │   eBPF programs attached to:                                  │  │
│  │   ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │  │
│  │   │  Network IO │  │ XDP (eXpress │  │  Tracepoints /    │  │  │
│  │   │  (TC hooks) │  │  Data Path)  │  │  Kprobes          │  │  │
│  │   └─────────────┘  └──────────────┘  └───────────────────┘  │  │
│  │                                                               │  │
│  │   BPF Maps (shared memory between user ↔ kernel space)       │  │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

> **Key Mental Model:** Cilium's Go userspace **compiles and injects** C programs directly into the Linux kernel at runtime. The kernel programs then handle packets at wire speed. This is called **eBPF** — Extended Berkeley Packet Filter.

---

## 📐 The Technology Stack You Must Learn

```
LAYER 1: FOUNDATIONS (Learn First)
══════════════════════════════════
┌─────────────┬────────────────────────────────────────────┐
│ Technology  │ What you need to know                      │
├─────────────┼────────────────────────────────────────────┤
│ Go          │ Goroutines, channels, interfaces, testing  │
│ Linux net   │ netfilter, routing tables, sockets, tc     │
│ C (eBPF)    │ BPF C subset, maps, helpers, verifier      │
│ Kubernetes  │ pods, CNI plugins, CRDs, operators         │
│ Networking  │ L3/L4/L7, TCP/IP, VXLAN, BGP              │
└─────────────┴────────────────────────────────────────────┘

LAYER 2: CILIUM-SPECIFIC
════════════════════════
  eBPF maps, policy enforcement engine,
  identity model, datapath generation,
  Hubble flow monitoring, ClusterMesh
```

---

## 🗺️ Repository Structure — Where Everything Lives

```
cilium/
├── daemon/          ← The HEART: cilium-agent (Go daemon running on each node)
├── pkg/             ← Core library packages (policy, datapath, endpoint mgmt)
│   ├── policy/      ← Network policy engine — very rich Go code
│   ├── datapath/    ← Abstraction layer between Go and BPF programs
│   ├── endpoint/    ← Per-workload state management
│   ├── maps/        ← BPF map definitions and Go wrappers
│   └── proxy/       ← L7 proxy (envoy integration)
├── bpf/             ← ALL kernel-side C code (eBPF programs)
│   ├── lib/         ← BPF helper libraries in C
│   └── bpf_lxc.c   ← Main eBPF program for container traffic
├── operator/        ← Kubernetes operator (watches CRDs, syncs state)
├── hubble/          ← Observability layer
├── hubble-relay/    ← Aggregates flow data from all nodes
├── cilium-cli/      ← CLI tool (your first contribution playground)
├── plugins/         ← CNI plugin entry point
├── test/            ← Integration & e2e tests
└── Documentation/   ← Sphinx-based docs (easy first contributions)
```

---

## 🎯 Your Learning & Contribution Path (Phase by Phase)

```
PHASE 0          PHASE 1          PHASE 2          PHASE 3
FOUNDATIONS  →   READ CODE    →   SMALL FIX    →   FEATURES
(4-8 weeks)      (2-4 weeks)      (2-4 weeks)      (ongoing)
    │                │                │                │
    ▼                ▼                ▼                ▼
  Go deep         Understand       good-first       pkg/policy
  eBPF basics     daemon/          issue labels     bpf/ C code
  k8s basics      pkg/policy       docs fixes       new maps
```

---

## ⚡ PHASE 0 — The Non-Negotiable Foundations

### A) Go (Your Primary Language Here)

Since you know Go, focus on **Go's concurrency patterns as used in Cilium**:

```go
// Cilium heavily uses this pattern — "controller" loops
// A controller is: infinite loop + mutex-protected shared state + channels
// CONCEPT: "controller" = a goroutine that watches for changes and reacts

// Example mental model of how cilium-agent works:
go func() {
    for {
        select {
        case event := <-k8sEvents:      // Kubernetes told us something changed
            processPolicy(event)        // Recompute BPF policy maps
        case <-ctx.Done():              // Shutdown signal
            return
        }
    }
}()
```

### B) eBPF (The Core Kernel Technology)

> **What is eBPF?**
> eBPF = a virtual machine inside the Linux kernel. You write small C programs, compile them to BPF bytecode, and inject them into the kernel. The kernel **verifies** these programs are safe (no infinite loops, bounded memory) before running them.

```
  Your C Code         BPF Bytecode         Linux Kernel
  ┌──────────┐       ┌──────────────┐      ┌─────────────┐
  │bpf_lxc.c │──────▶│  clang/LLVM  │─────▶│  verifier   │
  │          │       │  compiles    │      │  checks     │
  └──────────┘       └──────────────┘      │  safety     │
                                           └──────┬──────┘
                                                  │
                                          ┌───────▼──────┐
                                          │  JIT compile │
                                          │  → native    │
                                          │  machine code│
                                          └──────────────┘
                                          runs at wire speed!
```

**Start here for eBPF learning:**
- `https://ebpf.io/what-is-ebpf/` — conceptual
- `https://github.com/lizrice/learning-ebpf` — practical book
- Cilium's own `bpf/` folder C files — read `bpf/lib/common.h` first

### C) Linux Networking Concepts (Critical Vocabulary)

```
TERM           │ MEANING
───────────────┼────────────────────────────────────────────────────
TC hook        │ Traffic Control — attachment point in kernel's
               │ network stack where BPF programs intercept packets
XDP            │ eXpress Data Path — even earlier hook, at NIC driver
               │ level. Fastest possible packet processing.
Veth pair      │ Virtual Ethernet — two linked virtual NICs. When
               │ you send to one end, it comes out the other.
               │ Used to connect container to host network.
netns          │ Network Namespace — isolated network stack.
               │ Each container has its own.
VXLAN          │ Virtual eXtensible LAN — overlay tunneling protocol
               │ to span L2 networks over L3 (used in cloud)
CNI            │ Container Network Interface — the plugin spec that
               │ Kubernetes calls to set up pod networking
CRD            │ Custom Resource Definition — Kubernetes extension
               │ mechanism. Cilium defines its own k8s resources.
Identity       │ Cilium's concept: a numeric ID assigned to a GROUP
               │ of pods sharing the same labels/security policy
───────────────┴────────────────────────────────────────────────────
```

---

## 🏗️ PHASE 1 — Reading the Codebase Intelligently

### Start With These Files In Order:

```
READING ORDER (follow the data flow of a packet entering a pod)
════════════════════════════════════════════════════════════════

Step 1: plugins/cilium-cni/main.go
        └─ Entry point when k8s adds a new pod. CNI plugin called.
           Understand: what happens when a pod starts?

Step 2: daemon/cmd/daemon.go
        └─ Main daemon initialization.
           Key function: NewDaemon()
           This is where everything is wired together.

Step 3: pkg/endpoint/endpoint.go
        └─ An "Endpoint" = Cilium's representation of a container.
           One endpoint per pod. Study the Endpoint struct.

Step 4: pkg/policy/distillery.go
        └─ How security policies get "distilled" into BPF maps.
           This is the bridge between Go policy objects and kernel.

Step 5: bpf/bpf_lxc.c
        └─ The actual eBPF program that runs in kernel for each
           container (lxc = Linux Container).
           Handle_xgress() is the main packet processing function.

Step 6: pkg/datapath/loader/loader.go
        └─ How Go compiles and loads the C BPF programs at runtime.
```

### How to Read a Large Codebase (Mental Model)

```
DON'T do this:                    DO this instead:
─────────────                     ────────────────
Read file by file                 Pick ONE data flow and trace it
Top to bottom                     Ask: "what happens when X occurs?"
Try to understand everything      Build a mental call graph
at once
```

Use this technique: **"Tracing a Pod Startup"**
```
$ git clone https://github.com/cilium/cilium
$ cd cilium

# Use grep/ripgrep to trace function calls:
$ rg "func.*endpoint.*Created" --type go
$ rg "RegeneratePolicy" --type go

# Read tests — they document intended behavior:
$ cat pkg/policy/distillery_test.go
```

---

## 🔧 PHASE 2 — Your First Contributions

### Finding Good First Issues

```
On GitHub, filter issues:
  label:"good first issue"
  label:"help wanted"
  label:"kind/bug"  +  "area/documentation"

URL: https://github.com/cilium/cilium/issues?q=label%3A%22good+first+issue%22
```

### Best First Contribution Areas (Ranked by Difficulty)

```
DIFFICULTY    AREA                    WHAT TO DO
──────────────────────────────────────────────────────────────
⭐ (Easy)    Documentation/          Fix typos, improve
             docs.cilium.io          examples, add clarity
             (Sphinx RST files)

⭐⭐         cilium-cli/             Add a CLI flag, improve
             (Go, self-contained)    error messages, add tests

⭐⭐         pkg/logging/            Improve log messages,
             pkg/metrics/            add Prometheus metrics

⭐⭐⭐       pkg/policy/             Policy engine bugs —
             (complex Go)            rich logic, testable

⭐⭐⭐⭐     daemon/ + pkg/          Core agent features
             datapath/

⭐⭐⭐⭐⭐   bpf/ C code             Kernel-level changes —
             (eBPF C programs)       requires deep expertise
──────────────────────────────────────────────────────────────
```

**→ Start with `cilium-cli/` — it is a standalone Go CLI tool,
   well-tested, and changes here don't risk kernel panics.**

---

## 🛠️ Development Environment Setup

```bash
# Prerequisites:
# - Linux (Ubuntu 22.04 recommended — eBPF requires Linux kernel ≥ 5.10)
# - Go 1.22+
# - Docker
# - kind (Kubernetes IN Docker)
# - clang/LLVM (for compiling BPF C code)

# 1. Fork the repo on GitHub, then clone YOUR fork:
git clone https://github.com/YOUR_USERNAME/cilium
cd cilium

# 2. Build the Go daemon:
make build

# 3. Run unit tests (start here — no k8s needed):
make tests

# 4. For integration testing, set up a local k8s cluster:
# kind = Kubernetes IN Docker — runs k8s inside containers locally
make kind             # Creates a local k8s cluster in Docker
make kind-image       # Builds your dev Cilium image
make kind-install-cilium  # Deploys your dev build into kind cluster

# 5. Run the connectivity tests:
cilium connectivity test
```

---

## 🧪 Understanding Cilium's Testing Pyramid

```
                        ╔═══════════════╗
                        ║  E2E Tests    ║  ← Slowest, most realistic
                        ║  (test/)      ║    Real k8s cluster + traffic
                        ╚═══════════════╝
                    ╔═══════════════════════╗
                    ║  Integration Tests    ║  ← Needs Linux + some k8s
                    ║  (pkg/.../*_test.go)  ║
                    ╚═══════════════════════╝
                ╔═══════════════════════════════╗
                ║       Unit Tests              ║  ← Fastest, start here
                ║  (pkg/policy/*_test.go etc.)  ║    Pure Go, no deps
                ╚═══════════════════════════════╝

# Run just unit tests while learning:
go test ./pkg/policy/...
go test ./pkg/endpoint/...
go test ./cilium-cli/...
```

---

## 🧠 Cognitive Strategy — How to Learn a System This Complex

### The "Compression" Mental Model

> From the field of **meta-learning**: Large codebases can't be understood linearly. Instead, build **compressed mental models** — simplified representations that you refine over time. Your first model of Cilium is: "Go daemon generates and loads C BPF programs into kernel to handle network packets." That's enough to start. Add detail iteratively.

### Deliberate Practice Approach for Open Source

```
WEEK 1-2:  Read 1 file deeply per day. Write a 3-sentence summary.
           Don't move on until you can explain it to a rubber duck.

WEEK 3-4:  Pick ONE bug from "good first issue". Don't fix it yet.
           Just REPRODUCE it. Understand WHY it happens.
           This teaches you more than fixing 10 random bugs.

WEEK 5-6:  Write the fix. Write a TEST first (TDD principle).
           The test forces you to understand the contract.

WEEK 7+:   Submit PR. The code review IS the education.
           Maintainers' comments are worth more than any tutorial.
```

### Key Cognitive Principle: **Chunking**

> In psychology, **chunking** means your brain groups related information into a single unit. Experts see "policy compilation" as ONE chunk; beginners see 500 lines of code. To develop expert-level chunking in Cilium:
> - Name the patterns you see: "ah, this is the observer pattern"
> - Group subsystems mentally: "BPF loading, policy engine, endpoint mgmt"
> - After 2-3 months of daily reading, you'll navigate the codebase instinctively

---

## 📋 Contribution Workflow

```
YOUR FORK                    UPSTREAM CILIUM
    │                              │
    │  1. Fork on GitHub           │
    │  2. git clone your fork      │
    │  3. git remote add upstream  │
    │     github.com/cilium/cilium │
    │                              │
    ▼                              │
  Create branch                   │
  git checkout -b fix/my-bug       │
    │                              │
    ▼                              │
  Make changes                    │
  Write/update tests               │
  Run: make lint                   │
  Run: go test ./...               │
    │                              │
    ▼                              │
  git commit -s                   │  ← -s adds DCO sign-off (REQUIRED)
  (follow commit message format)   │
    │                              │
    ▼                              │
  git push origin fix/my-bug      │
    │                              │
    │  Open Pull Request ──────────▶
    │                              │
    │◀──── Code Review ────────────│
    │                              │
    │  Address feedback            │
    │  git push (updates PR)       │
    │                              │
    │◀──── Merge! ─────────────────│
```

**Commit message format (strictly enforced):**
```
area: short description of change

Longer explanation of WHY this change is needed,
not just WHAT it does.

Fixes: #1234
Signed-off-by: Your Name <email@example.com>
```

---

## 🌐 Community Engagement (Non-Optional)

```
PLATFORM           ACTION                          PRIORITY
──────────────────────────────────────────────────────────
Slack              Join #development channel       HIGH
slack.cilium.io    Ask questions, offer help
                   Introduce yourself

GitHub Discussions Search before asking            HIGH
                   Answer others' questions

Weekly Zoom calls  Wednesday 5pm Zurich time       MEDIUM
                   Just listen for first month

eCHO Livestream    YouTube, weekly episodes        MEDIUM
                   Watch past episodes on BPF/Cilium
──────────────────────────────────────────────────────────
```

> **Psychological principle — "Learning in public"**: The act of explaining concepts to others on Slack (even imperfectly) accelerates your own understanding dramatically. This is called the **Protégé Effect** — teaching forces you to identify gaps in your knowledge.

---

## 🎯 Your 30-Day Action Plan

```
DAY 1-3:    Setup environment. Build Cilium locally.
            Read README.rst + docs.cilium.io/overview

DAY 4-7:    Read plugins/cilium-cni/main.go
            Read daemon/cmd/daemon.go (just the init sequence)

DAY 8-14:   Read pkg/policy/ with its tests
            Build the mental model of how policies become BPF maps

DAY 15-20:  Find 1 "good first issue" in cilium-cli/
            Read all related code. Reproduce the problem.

DAY 21-28:  Write the fix + test. Open a DRAFT PR.
            Ask in Slack: "I'm working on issue #X, does my
            approach make sense?"

DAY 29-30:  Polish PR. Submit. Learn from review feedback.
```

---

## 💡 The Hidden Insight Most People Miss

> Cilium's complexity is **not random** — it maps directly onto the complexity of the Linux kernel's networking subsystem. Every confusing abstraction in `pkg/datapath/` exists because the kernel forces it. Once you understand WHY eBPF has a verifier, WHY BPF maps exist, WHY the daemon regenerates programs on policy change — the code becomes obvious. **Study the kernel concepts, and the Go code will explain itself.**

The top 1% of systems programmers don't just read code — they understand the **physical and kernel constraints** that force the code to be structured a certain way. That's the mindset to develop here.

# 🎯 Cilium Entry Point — Complete Code Investigation Guide

Let me give you the **exact files, exact functions, exact call chain** — like a surgeon's map.

---

## 🧭 The Big Truth First

Cilium has **two major entry points** depending on what event triggers it:

```
EVENT                          ENTRY POINT FILE
──────────────────────────────────────────────────────────────
cilium-agent process starts  → daemon/cmd/daemon_main.go
                                  └── main() → runDaemon()

A Kubernetes pod is created  → plugins/cilium-cni/main.go
                                  └── main() → cmdAdd()
──────────────────────────────────────────────────────────────
```

**Start with #1** — the daemon is the brain of the entire system.

---

## 🔬 The Exact Call Chain — Follow This Like a Treasure Map

```
FILE: daemon/cmd/daemon_main.go
════════════════════════════════════════════════════════════════

main()
  │
  ├── cobra.OnInitialize(option.InitConfig())    ← reads config
  │       │
  │       └── FILE: pkg/option/config.go
  │               ReadDirConfig()
  │               MergeConfig()
  │               viper.MergeConfigMap()
  │
  └── Execute()  ← cobra CLI framework starts
          │
          └── runDaemon()   ← THE REAL START
                │
                ├── enableIPForwarding()
                │     └── turns on ip_forward in Linux kernel
                │         (tells kernel to route packets between interfaces)
                │
                ├── k8s.Init()
                │     └── FILE: pkg/k8s/init.go
                │         connects to Kubernetes API server
                │
                ├── NewDaemon()   ◄──── MOST IMPORTANT FUNCTION
                │     └── FILE: daemon/cmd/daemon.go
                │         (see full breakdown below)
                │
                ├── d.initRestore(restoredEndpoints)
                │     └── re-loads existing pods after restart
                │
                ├── d.initHealth()
                │     └── starts cilium-health checker
                │
                ├── srv.Serve()
                │     └── starts the HTTP REST API server
                │         other tools talk to cilium-agent through this
                │
                └── d.launchHubble()
                      └── starts observability subsystem
```

---

## 🏗️ NewDaemon() — The Heart of Everything

This single function initializes the **entire agent**. This is where all subsystems are born.

```
FILE: daemon/cmd/daemon.go
════════════════════════════════════════════════════════════════

NewDaemon()
  │
  ├── [BPF Map Initialization]
  │     ctmap.InitMapInfo()      ← connection tracking map
  │     policymap.InitMapInfo()  ← policy rules map
  │     lbmap.Init()             ← load balancer map
  │
  │   CONCEPT — "BPF Map":
  │   A BPF map is a key-value store shared between:
  │   - Go userspace (reads/writes policy/config)
  │   - C kernel BPF programs (reads during packet processing)
  │   Think of it as: shared memory between Go and kernel
  │
  ├── [Node Discovery]
  │     nodediscovery.NewNodeDiscovery()
  │     └── discovers this node's IP, name, labels in k8s
  │
  ├── [Daemon Struct Created]
  │     d := &Daemon{}
  │     └── this struct IS the agent — holds all state
  │
  ├── [Kubernetes Watcher]
  │     d.k8sWatcher = watchers.NewK8sWatcher()
  │     └── starts watching k8s API for:
  │         - new pods
  │         - new network policies
  │         - new services
  │         → each event triggers BPF map updates
  │
  ├── d.initMaps()
  │     FILE: daemon/cmd/datapath.go
  │     │
  │     ├── lxcmap.LXCMap.OpenOrCreate()
  │     │     "lxc" = Linux Container
  │     │     This map: endpoint_id → network interface info
  │     │
  │     ├── ipcachemap.IPCache.OpenParallel()
  │     │     This map: IP address → security identity
  │     │     (every pod's IP gets a numeric identity)
  │     │
  │     └── ctmap.GlobalMaps.Create()
  │           connection tracking: remembers active TCP connections
  │
  ├── d.svc.RestoreServices()
  │     FILE: pkg/service/service.go
  │     Reloads Kubernetes Service → BPF load balancer rules
  │
  └── d.fetchOldEndpoints()
        FILE: daemon/cmd/state.go
        Re-reads existing container endpoints from disk
        (survives agent restart without dropping traffic)
```

---

## 🧬 The Hive/Cell Architecture (Critical Concept)

Cilium uses a **dependency injection** framework called **Hive** (their own library). This is NOT standard Go. You MUST understand it.

> **What is Dependency Injection?**
> Instead of function A calling function B directly, you declare "A needs X" and "B provides X". A framework wires them together automatically. This makes subsystems independently testable.

```
TRADITIONAL GO:             CILIUM'S HIVE PATTERN:
──────────────              ────────────────────
func main() {               var MyCell = cell.Module(
  db := NewDB()               "my-subsystem",
  cache := NewCache(db)       cell.Provide(NewDB),      // provides DB
  svc := NewSvc(cache)        cell.Provide(NewCache),   // needs DB, provides Cache
}                             cell.Provide(NewSvc),     // needs Cache
                            )
                            // Hive figures out the order automatically
```

In `daemon/cmd/daemon_main.go`, you'll see this pattern:

```go
// This is how Cilium registers all its subsystems — each "cell" is a module
var (
    daemonCell       = cell.Module("daemon", "Legacy Daemon", ...)
    daemonConfigCell = cell.Module("daemonconfig", "Config", ...)
    // dozens more cells...
)
```

**When reading code, follow `cell.Provide(SomeFunction)` → find `SomeFunction` → that's the real implementation.**

---

## 🗺️ ASCII Map: How a Network Packet Flows Through Cilium

Understanding this flow tells you WHERE to look for each feature:

```
  POD A sends a packet to POD B
  ══════════════════════════════════════════════════════════════

  Pod A (container)
    │
    │  [veth pair — virtual cable connecting pod to host]
    │
    ▼
  Host network namespace
    │
    ├── TC ingress hook fires (Traffic Control)
    │     └── BPF program: bpf/bpf_lxc.c  ← KERNEL C CODE
    │           │
    │           ├── Lookup source IP in ipcache map
    │           │     → gets security identity of Pod A
    │           │
    │           ├── Check policy map
    │           │     → is Pod A allowed to talk to Pod B?
    │           │
    │           ├── If DROP: packet discarded here
    │           │
    │           └── If ALLOW: packet forwarded
    │
    ▼
  Routing decision
    │
    ├── Same node? → direct veth to Pod B
    └── Different node? → VXLAN tunnel to remote node
              │
              └── Remote node runs same BPF check on arrival
    │
    ▼
  Pod B (container) receives packet

  ══════════════════════════════════════════════════════════════
  GO CODE (daemon) manages the BPF MAPS that the C code reads.
  They never call each other directly — maps are the interface.
```

---

## 📁 Exact Files to Read, in This Exact Order

```
DAY 1  →  daemon/cmd/daemon_main.go
          Focus on: init() and runDaemon() functions only
          Question to answer: "What is initialized first?"

DAY 2  →  daemon/cmd/daemon.go
          Focus on: NewDaemon() function — read top to bottom
          Question to answer: "What is a Daemon struct?"

DAY 3  →  pkg/endpoint/endpoint.go
          Focus on: the Endpoint struct definition (top of file)
          CONCEPT: "Endpoint" = one running container/pod
          Question to answer: "What fields does an Endpoint have?"

DAY 4  →  daemon/cmd/datapath.go
          Focus on: initMaps() function
          Question to answer: "What BPF maps exist and what do they store?"

DAY 5  →  pkg/policy/distillery.go
          Focus on: distillPolicy() function
          CONCEPT: "distill" = convert k8s NetworkPolicy objects
                   into BPF map entries the kernel can read
          Question to answer: "How does a YAML policy become a BPF rule?"

DAY 6  →  bpf/bpf_lxc.c
          Focus on: handle_xgress() — main packet handler
          Read top 100 lines only — just the includes and first function
          Question to answer: "What does the BPF program do per-packet?"

DAY 7  →  plugins/cilium-cni/main.go
          Focus on: cmdAdd() — called when a pod starts
          Question to answer: "What happens when a new pod is created?"
```

---

## 🛠️ Practical: How to Navigate With Tools

```bash
# Clone the repo first:
git clone https://github.com/cilium/cilium
cd cilium

# ── TECHNIQUE 1: Find where a function is defined ──
grep -rn "func NewDaemon" --include="*.go"
# Output: daemon/cmd/daemon.go:func NewDaemon(...) {...}

# ── TECHNIQUE 2: Find all callers of a function ──
grep -rn "NewDaemon(" --include="*.go"
# Shows you every place NewDaemon is called

# ── TECHNIQUE 3: Understand a BPF map ──
grep -rn "LXCMap" --include="*.go" | head -20
# See all places the LXC (container) map is used

# ── TECHNIQUE 4: Trace an event (e.g., pod creation) ──
grep -rn "EndpointAdd\|endpointAdd\|AddEndpoint" --include="*.go"

# ── TECHNIQUE 5: Find what a Cell provides ──
grep -rn "cell.Provide" daemon/cmd/daemon_main.go

# ── TECHNIQUE 6: Read just function signatures quickly ──
grep -n "^func " daemon/cmd/daemon.go
# Lists every function in the file — gives you the API surface
```

---

## 🧠 The One Mental Model That Unlocks Everything

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   cilium-agent (Go) = a COMPILER + MAP MANAGER             │
│                                                             │
│   It does NOT process packets directly.                     │
│   It COMPILES C programs and FILLS BPF MAPS.               │
│   The KERNEL runs the C programs on every packet.          │
│                                                             │
│   Go daemon role:   "What are the rules?"                  │
│   BPF C code role:  "Apply the rules at wire speed"        │
│                                                             │
│   BPF Maps role:    "The contract between them"            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Once this model clicks — every file in the repo makes sense immediately. The Go code is **configuration management**. The C code is **execution**. The BPF maps are **the shared data structure** between them.

> **Deliberate Practice Principle:** Don't just read — after each day, close the code and write a paragraph explaining what you read, in your own words. This is called **retrieval practice**, and neuroscience shows it triples retention compared to passive reading. You're training your brain to compress these patterns into intuition.