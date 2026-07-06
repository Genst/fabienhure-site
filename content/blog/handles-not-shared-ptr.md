---
title: "Handles, not shared_ptr"
date: 2026-07-06
draft: false
summary: "Why shared_ptr is the wrong layer for objects with persistent identity and cross-language lifetime, and how Quantara's handle registry - generational indices, lock-free refcounting packed into one atomic word, four typed wrappers - solves the problem properly."
tags:
  - C++
  - Quantara
  - Concurrency
  - Memory
categories:
  - Quantara
  - Engineering
series:
  - Quantara Internals
math: false
mermaid: false
---

An equity spot inside a market object is referenced by several other objects. It gets written to disk and read back. Python holds it, Excel holds it, and neither cares where it lives in memory. Most C++ codebases would still hand it to a `shared_ptr`, because `shared_ptr` is a default, not a decision.

This post is about the layer Quantara built instead: a registry of explicit handles, with a refcount and a generation packed into a single atomic word. None of it is clever for its own sake. Each piece was forced by the one before it, and the point of writing it up is to show the chain.

## What is actually being modelled

The objects that matter in an analytics library, market data items, reference data, calibrated curves, instruments, share a set of properties that decide how they must be held.

They have a stable identity. A logical identity that means the same thing across runs, across machines, and inside an archive. They persist: serialised today, deserialised next week, still the same object. Other objects refer to them by that identity rather than by address; a schedule doesn't hold a pointer to its calendar, it refers to the calendar. Their lifetimes cross language boundaries, so a Python session or an Excel sheet can hold one longer than any C++ scope does. And they are immutable once registered. Quantara enforces this at the `Storable` level: a registered object is never mutated in place, it is replaced. That single invariant is what later lets readers touch these objects without locking.

An address answers exactly one question: where the object is right now. For this class of objects, that is the least durable fact about them. It changes across runs and means nothing the moment it leaves the process, which makes it useless as the thing other objects hold, serialise, or compare.

So the problem was never "how do I share ownership of this pointer". It is "how do I give this object a durable identity, let anything in the system ask whether it still exists, and keep it alive exactly as long as anyone, in any of three runtimes, still needs it". That is not a smart pointer problem. It is a registry problem.

Most codebases never build this layer. They get by with `shared_ptr` and a naming convention, and the gap shows up later as a serialisation subsystem that invents keys, a Python binding that pins objects forever, and a debugging session that ends in a dangling pointer nobody can explain. The rest of this post is the layer built properly.

## Where `shared_ptr` loses, specifically

Not in the abstract. `shared_ptr` is a perfectly good tool. It loses for this job, in five concrete ways.

Size and layout. A `shared_ptr` is 16 bytes in every owner, a pointer to the object and a pointer to a separately heap-allocated control block. The handle wrapper is 8. A market holds sorted vectors of hundreds of entries, each carrying handle fields; at that scale the difference is cache lines, not bytes.

Identity. Pointer equality changes across runs and across machines. As a durable identifier, an address is useless.

Cross-binding lifetime. A Python wrapper holding a `shared_ptr` has two options, both wrong: pin the C++ object for the entire interpreter session, or release it at a moment Python's ownership rules pick, not yours.

Archival round-trip. You cannot write a `shared_ptr` to disk and read it back. Every serialisation layer built on smart pointers ends up inventing a key scheme anyway. If you need the key regardless, start with the key.

Observability. "Is this object still alive" is a question other subsystems ask all the time. A `shared_ptr` can answer it for its own copies and nobody else. And the thread-safety it appears to offer is narrower than it looks: the control block is atomic, the object is not, and the registry you haven't built certainly isn't.

None of these is fatal alone. Together they say the refcount is at the wrong layer. Ownership isn't a property of each pointer that happens to be held; it is a property of the object's entry in the system.

## The table everyone builds first

The obvious fix is a registry. Everyone's first version looks the same:

```cpp
std::unordered_map<uint64_t, Storable*> g_objects;
uint64_t g_nextId = 1;
```

Hand out integers, keep the map, look objects up by ID. I reviewed code recently that reached for exactly this shape, and watched its author discover the gaps from the inside. Is this ID stale, or was it ever valid? Who deletes the object, and what happens to the other five holders of the same integer when they do? What happens when two threads register at once? What happens when the map rehashes while another thread is reading it?

