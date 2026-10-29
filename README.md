# interactor-character-domain-c-abi

The **character domain C-ABI** — a pure-Lean reference model for character work,
bridged to the **RISC-V Linux guest** (godot-sandbox / libriscv) through the same
C-ABI shape as the RECTGTN domain (`interactor-rectgtn-domain-c-abi`).

The domain **unions with the RECTGTN GTN planner over GitHub**: it does not copy
the planner kernel and does not reach it through a relative path. `lakefile.lean`
requires the RECTGTN domain by URL, so the two Lean packages compose into one
build and a character goal is an ordinary RECTGTN task node.

## How it runs: the host calls the domain natively

```
pure-Lean character domain (Character/)
  -> Lean-compiled RISC-V guest ELF (character.elf)
  -> godot-sandbox addon binary-translation  -> native C
  -> host calls character_* via sb.vmcall
```

The "compiles to C" step is the **existing green engine** — the same mechanism
the motion / skintokens / rfdetr / rectgtn guests are green on. It is what proves
the character domain runs on the host.

## The union with RECTGTN

`lakefile.lean`:

```lean
require «taskweft» from git
  "https://github.com/V-Sekai-fire/interactor-rectgtn-domain-c-abi" @ "main"
```

`Character/GTN.lean` then imports `Planner.Types` and lifts every character goal
into the shared space:

```lean
def toGTN : CharGoal → RECTGTNNode
  | .cleanRig          => .task "char.cleanRig" []
  | .fitUnderGarment g => .task "char.fitUnderGarment" [g.length]
  | .exportUsd p       => .task "char.exportUsd" [p.length]
```

So a character plan is a RECTGTN plan, and the RECTGTN planner solves it
unchanged. `theorem toGTN_is_task` certifies the lift is total and closed.

## What it is

- A pure-Lean character model (`Character/`) — rig, skin, pose, roll-orientation
  normalization, harness-bone removal — with formal properties.
- A **C-ABI** (`api/character_api.h`) the domain is exposed through, built for
  the RISC-V guest.
- A **guest** (`guest/`) — the RISC-V64 Linux executable the sandbox loads,
  emitting the `.sandbox_variant` section and exporting the `character_*` symbols.

## Two invariants

1. **Pascal strings.** Every string across the C-ABI is length + bytes, exactly
   like Godot's native `String` — never null-terminated.
2. **The GTN is shared.** A character goal is an ordinary RECTGTN task node; the
   character domain defines no plan type of its own.

## The domain, in one page

- `shortestArc bind rest` — the roll orientation normalization: the canonical
  minimal-twist (rank-1 / shortest-arc) rotation that maps a bone's bind
  direction onto its rest direction with **zero roll** about the bone axis. This
  is the completion the pose pipeline requires, and the one LabRCSF's shoulder
  DOF (flexion + abduction, no roll) forces to zero.
- `CleanRig` / `HarnessBones` — the bone-removal criterion: a controller/harness
  bone is exactly one with zero skin weight. `harness_not_skinned` proves the
  removal set is disjoint from the skinned set.

## Build the Lean reference model

```sh
lean            # Lean 4 toolchain (see lean-toolchain)
lake build
lake exe character_witness   # green gate: exits 0, prints "OK:"
```

## Build the guest ELF

```sh
cmake -B build -DCMAKE_TOOLCHAIN_FILE=guest/toolchain.cmake \
  -DCMAKE_BUILD_TYPE=MinSizeRel -DSTRIPPED=ON -DDOUBLE_PRECISION=ON -S guest
cmake --build build
```

## Stage 1: the C-shim guest

The Lean core is not yet compiled to a RISC-V guest — that needs the Lean runtime
port to `riscv64-linux` (RFD 66606-2153 stage 3). Until it lands, the guest
exports **stub** `character_*` bodies so CI can prove the guest ELF shape
(symbol names present, `.sandbox_variant` correct) without the runtime port. When
stage 3 lands, these stubs become the real Lean-compiled RISC-V functions.

## The problem inventory

[`PROBLEMS.md`](PROBLEMS.md) records the inventory of problems this domain is
built against — the first-draft triage of the RECTGTN domain's issue tracker
plus the character-work problems the domain formalizes.

## License

MIT.
