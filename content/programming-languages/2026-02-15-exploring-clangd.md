---
title: "Exploring Clangd: Why It Eats All Your RAM and What We Can Do About It"
date: 2026-02-15
tags:
  - clangd
  - llvm
  - cpp
  - tooling
---
If you write C++ and use an LSP-based editor, there's a decent chance clangd is the thing making your code navigation work. It's also, if you're anything like me, the thing that keeps getting murdered by earlyoom because it decided 8GB of RAM wasn't enough for a medium-sized project.

I got frustrated enough to actually read the source code. Here's what I found.

## What Clangd Actually Does With Your Memory

The architecture is pretty straightforward once you see it. Clangd runs one worker thread per open file (`TUScheduler.cpp`), and each worker maintains its own copy of the parsed AST. The critical data structure is `ASTCache` — an LRU cache that holds onto parsed ASTs even after you switch away from a file:

```cpp
// TUScheduler.cpp
ASTCache(unsigned MaxRetainedASTs) : MaxRetainedASTs(MaxRetainedASTs) {}

void put(Key K, std::unique_ptr<ParsedAST> V) {
    // ...
    LRU.insert(LRU.begin(), {K, std::move(V)});
    if (LRU.size() <= MaxRetainedASTs)
      return;
    // We're past the limit, remove the last element.
    std::unique_ptr<ParsedAST> ForCleanup = std::move(LRU.back().second);
    // ...
}
```

The default? **3 retained ASTs**. That might not sound like much, but a single AST for a file that includes half of Boost or a chunk of the standard library can easily be hundreds of megabytes. Three of those and you're knocking on the door of a GB just from idle caches.

On top of the AST cache, there are two index structures running concurrently:

- **Dynamic index** — rebuilt live as you edit, tracks symbols in open files
- **Background index** — crawls your entire project in the background, persists to disk

Both of these accumulate memory. The `ClangdServer::profile()` method reveals the breakdown:

```cpp
void ClangdServer::profile(MemoryTree &MT) const {
  if (DynamicIdx)
    DynamicIdx->profile(MT.child("dynamic_index"));
  if (BackgroundIdx)
    BackgroundIdx->profile(MT.child("background_index"));
  WorkScheduler->profile(MT.child("tuscheduler"));
}
```

But here's the thing — this profiling infrastructure is **only accessible through tracing**. There's a `$/memoryUsage` LSP extension, and the `maybeExportMemoryProfile()` function fires every 5 minutes if tracing is enabled, but neither of these does anything by default. You can't just ask clangd "how much memory are you using and why?"

## The Cleanup Mechanism (Or Lack Thereof)

Clangd does have a memory cleanup path. On glibc systems, there's a `--malloc-trim` flag (enabled by default) that periodically calls `malloc_trim()`:

```cpp
constexpr size_t MallocTrimPad = 20'000'000;  // leave 20MB headroom
return []() {
    if (malloc_trim(MallocTrimPad))
      vlog("Released memory via malloc_trim");
};
```

This runs roughly once per minute, triggered by the `ShouldCleanupMemory` debouncer. The problem is that `malloc_trim` only releases memory that's *already been freed* back to the OS. It doesn't actually free anything — if clangd is holding onto three fat ASTs in the LRU cache, `malloc_trim` can't touch them.

The other big lever is `--pch-storage=disk` (which is actually the default now). Preambles — the precompiled headers that clangd builds for the stable prefix of each file — can be stored on disk instead of in RAM. This is probably the single biggest memory saver available today.

## What's Missing: A Real Memory Limit

Here's what I think clangd actually needs: a `--memory-limit` flag. The idea is simple — you tell clangd "you get 2GB, figure it out," and when it crosses that threshold, it starts evicting ASTs from the cache and calling `malloc_trim` more aggressively.

The implementation would look something like:

1. Add a `--memory-limit` flag in `ClangdMain.cpp` (in MB, 0 = unlimited)
2. In `maybeCleanupMemory()`, read the process RSS from `/proc/self/statm`
3. If over the limit: drop `MaxRetainedASTs` to 0, force `malloc_trim(0)`, log the `MemoryTree` breakdown
4. If back under: restore normal retention

This is pretty light-touch. The `MemoryTree` infrastructure already exists and knows how to walk every component. The `ASTCache` already supports dynamic sizing. You'd basically be connecting plumbing that's already there. The reason earlyoom kills clangd is that clangd has no concept of a budget — it just accumulates state until something external intervenes.

## Template Specializations: A Hole in the Index

While reading through the code, I noticed something interesting about "go to implementations." Right now, if you invoke it on a virtual method, clangd finds all overrides. If you invoke it on a base class, it finds all derived classes. This works through the `RelationKind` system:

```cpp
// index/Relation.h
enum class RelationKind : uint8_t {
  BaseOf,
  OverriddenBy,
};
```

Two kinds of relations. That's it. But Clang's own indexer actually emits a *third* kind of relation that clangd completely ignores:

```cpp
// clang/include/clang/Index/IndexSymbol.h
RelationSpecializationOf = 1 << 19,
```

The `indexableRelation()` function in `SymbolCollector.cpp` is the gatekeeper, and it explicitly only maps `BaseOf` and `OverriddenBy`:

```cpp
std::optional<RelationKind> indexableRelation(const index::SymbolRelation &R) {
  if (R.Roles & static_cast<unsigned>(index::SymbolRole::RelationBaseOf))
    return RelationKind::BaseOf;
  if (R.Roles & static_cast<unsigned>(index::SymbolRole::RelationOverrideOf))
    return RelationKind::OverriddenBy;
  return std::nullopt;  // everything else is thrown away
}
```

This means if you have a class template and several explicit or partial specializations, "go to implementations" does nothing. The data is right there in the AST — Clang is emitting `RelationSpecializationOf` — but clangd drops it on the floor.

Fixing this would involve:

1. Adding `SpecializedBy` to the `RelationKind` enum
2. Mapping `RelationSpecializationOf` in `indexableRelation()`
3. Storing the relation in `processRelations()`
4. Handling `ClassTemplateDecl` and `FunctionTemplateDecl` in `findImplementations()`

The `findImplementations` function currently has a chain of `if/else if` checks for `CXXMethodDecl`, `CXXRecordDecl`, `ObjCMethodDecl`, etc. You'd just add cases for template decls:

```cpp
} else if (const auto *CTD = dyn_cast<ClassTemplateDecl>(ND)) {
  IDs.insert(getSymbolID(CTD));
  QueryKind = RelationKind::SpecializedBy;
} else if (const auto *FTD = dyn_cast<FunctionTemplateDecl>(ND)) {
  IDs.insert(getSymbolID(FTD));
  QueryKind = RelationKind::SpecializedBy;
}
```

The index serialization version would need a bump to force re-indexing, but that's a one-liner.

## The make_unique Problem

Another thing that bugs me: "find all references" on `std::make_unique` or `std::make_shared` is often incomplete. The `findReferences()` function already tries to resolve through template patterns using `DeclRelation::TemplatePattern`, but there's a subtlety. When your cursor is on `std::make_unique<Foo>(args...)`, the resolved decl points to the *implicit instantiation*, not the primary template. The `TemplatePattern` flag should resolve back, but the SymbolID that gets queried might not match what the index stored.

The fix would be to explicitly chase the primary template in the references lookup:

```cpp
if (const auto *FD = dyn_cast<FunctionDecl>(D)) {
  if (FD->isTemplateInstantiation()) {
    if (auto *Primary = FD->getPrimaryTemplate())
      if (auto PrimaryID = getSymbolID(Primary))
        IDsToQuery.insert(PrimaryID);
  }
}
```

The index may store references under either the `FunctionDecl` ID or the `FunctionTemplateDecl` ID, so querying both covers the bases.

## Practical Advice (For Now)

Until any of these changes land, here's what actually helps with memory:

| Flag | What it does |
|---|---|
| `--pch-storage=disk` | Store preambles on disk (default, but verify) |
| `--malloc-trim` | Periodic `malloc_trim()` to release freed memory (default on glibc) |
| `--limit-results=N` | Cap completion results, reduces transient allocations |
| `--log=verbose` | See memory-related log messages |

You can also query memory usage from your editor if it supports the `$/memoryUsage` LSP method — VS Code with the clangd extension does. This gives you a hierarchical breakdown of what's eating RAM.

And if you're on a memory-constrained machine and earlyoom keeps winning, the nuclear option is to reduce `MaxRetainedASTs`. There's no command-line flag for this (which is part of the problem), so you'd need a custom build. But going from 3 to 1 cuts idle AST memory by roughly 2/3.

## Wrapping Up

Clangd is genuinely impressive engineering. It's parsing C++ — one of the most complex languages ever designed — in real time, maintaining indexes, resolving templates, and doing it all incrementally. The memory usage isn't a bug so much as a consequence of the problem's inherent complexity.

But it could be smarter about it. The profiling infrastructure is already there. The eviction mechanism is already there. What's missing is the connective tissue: a way to say "this is your budget, live within it." And the template specialization gap is just a case of existing infrastructure (`RelationSpecializationOf`) not being wired up.

These feel like tractable problems. The codebase is well-organized, the abstractions are clean, and most of the pieces are already in place. Someone just needs to connect them.