Game engines answered most of this years ago. The pattern is the generational index: the handle packs an array slot index with a generation counter, the slot stores the current generation, and a lookup compares the two. Reusing a slot bumps the generation, which invalidates every outstanding handle to the previous occupant, structurally and for free. floooh's 2018 piece ["Handles are the better pointers"](https://floooh.github.io/2018/06/17/handles-vs-pointers.html) is the canonical write-up, and jlaumon's [`Handle<T>`](https://github.com/jlaumon/handle) is a clean public implementation. If you know that literature, the shape of what follows will be familiar.

The delta is ownership. A game-engine handle is a weak reference: one owner destroys the object, and the generation bits answer a single question, "is it still alive?". Quantara's handles have to answer a harder one: "keep it alive until three runtimes are done with it". C++ holds these objects, Python holds them, Excel holds them, and no single owner knows when the last user is finished. That forces reference counting. Reference counting under concurrent readers forces the counting to be lock-free. And lock-free refcounting next to a generation counter forces the two into the same atomic word, which is where the design gets interesting.

## Generation and refcount in one word

The handle itself is a 64-bit integer: generation in the high 32 bits, slot index in the low 32, zero reserved as the invalid handle. Packing buys the properties a reference should have. Copy is a register move, equality is integer comparison, and the whole thing crosses the Python and Excel boundaries as a plain number.

```cpp
using Handle = uint64_t;    // (generation:32 | index:32); 0 is INVALID_HANDLE

// Per-slot record. Generation and refcount merged into ONE atomic word;
// alignas(64) keeps adjacent slots off each other's cache lines.
struct alignas(64) SSlot
{
    std::atomic<uint64_t> m_state { 0 };     // (gen << 32) | refcount; gen == 0 means never used
    std::atomic<Storable*> m_item { nullptr };
};

static constexpr uint64_t EncodeState(uint32_t generation, uint32_t refcount) noexcept
{ return (static_cast<uint64_t>(generation) << 32) | static_cast<uint64_t>(refcount); }

static constexpr uint32_t GenerationOf(uint64_t state) noexcept { return static_cast<uint32_t>(state >> 32); }
static constexpr uint32_t RefCountOf(uint64_t state) noexcept   { return static_cast<uint32_t>(state); }
```

Each slot pairs the payload pointer with a single state word holding the generation and the refcount together. The `alignas(64)` is not decoration. Adjacent slots land on separate cache lines, so threads working on unrelated handles never contend on one. That padding quadruples the per-slot footprint, paid deliberately.

The merged word is the part I'd call a contribution, so I'll state it plainly. Generational indices are prior art; neither of the references in the previous section carries ownership semantics. Their generation bits answer "is it alive", and a single owner decides when it dies. Once three runtimes can hold the same object, the slot needs a refcount, and the refcount cannot live in a separate atomic from the generation. Between two separate writes there is a window: cleanup drives the count to zero, and before it gets to bump the generation, a racing `AddRef` on the old handle sees the old generation still standing and revives a slot the cleanup thread already owns. You can try to defend that window with protocol. Packing both into one word removes it structurally. There is no "between".

Taking a reference is a CAS loop, not a `fetch_add`:

```cpp
uint64_t state = slot->m_state.load(std::memory_order_acquire);
for (;;)
{
    if (GenerationOf(state) != generation) return false;            // stale handle
    const uint32_t refcount = RefCountOf(state);
    if (refcount == UINT32_MAX) { assert(false); return false; }    // saturated
    if (refcount == 0)          { assert(false); return false; }    // use-after-free; no resurrection
    const uint64_t next = EncodeState(generation, refcount + 1);
    if (slot->m_state.compare_exchange_weak(state, next,
            std::memory_order_acq_rel, std::memory_order_acquire))
        return true;
    // CAS failed; `state` now holds the observed value. Re-check and retry.
}
```

Three exits sit before the increment. The generation doesn't match: stale handle, fail cleanly. The count is saturated: fail rather than wrap. The count is zero: someone is using a dead object, and the assert makes that loud instead of quietly reviving it. Why a loop and not `fetch_add(1)`? Because the check and the mutation must act on the same observed word. A pre-check followed by `fetch_add` lets two threads both read `UINT32_MAX - 1`, both pass, and wrap the count to zero; the next release is a use-after-free. And a generation held elsewhere could be bumped between the read and the add. The CAS re-validates against the exact value it is about to replace. The same argument kills `fetch_sub` on the release side: decrementing a slot already at zero wraps the count to `UINT32_MAX` and pins the slot permanently.

Which leaves the move the whole design exists for. The last release:

```cpp
uint64_t next;
if (refcount == 1)
{
    // Drive the count to zero AND bump the generation in the same CAS.
    next = EncodeState(BumpGeneration(generation), 0);
    wasLast = true;
}
else
{
    next = EncodeState(generation, refcount - 1);
}
if (slot->m_state.compare_exchange_weak(state, next, /*...*/)) break;

// After a successful last-reference CAS, any racing AddRef on the old handle
// observes the new generation and bails BEFORE this thread strips the payload:
Storable* item = slot->m_item.exchange(nullptr, std::memory_order_acq_rel);
```

The CAS does two things at once because the bug lives between them. Driving the count to zero makes this thread the sole owner of the payload. Bumping the generation invalidates the handle everywhere else. In the same instruction. Any `AddRef` racing on the old handle now fails the generation check before this thread touches the payload pointer. The same invariant runs forward at registration: the payload pointer is published before the live state word, and readers check the state word before loading the pointer. No epoch reclamation, no hazard-pointer machinery. Slot memory is never unmapped while the registry lives, and the state word gates everything else. One atomic, one truth.

## The lock that remains

A mutex survives, scoped to what genuinely needs it. Registration, removal, and free-list maintenance all change the shape of the registry, and they take the structural mutex. Everything that merely consults the registry stays off it. Taking and dropping references, resolving a handle to its object, asking whether a handle is valid: atomic loads and CAS loops on the slot, nothing more. That separation reads like a performance decision, and it is one, but it is first a correctness clarification. The lock is for changing the registry's shape. Reading from it shouldn't need to ask permission. In a resolved market being priced across a thread pool, every lookup is three atomic loads, and no reader queues behind a writer that isn't there.

One admitted asymmetry: the reverse lookup, pointer to handle, still takes the mutex. It is a facility no hot path uses, and it stays locked until measurement says otherwise.

Even the last-reference path holds the lock only for bookkeeping. The claiming CAS is lock-free; the mutex covers the free-list push and the map erase; then it is dropped before the payload is deleted. The order matters because destructors recurse. A `Storable`'s destructor may hold handle references of its own, each of which releases on destruction, and if one of those releases turns out to be a last reference it will want the structural mutex too. Deleting under the lock would deadlock on that nested acquire. So the slot is claimed, the books are updated, the lock is dropped, and only then does the object die.

## Four wrappers, four intentions

One integer, four C++ types layered over it. Not for variety; each encodes a different answer to a question APIs keep having to ask.

`HandleRef<T>` is the base: eight bytes of handle, RAII, nullable. Copy takes a reference, destruction releases one, move transfers without touching the count. `RequiredRef<T>` narrows it: validity asserted at construction, never null afterwards, the standard production ownership type. `OptionalRef<T>` goes the other way: may legitimately be empty, and the caller is expected to check. And `THandle<T>` is the non-owning view, a typed const pointer for hot loops where paying a CAS per copy would be waste and the caller guarantees the object stays pinned.

The payoff is in signatures.

```cpp
// The signature says: this dependency must exist. No null check in the body.
Expected<double> Price(RequiredRef<EquityListing> listing, const DateTime& t);

// The signature says: absence is expected. The compiler makes you decide.
Schedule Generate(const SPeriod& period, OptionalRef<Calendar> calendar);
```

The first says this dependency must exist, and the body contains no null check because none is needed. The second says absence is expected here, and the compiler won't let you forget to decide. "Is this allowed to be missing?" stops being a convention buried in a comment and becomes something the type system answers. An `OptionalRef` converts implicitly from a `RequiredRef`, because always-valid into maybe-valid is safe. There is no implicit reverse conversion, and that is the point.

Casting follows the same discipline. A dynamic cast from a `RequiredRef<T>` returns an `OptionalRef<U>`, because a failing cast must remain expressible. A static cast is constrained at compile time to hierarchy-related types; the handle integer carries no type of its own, so an unconstrained cast would happily rewrap anything as anything. A debug assert catches the one case the constraint can't: right hierarchy, wrong instance.

## The adopt-vs-AddRef trap

When `Register` allocates a fresh slot, it stores the payload and sets the refcount to 1. That first reference belongs to the slot itself, the intrinsic owning reference the wrapper is meant to take over. And there is the trap. If the wrapper constructed around that fresh handle also takes a reference, the count is now 2. The wrapper's destruction releases exactly once. The count settles at 1 and stays there, and the object is never freed for the life of the process.

Consider what kind of bug that is. It isn't a crash, and it isn't a leak valgrind will flag, because the object remains reachable through the manager. If test teardown force-clears the registry, the test suite won't see it either. Silent memory growth, with a perfectly plausible explanation attached to every byte.

I know because Quantara shipped it, internally. An early version of the market resolution path wrapped freshly registered handles in the ordinary constructor. Every Storable in every Market leaked one refcount. Nothing failed; the tests tore the registry down between runs, which cleared the evidence each time.

The fix could have been a comment. "Do not AddRef handles returned by Register." Comments like that lose the fight eventually, because the losing code looks right: `RequiredRef<Currency>(handle)` reads as the obvious way to wrap a handle, compiles cleanly, and leaks. The fix that holds is at the type level.

```cpp
namespace NHandleManager::Internal
{
    // Tag selecting the adopting constructor. The bare `Internal` name (no `N`
    // prefix, deliberately breaking the project convention) is the visual
    // marker: production code has no business here.
    struct SAdoptHandleTag
    {
        explicit SAdoptHandleTag() = default;
    };
    inline constexpr SAdoptHandleTag s_adoptHandle {};
}

// Non-adopting: AddRefs. For handles received from a live owner.
explicit HandleRef(Handle handle);

// Adopting: takes over the slot's intrinsic refcount of 1. No AddRef.
HandleRef(Handle handle, NHandleManager::Internal::SAdoptHandleTag) noexcept;

// The canonical factory - the only production path from a fresh object to a wrapper.
template<typename T>
requires std::derived_from<T, Storable>
[[nodiscard]] RequiredRef<T> Register(std::unique_ptr<T> obj)
{
    Handle handle = GetHandleManager().Register(std::move(obj));   // slot refcount = 1
    return RequiredRef<T>(handle, Internal::s_adoptHandle);        // adopt - do NOT AddRef
}
```

Three pieces work together. A tag struct whose only job is selecting an overload. An adopting constructor that takes over the slot's intrinsic reference instead of adding one. And the canonical factory, the single production path from a fresh object to a wrapper, which is the only code that passes the tag. The tag lives in a namespace literally named `Internal`, deliberately breaking the project's naming convention so it reads as a warning. You can no longer write the buggy version by accident. You have to reach into `Internal` to write it at all, and at that point it is not an accident.

One more decision hides in the factory: failure is fatal. A null pointer or an exhausted pool aborts rather than returning. `RequiredRef`'s validity assert compiles out in release builds, so handing back a broken wrapper would defer the failure to first use, far from the cause. Aborting at the factory keeps the crash next to the bug.

A regression test pins the behaviour: after the last wrapper drops, the live-object count returns to baseline and the handle reports invalid. But the test is the second line of defence. The first is that the leaking version no longer compiles from ordinary code. Documentation rots. The type system doesn't.

## What handles don't solve

None of this dethrones `shared_ptr`. It remains the right tool for anything purely local, for objects without identity, for anything that will never see an archive. Handles are the right tool for one specific layer, and reaching for them elsewhere buys cost without benefit.

The costs are real. Copying an owning wrapper is a CAS, not a register move; hot loops use the non-owning view and let the caller pin. The registry reserves a fixed table up front and allocates slot slabs as they're first touched, and the 64-byte slot padding quadruples the per-slot footprint. The mental model is heavier than "the pointer keeps it alive". And a handle in a debugger is an opaque integer until you teach the debugger to decode it. The library ships a natvis that does, but that is a tool you have to build, not one you get for free.

The gamedev reader has been waiting to object since the floooh citation, so let me concede the point before it's raised. Game-engine handle systems store values inline, packed into same-type arrays, and harvest cache locality from iteration order. This registry cannot. The payloads are heterogeneous polymorphic objects, individually heap-allocated; the slabs hold slots, not values. Handles here buy identity and lifetime. They do not buy data-oriented layout, and pretending otherwise would claim a benefit the design deliberately traded away.

Two more honest edges. The generation counter can wrap: 2^32 register-release cycles concentrated on one slot, with a stale handle surviving the whole time, would alias. That ABA is documented and accepted rather than engineered away. And although the handle format leaves 32 bits for the slot index, the registry deliberately caps the pool at roughly sixteen million slots, enforced fatally, because the ceiling is sized so far past plausible usage that reaching it means something else is deeply wrong.

Which returns to the opening. `shared_ptr` was never the villain; the default was. If the thing you are modelling has a name that must survive a restart, or a lifetime that some other runtime gets a say in, then refcounting through smart pointers is bookkeeping at the wrong layer, and what you want is a registry with explicit handles. Once that decision is made, the rest of this post stops looking like cleverness. The merged atomic, the small lock, the wrapper types, the adopt tag: each was the forced move after the one before it.

Everything else in Quantara stands on this layer. Markets refer to their items by handle at runtime. Serialisation writes each object once and cross-references any repeats with archive-local IDs; loading re-registers the objects and hands back fresh handles. Parallel resolution leans on the lock-free reads described above. Those are their own posts.
