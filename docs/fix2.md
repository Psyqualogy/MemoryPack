# UnityFormatters.cs — CS0460 / CS8987 Fix

**File:** `src/MemoryPack.Unity/Assets/MemoryPack.Unity/Runtime/UnityFormatters.cs`
**Bad commit:** `4bbc527` | **Correct fix:** `c2d8e21`

## Errors

```
UnityFormatters.cs(17,30):  error CS8987  The 'scoped' modifier of parameter 'value' doesn't match overridden or implemented member.
UnityFormatters.cs(18,42):  error CS0460  Constraints for override and explicit interface implementation methods are inherited from the base method, so they cannot be specified directly, except for either a 'class', or a 'struct' constraint.
```

Nine total — CS8987 and CS0460 across the six `Serialize`/`Deserialize` overrides in
`AnimationCurveFormatter`, `GradientFormatter`, `RectOffsetFormatter`.

## Root cause

### CS8987 — `scoped` modifier mismatch

`MemoryPackFormatter<T>` (from `MemoryPack.Core.dll`) declares its abstract
methods with `scoped ref value`. C# 11+ encodes this as
`[System.Runtime.CompilerServices.ScopedRefAttribute]` in compiled metadata.
Any override compiled with C# 11+ must repeat `scoped` on the parameter or
CS8987 fires.

**Key fact about this distribution:** `MemoryPack.Core.dll` is distributed as
a precompiled reference in the Unity package (see `MemoryPack.Unity.asmdef`
`precompiledReferences`). The Cysharp-built DLL encodes `scoped` in metadata
**regardless of TFM** — the `netstandard2.1` artifact was compiled with a
C# 11+ compiler that wrote `[ScopedRefAttribute]` unconditionally. Unity 6
resolves this `netstandard2.1` build and does **not** define
`NET7_0_OR_GREATER` in its compilation context, yet the base method still
carries `scoped` in metadata. An override without `scoped` therefore
fails CS8987 in Unity 6.

### CS0460 — explicit `where` clause on an override

An earlier deployed snapshot (`@22866c07cf6f`) restated `where TBufferWriter : ...`
on the override. Generic constraints on overrides are always inherited from
the base — restating them triggers CS0460.

## Fix history

### Bad fix — `4bbc527`

Guarded `scoped` on `#if UNITY_2022_3_OR_NEWER`. Wrong axis: Unity editor
version has no relationship to which `scoped` metadata the resolved DLL carries.
CS8987 persisted.

### Attempted improvement — `0a93f1b` (`#if NET7_0_OR_GREATER`)

Applying the lesson from PR #380 (`docs/fix1.md`), guarded `scoped` on
`NET7_0_OR_GREATER`. Backfired: Unity 6 does not define that symbol, so the
`#else` (plain `ref`) arm compiled against a DLL that still has `scoped` in
metadata — CS8987 returned on the `#else` arm.

**Why this differed from the generator fix (PR #380):** The generator inspects
`LanguageVersion` and the `NET7_0_OR_GREATER` preprocessor symbol at Roslyn
analysis time to decide what to emit. The generator's `IsNet7OrGreater` tracks
whether the **consumer project** defines that symbol. `UnityFormatters.cs` is
hand-written source that the Unity compiler compiles in the consumer's context
— and Unity 6 genuinely does not define `NET7_0_OR_GREATER` even though it
loads a .NET 8 runtime. The axes that are independent in the generator are
collapsed into a single observable fact for hand-written Unity source: the
precompiled DLL always has `scoped`.

### Correct fix — `c2d8e21`

Unconditional `scoped ref`. No `#if` guards. Matches what the DLL always
encodes, and matches every other formatter in `MemoryPack.Core`
(`ArrayFormatters.cs:38`, etc.).

```csharp
public override void Serialize<TBufferWriter>(ref MemoryPackWriter<TBufferWriter> writer, scoped ref AnimationCurve? value)
public override void Deserialize(ref MemoryPackReader reader, scoped ref AnimationCurve? value)
// same for Gradient, RectOffset
```

No `where` clause (inherited). No `#define` block. Requires C# 11+ — which
Unity 6 with .NET 8 and `LangVersion >= 11` satisfies.

## Summary table

| Approach | Guard | CS8987 Unity 6 | CS8987 Unity 2022.3 (default C# 9) |
|---|---|---|---|
| `4bbc527` | `UNITY_2022_3_OR_NEWER` | yes | yes |
| `0a93f1b` | `NET7_0_OR_GREATER` | yes (`#else` arm) | no |
| `c2d8e21` ✓ | none | no | n/a (C# 9 can't parse `scoped`) |

The C# 9 / Unity 2022.3 default row is not a supported configuration — `scoped`
is a C# 11 keyword and `MemoryPack.Core.dll` encodes it regardless. Any
project on default C# 9 would need to opt into `LangVersion >= 11` to use
this package in Unity 6.
