# Inventory of problems

Two inventories: the character-work problems this domain formalizes, and the
first-draft triage of the RECTGTN domain's issue tracker
(`V-Sekai-fire/interactor-rectgtn-domain-c-abi/issues`), which is the sibling the
domain unions with.

## A. Character-work problems (what the domain formalizes)

| # | Problem | Where it lives now | Status |
|---|---------|--------------------|--------|
| C1 | **Controller/harness bones in the rig.** The Elaina rig carries 877 bones; only ~292 are needed (202 skinned + their ancestors). The other 585 are zero-skin-weight controller/reference/pole/IK harness bones that must be removed before export. | `Character.HarnessBones` / `CleanRig`; witness checks the removal set is exactly the zero-weight bones and disjoint from the skinned set. | Formalized; the removal criterion is proved. |
| C2 | **The harness leaks into the USD export.** A single-file `.usdz` must exclude controller harness, `cs_*` gizmos and junk empties, but `bpy.ops.wm.usd_export` ships everything. | `Character.GTN.CharGoal.cleanRig` (a GTN task); the export step is `CharGoal.exportUsd`. | Goal modeled; the USD-side filter is open. |
| C3 | **Grey/shiny exported materials.** The live Blender PBR is correct (skin R=0.45/S=0.2), but the exported USD renders grey-shiny because the imported Principled BSDF inputs are node-linked rather than flat. | — (material graph, not a rig/pose fact) | Open; out of this domain's scope. |
| C4 | **Arms penetrate the coat.** The authored rest pose has arms ~65 deg from vertical and pokes the overcoat; one shoulder is flapped. The fix must rotate arms under the coat without introducing twist. | `Character.shortestArc` (roll orientation normalization); `CharGoal.fitUnderGarment`. | Roll normalization formalized; the target pose is open. |
| C5 | **Arm asymmetry with identity quats.** Reverting FK quats to identity still gave asymmetric arms (79.7 deg left vs 63.7 deg right), so the asymmetry is baked into the loaded armature, not the quats. | `Character.shortestArc` (a symmetric, deterministic completion) | Method available; the baked rest frames are open. |
| C6 | **A-pose vs T-pose for VRM/springbone.** VRM expects a clean A-pose base it converts to T-pose; the coat is sculpted to the rest/armature, so final rotations must match rest bones without poking. | `Character.Pose`, `Character.shortestArc` | Pose type + roll normalization in place. |
| C7 | **Blender edit-mode bone deletion is blocked headlessly.** `bpy.ops.object.select_all.poll() failed, context is incorrect`; `armature.data.bones` is read-only, `arm.mode` is read-only. | — (tooling) | Open; the USD-side removal (C1/C2) is the workaround. |

## B. First-draft triage of the RECTGTN domain issues

Triaged against `docs/rectgtn.md`, `priv/schemas/rectgtn_domain.schema.json`,
`lib/taskweft/compose.ex`, and `Planner/Types.lean`. Each issue carries a
`triage` label and a verdict comment on the tracker.

| # | Title | Type | Verdict | Priority |
|---|-------|------|---------|----------|
| 3 | Branch concurrency in temporal planning | enhancement | **Folded into #8** as its "implement real concurrency" branch; closed on the tracker, not tracked separately. | medium |
| 4 | `goal_methods` in `Taskweft.Compose` | doc/cleanup | **Verified:** absent from the schema (`additionalProperties:false`) yet merged at `compose.ex:52` and listed in both tables. | low |
| 5 | Syntactic sugar `capabilities.entities` | needs-decision | **Verified inconsistency:** sugar exists for capabilities but not for action requirements. Decide keep/unship. | low |
| 6 | Hardcoded ReBAC relations | bug / spec-drift | **Verified:** spec lists 9 relations incl. `CAN_ENTER`/`CAN_INSTANCE`; the Lean `RelationType` has 8 incl. `IS_A`. The sets differ. | high |
| 7 | `TwMultiGoal` permutation backjumping | question | Needs usage data before any keep/unship call. Not a defect. | low |
| 8 | Purely linear "temporal" scheduling | enhancement | **Verified:** durations accumulate linearly; the model already has `after`/`before`/`between`/`within`. Decide unship vs implement. | medium |
| 9 | Composition rules for `capabilities`/`enums` | bug | **Verified, highest impact:** overlays cannot add capability edges, memberships or enums — breaks the documented overlay use case. | high |
| 10 | Tier-2 flow-graph execution engine | enhancement | Valid roadmap item (ADR 0003); not a defect. | low |
| 11 | Replan spec for `fail_step >= 0` | bug / doc | **Verified contract hole:** the soundness contract defines only `fail_step == -1`; the C-ABI implies `>= 0` is re-solved but never says how. | medium |

### Verified defects (fix first)

- **#6** — align the doc's relation list to the `RelationType` enum (or vice
  versa). The core should stay minimal; domain verbs belong to domain
  definitions, since ReBAC already composes relation expressions.
- **#9** — add explicit merge rules: `capabilities.graph.edges` union;
  `capabilities.graph.definitions` merge by key (later wins); `enums` merge by
  key.
- **#11** — ship the explicit `fail_step >= 0` contract.
- **#4** — drop `goal_methods` from `Taskweft.Compose` and both tables.

### Design calls needed

- **#5**, **#8** — decide between unship and implement; each is a valid position.
  #8 carries the concurrency branch folded in from #3, so the one call covers both.
- **#7** — needs usage evidence.

### Roadmap

- **#10** — track against ADR 0003.
