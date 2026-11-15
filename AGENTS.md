# RBXDecomp

This is a matching decompilation of Roblox as it shipped on 2007-12-20, built with MSVC 8.0 SP1 (`cl.exe` 14.00.50727.762). It follows the playbook of the [LEGO Racers decompilation](https://github.com/isledecomp/racers).

## Contributing with an LLM

LLMs are welcome as tools, but a human is always the author. These rules are strict and maintainers will close PRs that break them.

**Human supervision**

- A human opens every PR, reviews every line of it, and can explain every line when asked. "The model wrote it" is not an answer.
- The human runs every check in [Checks](#checks) locally before opening the PR and pastes the WEBSERVICE and ROBLOX numbers, before and after, into the description.
- Say in the PR which parts an LLM produced.
- No autonomous agents opening PRs, no bulk PRs, no drive-by "cleanup" PRs. One PR does one thing.
- If a maintainer asks a question, the human answers it, not a model.

**The quality bar**

Every one of these is a reason to reject a PR.

- **Only real matches.** A `// FUNCTION:` claims 100%. Commit matches you measured on the linked image, nothing that merely looks right.
- **Nothing is sacrificed in the reccmp report.** Compare both targets against `main`. A function that drops, or a `FUNCTION` downgraded to `STUB` to make room for a change, is a bug in your PR, even if the total went up.
- **Grounded, never guessed.** Every name, layout, offset, type and enum value comes from the PDB, the binary or Ghidra. If you can't point at the record, don't write it.
- **Original names first.** Recover real names and hierarchies from the PDB and RTTI. A placeholder that happens to match is unfinished work, not a result.
- **Clean C++.** No decompiler pseudocode, no `*(int*)(this + 4)`, no gotos (one exception, in [Loops](#loops-and-control-flow)), no magic float bit patterns, no casts papering over a wrong type.
- **No unrequested additions.** No new headers, notices, helpers, macros, tools or refactors nobody asked for. A macro is allowed only where the original plausibly had it.
- **No explanatory comments.** Markers, offsets and `vtable+0xNN` comments are required. Prose explaining your reasoning is not; that knowledge belongs in this file.
- **Same behaviour.** Code that runs keeps the original's behaviour exactly.
- **No build-environment tricks.** Don't recreate the original machine's paths, computer name or toolchain quirks, and don't patch binaries after linking, to win bytes. Reproducing the original build environment is a separate, future tool.
- **Leave CI alone** unless a maintainer asks for a change.
- **Commits:** one subject line, 40 to 60 characters, saying what matched or changed. No body, no `Co-Authored-By` or other trailers. "Match X" alone is too vague; "Match Humanoid jump state handlers" is right.

## The two targets

**WEBSERVICE** is `WebService.dll` from RBXGS, the game server: an ATL Server ISAPI extension with 12,633 functions. It goes first because it ships an **unstripped PDB**, and the engine libraries under `Client/` are shared with the client. Three of its six LTCG objects are `Client\win` sources.

**ROBLOX** is `Roblox.exe` 0.3.676.0, the client. It has no PDB but links the same engine libraries byte for byte, so its names come from WEBSERVICE:

- A client body that matches an RBXGS body with relocations masked takes the RBXGS name, and so does every call and data reference reached through matched bodies. `reccmp/roblox-*.csv` and each `ROBLOX` marker under a `WEBSERVICE` marker were generated this way.
- Vtables pair by RTTI class and offset. A client slot whose RBXGS slot holds one unpaired function is that function.
- `roblox-function-size.csv` lists every other function Ghidra finds. Unwind actions and catch handlers are filtered out by reading each `__CxxFrameHandler3` FuncInfo, and adjustor and import thunks are filtered by their bytes. The same filter keeps 12,609 of the 12,633 RBXGS procedures, so the two denominators are comparable.
- Third-party code is typed `library`: anything whose nearest RTTI-owned vtable slots belong to `CXT`, `XTP` or `Scintilla::` classes, anything Ghidra's function ID names from MFC or ATL (except `CreateObject`, which `IMPLEMENT_DYNCREATE` expands in Roblox's own code), and everything from the first `GLCaps` body at 0x4ea5a0 to App's first object at 0x5640f0, which is G3D. The exception inside that range is 0x50b280 up to the CPUID probe at 0x537910: that's the client's rbxview and renderlib (RTTI spells them `RBX::ViewNew` and `RBX::RenderNew`).
- A `ROBLOX` marker says `FUNCTION` only where the client comparison reaches 100, and `STUB` otherwise.
- Quote the implemented count over **every** client function, never the accuracy over the transferred subset.

How our `Roblox.exe` is built:

- `Client/win` compiles the way the client's objects did: `/O2`, stack cookies, no `/GL`, and a precompiled `StdAfx.h`.
- `mfc80.lib` links ahead of msvcrt, so `operator new` and `delete` import from `MFC80.DLL` by ordinal (762 and 764), as the original does.
- `/FIXED:NO`, because the original carries `.reloc`.
- `Client/win/Roblox.idl` compiles under MIDL 6.00.0366 to the shipped type library except for MIDL's build stamp, so the resource embeds `res/Roblox.tlb` and the build only takes `Roblox_h.h` and `Roblox_i.c`. A typedef only reaches the library when it's `[public]`.

CI fetches both original binaries from archive.org.

## Building

```
<path-to-msvc8>\Common7\Tools\vsvars32.bat
cmake -B build -G "NMake Makefiles" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build
```

Use the portable [MSVC800-SP1](https://github.com/Asphaltian/MSVC800-SP1). Call `vsvars32.bat`; `VC\bin\vcvars32.bat` needs a registered install. RTM (`.42`) doesn't reproduce the original's codegen. CMake refuses anything but cl 14.x.

> **Note:** after anything that moves an include, delete `build/` before you trust a green build. The object that breaks is the one that didn't need recompiling.

## Checks

CI runs seven checks. Run all of them before you commit, not just the score.

```bash
reccmp-reccmp   --target WEBSERVICE --nolib                       # score
reccmp-reccmp   --target WEBSERVICE --nolib --verbose 0xADDRESS   # one function
reccmp-reccmp   --target WEBSERVICE --nolib --json out.json       # what's left
reccmp-decomplint WebService Client --module WEBSERVICE --warnfail
reccmp-vtable   --target WEBSERVICE
reccmp-datacmp  --target WEBSERVICE
python tools/check_folded.py --target WEBSERVICE
python tools/reccmp_addr_padding.py
```

CI also scores `--target ROBLOX` and diffs both targets against `main`. One more check runs in its own job:

- **ncc** (`tools/ncc/ncc.py`) needs real LLVM's `libclang.dll` and the `clang==21.*` bindings. It exits 0 either way; the finding count is the verdict. A recovered name that trips the style is a style-file change, never a rename.

What each one is actually telling you:

- **decomplint** runs with `--warnfail`, so a warning fails CI. Vendored trees are excluded by path, which means a bad marker there fails silently. Point it at `Client/Network/raknet30` by hand after touching RakNet.
- **vtable** checks every vtable named on both sides. A `// VTABLE:` marker asserts the whole layout, so don't add one until every slot can be claimed.
- **datacmp** matches statics by bare name, so a name has to be unique in both builds unless its `webservice-statics.csv` row names the object.
- **check_folded** confirms every annotation on a shared address landed on one recompiled address.

Reading reccmp output:

- `--json` reports an effective match with its raw ratio plus an `effective` flag. Filter on the flag, or finished functions show up as outstanding.
- reccmp prints one row per recompiled address. If our link folded two functions the original kept apart, one is simply absent. Probe it with `--verbose` before calling it a miss.
- `--verbose` compares raw `[esp+N]`, so one missing local shifts the whole body by a constant. Add the frame difference back before reading a diff, and read the original-only lines for callee names first.

## Ground truth

`WebService.pdb` is not stripped. It has symbol names, namespaces, class hierarchies, source paths, line numbers, per-object compiler command lines and the toolchain version. Check it before you work anything out from disassembly.

Both files come from the RBXGS installer on archive.org, extracted with [lessmsi](https://github.com/activescott/lessmsi). The README has the URL and hashes; `.github/workflows/robloxbin.yml` does the same fetch.

### Ghidra

Load the PDB into Ghidra before writing anything. Then push the recorded local and parameter names into the project once, because Ghidra's PDB loader only takes function names. Re-deriving names per function is a waste of everyone's time.

- MSVC records `S_BPREL32` against its own frame base, which doesn't agree with Ghidra's offsets. Work out the delta per function by voting: count how many recorded offsets land on slots Ghidra already has and take the winner. Guessing it renames real slots wrongly and silently.
- `reccmp-ghidra-import` pushes our matches the other way. It runs Ghidra headless, so close the project in the GUI first.
- Several hundred `IllegalArgumentException` warnings from the line importer are expected; they come from ICF folding.

### Querying the PDB

Load every cvdump section into one SQLite database and query that. Real questions span sections: an address is in SYMBOLS, its lines in LINES, its object in SECTION CONTRIBUTIONS, that object's flags in MODULES, its parameter types in TYPES, and its constness in the `LF_MODIFIER` its `This type` points at. Extract our own `build/WebService.pdb` the same way and diff the two for a work list.

Parser traps that each lose data silently:

- The contribution table's `Imod` is hex.
- 61,440 type ids have four digits, not eight. They're referenced upper case and defined lower case.
- A basic-typed argument appears as `T_INT4(0074)`, not `0x....`.
- Overloads appear as `LF_METHOD` pointing at an `LF_METHODLIST`, not `LF_ONEMETHOD`.

Stream 4 is `/LinkInfo`, so there's no `LF_UDT_SRC_LINE`; a class's header has to come from LINES. Thirteen `/src/files/*...webservice.inj` streams hold the attribute provider's injected source.

### What each record proves

- **`reccmp-cvdump -m`**: each module's `src =` path with real casing. LINES lowercases everything, so this is the only casing source, and only for `.cpp` files.
- **TYPES (`-t`)**: member names, offsets, class sizes, enum values, typedefs. Prefer it when Ghidra disagrees.
- **Method constness**: resolve the `This type` pointer to either the class or an `LF_MODIFIER const`. It costs no instruction but changes what alias analysis may assume.
- **LINES**: which file an address came from. An inlined expansion is attributed to the call site, not the callee. MSVC 8 has no `S_INLINESITE`.
- **`-m -seccontrib`**: which object contributed an address. It's the only way to place a namespace-scope global.
- **`S_BPREL32`**: every stack local and its offset. Locals are laid out by size, so declaration order can't be read back. Shared offsets between non-overlapping lifetimes are normal.
- **Parameters**: positive-offset entries in offset order are the argument list. When renaming to them, leave alone: slots with several names, `__cdecl` variadic members (where `this` is a stack parameter), swapped pairs, and initialisers or member accesses already spelled right.
- **A parameter named like a member** means the original wrote `this->`. Renaming otherwise turns `member = value` into a self-assignment that still compiles.
- **Where a class is declared** follows from where its in-class inline functions are attributed.
- **`S_FRAMEPROC`**: `inl_specified` (the original wrote `inline`), `wasinlined` (it was ever expanded), and `inlasm` (it reached inline assembly; 93 functions).
- **`S_GPROC32` vs `S_LPROC32`**: a `S_GPROC32` free function was not `static`, and that reaches codegen.
- **The raw string table**: 2,001 lowercased source paths. It's the authority on whether a header existed.
- **Jump tables** settle enum values.

Prove layouts with `DECOMP_SIZE_ASSERT(T, size)`. Assert a wrong size once to make sure the assert is live.

## Annotations

Functions in a file go in ascending address order, and namerefs follow the same order. A marker placed above a lower address silently drops every annotation after it.

```cpp
// FUNCTION: WEBSERVICE 0x100a77d0         complete, compared by reccmp
// FUNCTION: WEBSERVICE 0x100a77d0 FOLDED  identical code the linker merged
// TEMPLATE: WEBSERVICE 0x10103660         template body, nameref on the next line
// STUB: WEBSERVICE 0x100a7800             incomplete, skipped by reccmp
// LIBRARY: WEBSERVICE 0x10001000          CRT or third party, in library_msvc.h inside #ifdef 0
// SYNTHETIC: WEBSERVICE 0x10007040        compiler-generated, like scalar deleting destructors
// GLOBAL: WEBSERVICE 0x102c7000           global variable
// VTABLE: WEBSERVICE 0x10056440           virtual function table
// SIZE 0xf8                               class size
```

- **The colon is required** on everything except `// SIZE`. reccmp silently ignores `// VTABLE WEBSERVICE 0x...`.
- **`FUNCTION` and `SYNTHETIC` mean 100%.** Anything less is `STUB`, or no nameref at all.
- **`FOLDED`** siblings share an address, are exempt from ordering, and don't need `STUB()`.
- **Two bases** need the base-qualified marker, `// VTABLE: WEBSERVICE 0xADDR Base`, spelled exactly as the recompiled PDB does: `RBX::Notifier<RBX::World, struct RBX::AutoDestroy>`. The second base's adjustor thunk goes in `webservice-synthetic.csv`.
- **Namerefs go in headers**, never in a `.cpp`.
- **No access label between a marker and its function.** Put `private:` above the marker.
- **Long namerefs** go inside `// clang-format off`, or the reflow truncates them.
- **A one-line `{}` body** confuses decomplint's parser. Split the braces.
- **A nameref can't carry a parameter list.** On an overload set it lands on whichever overload reccmp indexes first; pair the intended one with a `synthetic` row on its mangled symbol.
- **On a folded address**, use the alias reccmp's recompiled entity holds, and it must be one of the original's aliases. The held alias shifts when a new instantiation joins the fold.
- **Stubs go stale.** Flip every `STUB` in a directory to `FUNCTION`, score once, and revert the misses. It costs one build and finds every finished body.

## Classes

```cpp
// VTABLE: WEBSERVICE 0x10056440
// SIZE 0xf8
class Instance : public GuidItem<Instance> {
public:
	virtual ~Instance();      // vtable+0x00
	virtual void VTable0x04();

	// SYNTHETIC: WEBSERVICE 0x100078f0
	// RBX::Instance::`scalar deleting destructor'

private:
	int worldIndex;                       // 0x28
	undefined m_unk0x2c[0x40 - 0x2c];     // 0x2c
};
```

- `DECOMP_SIZE_ASSERT(Instance, 0xf8)` goes in the source. Offset and `vtable+0xNN` comments are required. Size gaps use a subtraction.
- Members and virtual slots follow declaration order, whatever the access section.
- Under `_SECURE_SCL`, `std::set` is `_Myproxy, _Myhead, _Mysize` and `std::vector` is `_Myproxy` then three pointers.
- Starting a new class? Match the constructor, destructor and scalar deleting destructor first. Together they prove size, bases, vtable and member order.

## Types and names

- `undefined`, `undefined2` and `undefined4` from `decomp.h` stand in for unproven types. A type is proven when a 100% `FUNCTION` uses it.
- The PDB usually has the real name. Where it doesn't, use NCC: `FUN_XXXXXXXX`, `g_unk0xXXXXXXXX`, `m_unk0xXX`, `p_unk0xXX`, `VTable0xXX`, `c_` for enum constants. `ncc.style` also allows `prop_`, `event_`, `signal_` and all-caps, the original's own conventions.
- Rename a placeholder only on unambiguous evidence. A misleading name is worse than a neutral one.

## Code style

- `if (flags & c_flag)`, never `!= 0`.
- No redundant `this->`.
- Un-suffixed Win32 macros (`CreateWindowEx`), as the source had.
- `NULL` for pointers, `TRUE`/`FALSE` for `BOOL`, plain `0` for scalars.
- No leading `const` on return-by-value.
- clang-format what you touch. `IncludeBlocks: Regroup` sorts across blank lines, so only `// clang-format off` holds an include in place.

## Matching a function

1. **Check the byte budget.** The gap to the next address bounds the body. If your logic can't fit, it's a wrapper and the work is in a callee.
2. **Read the line records.** Count the statements, note lines with no code, check `S_BPREL32` for the locals. Getting the statement count right beats any respelling.
3. **Look for a helper in TYPES** before writing arithmetic out. Hand-inlining a helper the original called is the most expensive mistake there is.
4. **Take the calling convention from the definition.** `__thiscall` is a member, or the global is an instance. A member ending in a bare `ret` is `__cdecl`, so `static`.
5. **Stub unknown callees** with `STUB(0xADDRESS)` in the body, in address order, so ICF can't fold them. Empty destructors are exempt.
6. **If a caller needs the real callee, write the callee.** App builds `/Ob2`, so a one-store stub inlines and changes the caller's frame. There's no `noinline` shortcut.
7. **Write clean C++.**
8. **Build, run `--verbose`, iterate to 100%.**
9. **Compare the vtable address too.** It exposes missing or misordered virtuals.
10. **Re-verify everything that touches a class you changed**, then run every check.

Principles that keep you honest:

- A diff means the code is wrong: layout, types, a missing base, an inlining decision. Register allocation is never the explanation.
- Fix the callee before the caller.
- Re-measure anything inherited or generated. A claim isn't a verified match.
- Casts plus pointer arithmetic mean the types are wrong.
- One root type per header; nest single-owner helpers in their owner.
- An unexplained address gap means a function sits in the wrong class. Move whole classes, never split one across files.
- Measure on the linked image, not an `/FA` listing. A listing's mnemonic stream is still fine for cheap sweeps.

## MSVC 8 codegen

Each of these cost someone a round trip. Keep the source clean and the reasoning here.

### Floats and x87

- `a > b` emits `test ah,5 / jp`; `!(a <= b)` emits `test ah,1 / je`. Not the same compare.
- A memory-to-memory compare loads the right operand into st(0) either way. Read operand order off the load and direction off the flag test (`>`: `test ah,5`; `<`: `test ah,0x41`).
- A literal on the right is loaded (`fldz`, `fcom st(1)`); on the left it stays a memory operand (`fcom [__real@...]`).
- A compare whose loaded side is also stored uses `fcom`, not `fcomp`.
- MSVC reorders `&&` chains of float compares. Write them in the order the binary tests.
- `member != value` keeps the member in memory (`fcom`); the reverse needs `fucom`.
- `std::swap(a, b)` loads `a` first. `std::min`/`max` return references (`lea` of both); suppress `windows.h`'s `min`.
- Locals are pushed in declaration order. A wrong order leaves the x87 stack off by one and shifts every later `st(n)`.
- `x * -1.0f` keeps the constant live in a register (`fmul`); `-x` emits `fchs` plus an `fxch`.
- A scalar on the left of a vector binds G3D's free `operator*`, which schedules differently from the member.
- `RBX::Math::sign` returns `float`; `G3D::sign` returns `double`.
- Float to int goes through `G3D::iRound` (`fld`/`fistp`). A C cast calls `__ftol2_sse` and never matches. Feed it a `float` local.
- A literal one ULP off looks like a naming problem. Read the constant out of the image.

### Statement shape and evaluation order

- Arguments evaluate right to left, operators included. Naming each argument in a local restores left to right.
- A comparison loads its right operand into the register. `f() != g()` calls `g()` first; assigning `f()` first flips that.
- Operand order decides evaluation order even when the meaning is commutative.
- A constructor return evaluates every argument before storing; member-by-member assignment interleaves.
- Vector expressions batch loads then drain x, y, z; component assignments store as they go. `Vector3 d = v - t;` then per-component assignments gets both.
- `a = b = c = v` stores right to left from one register.
- Blocks land where their statements sit in the source. Move a rare `return true` below the hot path.
- `if (c) return x; return y;` makes `x` the fall-through value. `x = a; if (c) x = b;` and `x = c ? b : a;` differ by an instruction.
- A ternary materialises into a 32-bit register (`movzx eax, al` or `mov eax, 1`); a plain `return false` is `xor al, al`.
- `return p ? true : false` is not `return p != NULL`.
- Comparing two `bool`s is a byte compare; negate into a `bool` local first.
- Multiply operand order is normalised; parenthesisation isn't.
- Two statements on one line are one line record.
- A base conversion evaluated before the null test guarding it is a helper's argument (`startListening`/`stopListening` in `Events.h`). One computed before two member compares is a constructor argument.
- Naming a call's argument moves the object load below the arithmetic.
- An object reached through an inline wrapper is evaluated before the arguments; reached directly, after.
- `f(x)` through a converting constructor builds the temporary in the argument slot; `f(T(x))` copies it.

### Naming locals

Lines with no code in the line records mean the original named something. Naming it back is worth more than any respelling.

- A pure permutation of `ebx`, `ebp`, `esi`, `edi` means a local is missing. Add the name.
- Which constant a callee-saved register holds counts that literal's uses. A mismatch means a wrong argument somewhere.
- Return slot filled directly means the value was returned unnamed; built in registers and copied out means a named local. A second `return` of a different object disables NRVO.
- A returned aggregate read through `eax` was never named.
- Naming a temporary passed by const reference changes its x87 drain.
- A leading underscore on a parameter means the body binds the unprefixed name.
- Alias only what the type records give an accessor for. Aliases leave no `S_BPREL32` record.
- Declaration order of two symmetric values decides which callee-saved register each gets. Swap declarations, not statements.
- A constructor's stores landing inside a later call's line record means its declaration came first.
- Where a declaration sits decides when its value loads.
- The frame size bounds which locals can exist.
- `add reg, offset` before a loop is strength reduction, not a source alias.

### Inlining

- Check `wasinlined` before assuming either way.
- An empty callee must be defined out of line to keep its call.
- If the original calls and ours inlines, our callee body is too small. Write the real body.
- Which sites inline a large function is cost-driven, not spelled. Source order and attributes aren't levers.
- Two expansions of one inline in one function can schedule differently. A residual in just one of them is scheduling.
- An inline emitted only because its address was taken can't be reproduced until that reference exists.
- A stub's register footprint reaches its caller. Give stubs the calls their real body starts with.
- A condition spelled through an inline helper spends inline budget differently from the same tests written out.
- `!fuzzyEq(a, b)` and `fuzzyNe(a, b)` don't inline alike.
- A stub ending `return otherStub();` folds onto it.
- A virtual destructor with no standalone PDB function was inlined everywhere: define it in the header, no annotation.
- A base destructor kept from an unrelated object is inline too.
- Inlined setter store order isn't recoverable. The lever is in the caller.
- Tail recursion becomes a loop at the definition, but callers inline exactly one level of the recursion. A loop body appearing twice with the same line numbers is that.
- Score a vendored inline's own COMDAT copy before blaming its spelling.
- If two spellings each fix half a residual, neither is the cause. Fix the statement that fills the x87 stack.

### Loops and control flow

- A dense switch lowers to descending `sub eax, N / je`; an if chain emits repeated `cmp`.
- A jump table that keeps default-target entries had a `return` in those cases, not a `break`.
- A `switch` case isn't a scope for temporary reuse. `case NORM_X: goto x;` over labelled statements is the **one sanctioned goto**.
- The comma operator keeps every temporary alive at once.
- A `for` increment is attributed to the last body statement.
- Branches leaving the x87 stack at different depths can't merge, so the next statement appears twice.
- `do { } while (0)` with `break` hoists every callee-saved push to the preheader.
- `std::for_each` uses `_Unchecked` iterators; a check-free node walk is `for_each` plus `mem_fun`.
- `cmp a / jne / body / cmp b / je body` was two `if`s with identical bodies, not `if (a || b)`.
- Identical early exits are one shared return.
- A missing line record on a `for` line means the loop was fully unrolled.
- A catch funclet returning somewhere other than the fall-through jumped out of the handler.
- The same test on two lines is an `if` wrapping a `while`.

### Class shape

- Abstract bases are `__declspec(novtable)`.
- `ComputeProp` size records inheritance shape (16 vs 24 bytes).
- A gap between the last member and the recorded size is a vtordisp, not padding.
- A derived class starts at the base's non-virtual size.
- `Math` and `Units` are classes of statics, not namespaces.
- A `static const float` without a guard was folded; a guard means the initializer was a call.
- Write float literals at each use, not as namespace-scope `const float`s.
- Which object emits a scalar deleting destructor decides whether it inlines the destructor.
- A direct call to a virtual method means the original qualified it.
- boost's `shared_ptr::operator<` compares ownership (offset 4).
- `__int64` multiply is `_allmul`; the conversion back tells you signedness.
- Read CRT call names from the publics, never from what the code looks like.

### Templates and folding

- Most of what's left is template instantiations. Count by owning template; one template done right settles all its instantiations.
- An instantiation belongs to the object the linker kept it from, flags included. One kept from a `/GL` object is LTCG.
- The instantiation set reveals the spelling (`vector<bool>::back` out of line vs `*(end() - 1)`).
- Explicit instantiations carry no line records: `// TEMPLATE:` plus a nameref on the body. They don't reach base classes.
- A template constructor is user-declared; add the default constructor back.
- Don't name a non-type template parameter like a visible member.
- Template member namespaces sit after the arguments in the mangling.
- A COMDAT one of the six `/GL` objects defines resolves to the LTCG copy and never folds with app copies.
- `/OPT:ICF` folds heavily. Run `check_folded.py` before calling a shared-address function unmatched. A wrong constant in a three-byte body still scores 100 against the alias it pairs with.

## reccmp data files

reccmp only parses the recompiled PDB, so the original's names come from CSVs in `reccmp/`. One file per library, rows only, no comments. reccmp ignores unknown columns (that's what `x-text` is for).

| File | What it holds |
| --- | --- |
| `webservice-function-size.csv` | The original's 12,633 function extents. Everything else edits this. |
| `webservice-{atl,boost,crt,g3d,ijg,libpng,stl,zlib}.csv` | Third-party code, typed `library`. |
| `webservice-symbols.csv` | Roblox functions not annotated yet, untyped. Regenerated, never edited. |
| `webservice-floats.csv` | The 394 `__real@` constants. Loads before statics. |
| `webservice-globals.csv` | Globals, class statics and vftables by mangled symbol. |
| `webservice-statics.csv` | Statics by bare name; a `module` cell pins the object. |
| `webservice-strings.csv` | 3,569 literals; `??_C@_1` is wide. |
| `webservice-synthetic.csv` | Adjustor thunks and compiler-generated bodies. `synthetic` asserts a match; `stub` pairs without scoring. |
| `roblox-client.csv` | Client-only rows. Every other `roblox-` file is generated and overwritten. |
| `roblox-xtp.csv` | Codejock functions matched against our own `ToolkitPro1121vc80DS.lib`. |

Rules that bite:

- **`library` means third-party code, nothing else.** Typing Roblox code `library` hides it from the score and hides wrong claims behind it.
- **A `symbol` or `name` pairs.** You can't label without matching, so naming files only hold originals with no counterpart.
- Mangled symbols go in `symbol`, bare names in `name`. Mix them up and rows vanish silently.
- Addresses are section-relative. Check a `__real@`'s bytes against its name.
- A blank type is skipped, not applied. It can't clear an earlier `library`.
- Load order is file order, then row order. An override loads after what it overrides.
- Rows on folded addresses carry the alias the recompiled entity holds, and new instantiations shift it. The symptom is `call <OFFSET>` on the original's side.
- Labels with commas must be quoted.
- Third-party vftables and unsized library rows go in one at a time (`VENDORED_VTABLES`, `UNSIZED_KEEP`).
- Two overloads sharing a 255-character prefix pair in address order. Blank the wrong row's symbol and give its address a bare-name statics row.
- Dynamic initializers and atexit destructors pair by name: `// SYNTHETIC:` in the owning header, or `// LIBRARY:` in `library_msvc.h`.
- A file static an initializer stores through needs its own row first.
- Where our link folds two functions the original kept apart, only one can ever be claimed.

reccmp is the [`Asphaltian/reccmp@msvc800`](https://github.com/Asphaltian/reccmp) fork, pinned in `tools/requirements.txt`. A module isn't a file: claim a file complete only by scoring every annotation in it.

## Build layout

Each original static library is its own OBJECT library, because each had its own flags. `reccmp-cvdump -s` prints every object's command block, so read the flags, don't infer them.

- App, RbxView and AppDraw are `/GS-`; Network and RenderLib aren't.
- App alone has `/fp:fast` and `/Ob2 /Oi /Oy`. It also carries `/Gy` here (the original didn't); keep it or `check_folded.py` breaks.
- RenderLib alone has `/Wp64`.
- Six translation units are `/GL` and the link is `/LTCG`. `/GL` is scoped to the `webservice` target, because `Client/win` also builds into the client without it. Prove a `/GL` function against an `/FA` listing built without `/GL`.
- The client links its own objects, then rbxview, renderlib, network, glg3d, graphics3d, app, appdraw and Lua. App's objects link in `OPENRBX_APP_CLIENT_SOURCES` order, recovered from the client. Any reorder moves held aliases, so re-key the affected rows and namerefs.
- RBXGS linked real static libraries (appdraw, app, graphics3d, renderlib, rbxview, network, glg3d). OBJECT libraries can't reproduce that pull order exactly; switching would cost unreferenced objects and re-keyed aliases.
- `util/decomp.cpp` gives the DLL a source file. It's ours; keep it out of the LTCG list.
- `OPENRBX_KEEP_UNREFERENCED` links `/OPT:NOREF` so reccmp sees uncalled functions. The original used `/OPT:REF`.

App uses subdirectories (`util`, `v8world`, `v8tree`, `v8kernel`, `v8datamodel`, `v8xml`, `gui`, `humanoid`, `reflection`, `script`, `security`, `tool`) with headers under `include/`. AppDraw, RbxView, RenderLib and WebService are flat with headers under `include/<libname>/`.

## The DLL

- `WebService/WebService.cpp` is the ATL Server ISAPI surface: `_AtlModule` is a `CDllMainOverride`, `theExtension` a `CRbxIsapiExtension`. It needs `_WIN32_WINNT 0x0400` and `_WIN32_DCOM`.
- `_ATL_NO_COM_SUPPORT` is a `target_compile_definitions` entry, never a `#define` in one file.
- `WebService.def` names the eight exports the linker can't infer.
- `WebService.h` is the service's type system from the type records. `WebServiceMaps.cpp` is the `_soapmap` tables, generated from `.rdata` (`_soapmap` is fifteen dwords, `_soapmapentry` fourteen).
- Attributed ATL doesn't work in MSVC800-SP1 (provider attributes fail with C2337), so the maps stay generated.
- Smoke test by loading it from a 32-bit host: it maps at 0x10000000, resolves every export, and `GetExtensionVersion` answers 6.0 and `"ROBLOX Web Service"`.

## Reflection and registration

Every registered class hangs off `X` → `DescribedCreatable<X, Base, sX>` → `Described<X, sX, FactoryProduct<X, Base, sX>>` → `FactoryProduct<X, Base, sX>` → `Base`. None of the templates adds a byte. They live in `include/util/object.h`, `include/reflection/reflection.h` and `include/v8tree/Instance.h`, and each class emits its copies from its own `.cpp`.

- Wire a class from the type records and contribution table. A class whose base isn't declared yet can't be wired.
- `Described::classDescriptor`'s local static is named `foo` in all 98 instantiations.
- `TypedPropertyDescriptor<T>::GetSet` has no virtual destructor; its first slot is `isReadOnly`.
- `PropDescriptor` builds its `GetSet` in a static `getset` helper.
- The chain's namerefs live in one block per header. Only write one where the address is at 100%.
- `NonFactoryProduct` and `FactoryProduct` both declare `getClassName`; the nameref replaces the first only.
- Move placeholder members to the base that owns them before rebasing.
- Five of a `Notifier`'s six events are `struct`.
- `FactoryProduct` and `DescribedCreatable` have protected virtual destructors in the records. `DescribedCreatable` doesn't declare its own.
- `Creatable<T>::operator new`/`delete` are protected `malloc`/`free`.
- A `ClassDescriptor` static's atexit destructor is a real per-instantiation function.
- Property descriptors are `const` statics (mangling ends in `B`).

## Third party

Every dependency sits where the original tree had it, recorded in each object's `cwd`, `src` and `-I` arguments. Only `cwd` gives on-disk casing.

| Path | What |
| --- | --- |
| `Client/boost_1_34_1` | boost; headers under `src/boost` |
| `Client/App/lua-5.1.1` | Lua 5.1.1, compiled into app.lib as 25 units |
| `Client/Network/raknet30` | RakNet 3.0, reconstructed (never archived) |
| `Client/fmod/api` | FMOD Ex 4.8.8 |
| `Client/Rendering/SDL-1.2.6` | SDL |
| `Client/Rendering/Mesa-6.5` | Mesa, the software GL fallback |
| `Client/Rendering/OgreSDK/include` | Ogre 1.4.4 headers the client compiles against |
| `Client/Rendering/g3d` | G3D 6.09's `source` folder, with zlib, libpng and IJG jpeg inside |
| `Client/win` (Scintilla*) | Naughter's `CScintillaCtrl`, the 2006-04-30 release |
| `3rdparty/Microsoft DirectX SDK (November 2007)` | `dinput.h` and its import libraries |
| `3rdparty/Microsoft DirectX 9.0 SDK (October 2004)` | headers and the static `d3dx9.lib` for `rgdx` |
| `3rdparty/Codejock Software/...` | Xtreme ToolkitPro 11.2.1, reconstructed from 11.2.2 |
| `3rdparty/scintilla` | Scintilla 1.71 |
| `3rdparty/ogrenew` | Ogre's v1-4 branch at SVN r7025 (1.4.4 plus seven later files), cut down to what its four DLLs build from |

Vendored code builds with the flags it was built with. G3D's and the engine's come from the command blocks. boost uses its own 1.34.1 `release` variant for msvc-8.0. Codejock builds the Dynamic Release configuration of its `ToolkitProStatic_vc80.vcproj`. SciLexer and the four Ogre DLLs (`rgmain`, `rgpar`, `rggl`, `rgdx`) take the Release settings of `SciLexer.dsp` and Ogre's `*_vc8.vcproj`, and the client links the `rgmain` built here.

Details worth knowing:

- Roblox wrapped `CScintillaCtrl`, Scintilla's `Platform.h` and `Scintilla.h` in a `Scintilla` namespace. `roblox-scintilla.csv` names the 95 single-candidate matches.
- Codejock 11.2.1 is 11.2.2 with each release-note item reverted where the client confirms it. Every `> _XTP_SCHEMA_1121` check marks one. Its `.cpp` files build with `ZLIB_WINAPI`, because the client holds one stdcall zlib, G3D's.
- `RakPeer`'s vtable marker needs a `// class RakPeer` line under it, because `class RAK_DLL_EXPORT RakPeer` confuses the class regex. The original spells many RakNet `char*` parameters as `const char*`.
- CI lints none of the vendored trees.

## Gotchas

- **Comment-only edits that add lines need a rebuild before scoring.** Markers resolve through line records, so a nameref inserted mid-header moves every body below it.
- **`reccmp-reccmp --verbose` crashes printing a match** on a cp1252 console. Set `PYTHONIOENCODING=utf-8`.
- **Never sweep timestamps in a clone.** `3rdparty/**` and the vendored trees are `-text`, so touching mtimes and then `git add -A` rewrites their line endings. The CI backdate step is fine only on a throwaway runner.
- **MSVC 8's `mt.exe` can crash** on newer Windows. Delete the half-linked output and rebuild, or the next build treats it as up to date.
- **Data past `.data`'s raw bytes is fine**; reccmp zero-fills it.

## Project structure

```
Client/App/                  # app.lib, subdirectories by subsystem, headers in include/
Client/Network/              # network.lib
Client/RbxView/              # rbxview.lib
Client/RbxGraphics/          # Headers only; compiled into AppDraw and RenderLib
Client/Rendering/AppDraw/    # appdraw.lib
Client/Rendering/RenderLib/  # renderlib.lib
Client/win/                  # The client; ErrorUploader, LogManager and VersionInfo also build into WebService
WebService/                  # The DLL, plus library_msvc.h and WebServiceMaps.cpp
util/                        # decomp.h, decomp.cpp
3rdparty/                    # SDKs and libraries built against from outside the tree
reccmp/                      # The CSVs reccmp reads for the original's side
cmake/                       # reccmp CMake integration
tools/                       # check_folded.py, reccmp_addr_padding.py, ncc
```

Directory names come from each library's `-Fo` and `-I` arguments. The original also had `Client/Rendering/RenderLibNew/` on the include path, but nothing in either binary has been traced to it yet.
