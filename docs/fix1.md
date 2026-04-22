# MemoryPack Source Generator — CS8987 `scoped ref` Mismatch Fix

**Issue:** [#378](https://github.com/Cysharp/MemoryPack/issues/378)
**PR:** [#380](https://github.com/Cysharp/MemoryPack/pull/380)
**Commit:** `a94b9eedf52b1a663a58ba514d216e38345cc8d2`
**File:** `src/MemoryPack.Generator/MemoryPackGenerator.Emitter.cs`

## Error

```
CS8987: The 'scoped' modifier of parameter 'value' doesn't match overridden or implemented member.
```

Triggered on any `[MemoryPackable]` type when the project uses C# 11+ language
version but targets a runtime **below .NET 7** (e.g. `net6.0`, `netstandard2.1`,
Unity 6 with a non-.NET 7 backend).

## Why It Happened

### Two independent axes control the generated signature

The source generator resolves two independent facts about the compilation at
analysis time (`MemoryPackGenerator.cs:64–67`):

```csharp
var langVersion = csOptions.LanguageVersion;                            // C# language version
var net7 = csOptions.PreprocessorSymbolNames.Contains("NET7_0_OR_GREATER"); // target framework
```

These are surfaced on `IGeneratorContext` as:

| Property / Method | Source | Meaning |
|---|---|---|
| `IsNet7OrGreater` | preprocessor symbol `NET7_0_OR_GREATER` | TFM is net7+ |
| `IsCSharp11OrGreater()` | `LanguageVersion >= 1100` | compiler speaks C# 11 |

They are **independent**. A project can be on C# 11 (via `<LangVersion>11</LangVersion>`)
while targeting `net6.0` or `netstandard2.1`. This is the exact scenario in
Unity 6 and mixed-target libraries.

### The base class `scoped` is TFM-gated, not language-gated

`MemoryPackFormatter<T>` in `MemoryPack.Core` is compiled per TFM:

```csharp
// IMemoryPackFormatter.cs
public abstract void Serialize<TBufferWriter>(
    ref MemoryPackWriter<TBufferWriter> writer,
    scoped ref T? value)            // 'scoped' present in net7+ build
#if NET7_0_OR_GREATER
    where TBufferWriter : IBufferWriter<byte>;
#else
    where TBufferWriter : class, IBufferWriter<byte>;
#endif
```

The `netstandard2.1` / `net6.0` build of the DLL is compiled without
`NET7_0_OR_GREATER` defined. Whether the C# compiler used to build that DLL
was C# 11 capable is irrelevant — without `NET7_0_OR_GREATER`, the `scoped`
keyword is present in source but the `[ScopedRefAttribute]` metadata may not
be emitted consistently, and in practice the shipped `netstandard2.1` DLL does
**not** encode `scoped` on that parameter.

### The broken code path

Before `a94b9ee`, all three `scopedRef` assignments in the emitter gated only
on the **language version**:

```csharp
// BEFORE — wrong: only checks language version
scopedRef = context.IsCSharp11OrGreater()
    ? "scoped ref"
    : "ref";
```

This caused the generator to emit `scoped ref value` in the generated override
whenever the project's C# version was ≥ 11 — even when the resolved
`MemoryPack.Core.dll` was the `netstandard2.1` build, whose abstract base
method does **not** carry `scoped` in metadata. The generated override then
disagreed with the base, triggering CS8987.

The three affected emit paths were:

| Location | Template |
|---|---|
| `EmitObjectFormatterTemplate` (line 333) | Object formatter `Serialize` |
| `EmitInlineTemplate` (line 983) | Inline (struct/record) `Serialize` |
| `EmitUnionFormatterTemplate` (line 1044) | Union type `Serialize` |

## The Fix

Add `context.IsNet7OrGreater &&` as a prerequisite in all three places:

```csharp
// AFTER — correct: both TFM and language version must qualify
scopedRef = context.IsNet7OrGreater && context.IsCSharp11OrGreater()
    ? "scoped ref"
    : "ref";
```

This makes the generated override signature match the base class DLL being
referenced:

| `IsNet7OrGreater` | `IsCSharp11OrGreater` | Generated | Base (resolved DLL) | Match |
|---|---|---|---|---|
| true | true | `scoped ref` | `scoped ref` | ✓ |
| true | false | `ref` | `scoped ref` | ✗ (edge case, rare) |
| false | true | `ref` ← fixed | `ref` | ✓ |
| false | false | `ref` | `ref` | ✓ |

The edge case `IsNet7OrGreater=true && IsCSharp11OrGreater=false` (targeting
.NET 7+ but C# < 11) is extremely unusual in practice — .NET 7 SDK defaults
to C# 11.

## Secondary Change

`EmitObjectFormatterTemplate` also simplified the `fixedSize` flag from a
manual `if`/assign block to a single expression:

```csharp
// before
var fixedSize = false;
if (Members.All(x => x.Kind is MemberKind.Unmanaged or MemberKind.Enum or MemberKind.UnmanagedNullable or MemberKind.Blank))
{
    fixedSize = true;
}

// after
bool fixedSize = Members.All(x => x.Kind is MemberKind.Unmanaged or MemberKind.Enum or MemberKind.UnmanagedNullable or MemberKind.Blank);
```

Purely cosmetic — no behavioral change.

## Relationship to `fix2.md`

`fix2.md` covers a **different manifestation of the same underlying concept**:
`UnityFormatters.cs` is hand-written source (not generated) that had its own
`#if UNITY_2022_3_OR_NEWER` guard on `scoped` — also the wrong axis. The
generator fix (this commit) and the hand-written formatter fix (`c2d8e21`) are
independent but symmetric: both replace a Unity-version or language-only guard
with the correct `IsNet7OrGreater` + C# 11 conjunction.
