# AnimationTree: file formats and editor

This document describes the AnimationTree file format and the in-engine graph
editor, so a contributor can **create / edit an AnimationTree** and hand-edit
the files safely.

---

## 1. Where things live

| thing | path |
|---|---|
| tree data files | `Assets/Animations/*.animTree` (+ `*.meta`) |
| runtime tree object | `Engine/Module/AnimationTree/AnimationTree.{h,cpp}` |
| asset wrapper (thin proxy) | `Engine/Module/Asset/AnimationTree/AnimationTreeFile.{h,cpp}` |
| node types (`IAnimationNode` subclasses) | `Engine/Module/AnimationTree/Node/**` |
| transitions / conditions | `Engine/Module/AnimationTree/NodePath/**` |
| controller parameters (`additionParameters_`) | `Libs/LibCore/BlackBoard/{AnimationParameter,IAnimationParameter}.h`, `Libs/LibCore/BlackBoard/Group/ParameterGroup.h` |
| in-engine graph editor | `Engine/Core/Application/Window/Main/Animator/AnimatorWindow.{h,cpp}`, `AnimationTree::OnDrawGraphEditorGui`, `Engine/Module/AnimationTree/Editor/AnimationTreeGraphDelegate.{h,cpp}` (ImGuizmo `GraphEditor` adapter, hosted by `Engine/Module/Gui/Graph/Editor/GraphEditorHost.{h,cpp}`) |

A GameObject/Prefab binds a tree through the `Animator` component's
`FIELD(Asset::AnimationTreeFile) animationTreeFile_`, which stores the tree's
asset GUID.

(Bone-driven transforms — e.g. a hitbox following a hand bone — are not part of
`Animator`. Add a `BoneSync` component next to the `ModelRenderer` and give it a
`TransformSync`; `Animator` v5 dropped the old, never-populated `animationSyncs_` field.)

### In-engine graph editor (Animator window)

Drawn with ImGuizmo's `GraphEditor` (`Libs/ImGuizmo/GraphEditor.{h,cpp}`, locally patched — every
change is marked `NanamiEngine patch`). Nodes are still positioned by each node's saved `position_`, so the
editor changes nothing in the `.animTree` format. `AnimationTreeGraphDelegate` derives from
`Gui::Graph::GraphDelegateBase` (`Engine/Module/Gui/Graph/Editor/`), which it shares with the BehaviourTree
editor (`docs/BehaviourTree.md` §1).

| action | how |
|---|---|
| pan / zoom / fit | middle-drag / wheel / `F` or toolbar *Fit All* (*Fit Selected* for the selection) |
| select | click a node (shows it in the Inspector), left-drag on empty space for a box, Shift to add |
| move | drag selected nodes |
| add a transition | drag from a node's output slot (right) to a clip node's input slot (left); from *Any State* it becomes a `fromAnyStateNodeNodePath`. A clip node takes any number of incoming transitions |
| edit a transition | click the link (arrow shows the direction) — its `AnimationNodePath` opens in the Inspector |
| delete | right-click a link / node, or select it and press `Delete` (Entry / Any State can't be deleted; deleting a clip node also removes its transitions) |
| add a clip node | right-click on empty space → *Add AnimationClipNode* |

The *Running AnimationTree Viewer* shows the same graph read-only (no moving / editing), with the playing
node outlined yellow, the fading-out node purple, a progress bar under each playing clip, and the blending
transition in orange.

---

## 2. `.animTree` format

Output of `cereal::JSONOutputArchive`, hand-rolled in `AnimationTree::OnSave`
(this class does **not** use cereal's own `serialize`/`save`/`load` dispatch —
every top-level key is written by an explicit `archive(cereal::make_nvp(...))`
call). Six top-level keys, in this exact order:

1. `additionParameters_` — `shared_ptr<BlackBoard::ParameterGroup>`, the
   controller's bool/int/float parameters.
2. `entryNode` — `shared_ptr<AnimatorEntryNode>`, the graph's single fixed
   entry point.
3. `visualAnyStateNode` — `shared_ptr<AnimationVisualAnyStateNode>`, the fixed
   "any state" node (a transition sourced from here fires regardless of the
   currently active node).
4. `nodesCount` + `nodes_0..N-1` — `shared_ptr<IAnimationNode>` (polymorphic).
   Only `AnimationClipNode` exists today. **This array's order carries no
   semantic meaning** — the engine holds `nodes_` in a
   `std::unordered_map<Guid, ...>`; hash-bucket iteration order at save time,
   not insertion order, decides it. Don't read anything into it when reviewing
   a diff.
5. `fromNodeNodePathCount` + `fromNodeNodePath_0..N-1` — direct transitions.
6. `fromAnyStateNodeNodePathCount` + `fromAnyStateNodeNodePath_0..N-1` —
   any-state transitions.

On disk: UTF-8, **no BOM**, **CRLF**, no trailing newline, 4-space indent.

### cereal bookkeeping you will see everywhere

| construct | meaning |
|---|---|
| `"polymorphic_id": 0` | null polymorphic pointer |
| `"polymorphic_id": 1073741824` (`0x40000000`) | pointer whose dynamic type == static type (no type name needed) — every `entryNode`/`visualAnyStateNode`/transition slot uses this, since each is always the concrete type its field declares |
| `"polymorphic_id": 0x80000000\|N` + `"polymorphic_name"` | first time type *N* is written; later refs use just `"polymorphic_id": N` |
| `"ptr_wrapper": {"id": 0x80000000\|K, "data": …}` | a `shared_ptr` (K counts every shared_ptr **and** every `Field<T>` inner pointer, in write order, across the *whole file*) |
| `"ptr_wrapper": {"valid": 1, "data": …}` | a `unique_ptr` (`additionConditionGroup_`) |
| `"cereal_class_version": V` | written **once per type per file**, on that type's first instance; omitted afterwards. `BlackBoard::ParameterGroup` itself is the one exception — its `save`/`load` take no version argument, so it never emits this key at all. |
| nested `"value0"` | a serialised base class: `IAnimationNode` (empty stub) for every node, `IObject` for a transition, `IAnimationNodePathAdditionCondition`/`IAnimationParameter` (both empty stubs) for a condition/parameter |

### Node types

| type (`polymorphic_name`) | `CEREAL_CLASS_VERSION` | fixed / addable | fields (save() order) |
|---|---|---|---|
| `NanamiEngine::Module::AnimationTree::AnimatorEntryNode` | 0 | fixed singleton (`entryNode`) | `position_`, `guid_`, `speed_` (vestigial, unused for logic) |
| `NanamiEngine::Module::AnimationTree::AnimationVisualAnyStateNode` | 0 | fixed singleton (`visualAnyStateNode`) | `position_`, `guid_` |
| `NanamiEngine::Module::AnimationTree::AnimationClipNode` | 4 | addable (`nodes_N`) | `animationFile_` (`FIELD(Asset::Mv1File)`), `name_`, `position_`, `guid_`, `speed_`, `blendAnimationOffset_secs_` (v≥1), `modelAnimationIndex_` (v≥2), `clipStartTime_` / `clipEndTime_` / `isLoop_` (v≥3), `nameCheck_` (v≥4) |

Each type keeps its identity guid (`guid_`) and canvas position (`position_`)
among its own named fields rather than in one common base struct.

`blendAnimationOffset_secs_` is how many seconds before a clip's nominal end
an outgoing transition may start blending in
(`GetAnimDuration_secs() = duration_secs_ - blendAnimationOffset_secs_`) — not
an "exit time" in the Unity sense; the only exit-time control is the
transition's `hasExitTime_` flag (below).

**Playback range** (`clipStartTime_` / `clipEndTime_` / `isLoop_`, class version 3). Times are in the clip's
own animation-time units (the same units as `MV1GetAnimTotalTime`, shown as `clipTotalTime`
in the node inspector); `clipEndTime_ <= 0` means "to the end of the clip". The node starts
at `clipStartTime_` every time it is entered, loops back to it on reaching the end, or — with
`isLoop_` off — stops at the end and holds that pose. Defaults (`0` / `0` / loop) reproduce the
pre-version-3 behaviour. Exit-time transitions use the range end.

**Bone matching** (`nameCheck_`, class version 4, default `false`). `MV1AttachAnim` maps the clip's
frames onto the model **by frame index** unless `nameCheck_` is on. A clip exported with a different
frame layout than the model - e.g. a skeleton-only Mixamo export played on a model whose mesh frames
come first (`Brute.mv1`: 10 mesh frames, then `mixamorig:Hips`) - then leaves every bone at the bind
pose (T-pose). Turn `nameCheck_` on for such clips; don't flip it on clips that already play, since
some (e.g. Brute's `Down *.mv1`) only work by index. The editor's Animation View has the same
NameCheck checkbox to try a clip both ways.

Transitions out of a node are only evaluated near the end of its clip
(`AnimationNodePath::TryAddNextCurrentNodePath`: `GetAnimDuration_secs() - transitionDuration < during`).
A non-looping node held at its range end re-evaluates its transitions every frame, but to let a state
change interrupt a node *before* it reaches the end, set its `blendAnimationOffset_secs_` to at least
the range length — the convention the existing player nodes already use (e.g. Idle 309, Walk/Run 1818).
A node left at `0` only reacts once its clip is about to end.

**Mixed class versions.** cereal stores a type's `cereal_class_version` once per archive
(first occurrence), so one `.animTree` cannot hold clip nodes of two versions: every clip
node in a file carries the members of that file's version.

### Transitions (`AnimationNodePath`)

`CEREAL_CLASS_VERSION` 2. Fields, save() order: `additionConditionGroup_`
(`unique_ptr<AnimationNodePathAdditionConditionGroup>`), `transitionDuration_secs_`
(float — crossfade/blend length), `fromNodeGuid_`, `nextNodeGuid_`,
`visualFromNodeGuid_` (v≥1 — the editor-only "drawn from" node; equal to
`fromNodeGuid_` for a directly-authored transition), `hasExitTime_` (v≥2,
default `true`; `false` fires as soon as the condition holds instead of at the
end of the current clip - hit reactions / flinch). **A transition has no
identity guid of its own** — `AnimationNodePath::GetGuid()` is a stub bug that
always returns an empty guid — so a transition can only be identified
positionally (its index in `fromNodeNodePath_N`/`fromAnyStateNodeNodePath_N`)
or by its `(fromNodeGuid_, nextNodeGuid_)` pair.

A direct transition's `fromNodeGuid_` must never equal the AnyState node's
guid, and vice versa — that's exactly what routes it into
`fromNodeNodePath_N` vs. `fromAnyStateNodeNodePath_N`.

### Condition groups (`additionConditionGroup_`)

Hand-rolled `{count, cond1, cond2, ...}` (unnamed `archive(count); for(...) archive(condition)`
calls → `value0` = count, `value1..N` = each condition). An **empty** group
(`value0: 0`, no further keys) means an unconditional/"always" transition.
Each condition (`AnimationNodePathAdditionCondition<bool|int|float>`, fields
`name_` + `equalValue_`) does **equality only** — there is no `<`/`>`/`!=`
operator anywhere in this format. `name_` looks up a parameter by name in
`additionParameters_`.

### `additionParameters_`

`{ptr_wrapper: {id: K, data: {value0: <count>, value1: <param>, value2: <param>, ...}}}`
— note **no `polymorphic_id`** at this level (`ParameterGroup` has no virtual
base) and, per the bookkeeping table above, no `cereal_class_version` either.
Each `value{i}` is a normal polymorphic `shared_ptr<IAnimationParameter>`,
concrete type `AnimationParameter<bool|int|float>` (fields `name_`, `value_`).

### `.meta`

`cereal::JSONOutputArchive` of a `shared_ptr<AssetBase>` = `AnimationTreeFile`
with `contentPath_` (backslash dirs, forward slash before the filename) and
the stable asset `guid_`. Created by `File::OnSave`. **Don't hand-write it.**

---

## 3. Adding a new `IAnimationNode` type

The in-engine "Add node" affordance is a single hardcoded block in the
background context menu of `AnimationTreeGraphDelegate::DrawContextMenus()`
(`Engine/Module/AnimationTree/Editor/AnimationTreeGraphDelegate.cpp`):

```cpp
if (ImGui::MenuItem("Add AnimationClipNode"))
{
    const auto newNode = std::make_shared<AnimationClipNode>(...);
    nodes_[newNode->GetGuid()] = newNode;
}
```

`IAnimationNode`'s virtual interface (`InitForGamePlay`, `OnUpdateBlendRate`,
`OnUpdateAnimation`, `OnExitNode`, `OnUpdated`, `Position`, `SetPosition`,
`GetAnimDuration_secs`, `GraphNodeName`, optional `GraphNodeDetail`) is far
more involved than BehaviourTree's `ActionBase` (effectively just `DoTick`).

### By hand — the anatomy

1. A new `.h`/`.cpp` under `Engine/Module/AnimationTree/Node/<Name>/`:
   `class <Name> final : public IAnimationNode` implementing every pure
   virtual. Parameter members are `[[serialize(0)]]`-tagged, with hand-rolled
   `save`/`load` templates: first `archive(cereal::base_class<IAnimationNode>(this))`,
   then each `CEREAL_NVP(member_)` (gate newer members in `load` behind
   `if (version >= N)`), always including a `guid_` (Guid) and `position_`
   (glm::vec2) member somewhere in the list. After the class, at file scope:
   `CEREAL_CLASS_VERSION`. The `.cpp` includes
   `Engine/Module/Serialization/Engine_Module_SerializationRegistration.h` and ends, at
   file scope, with `NANAMI_REGISTER_TYPE(<fqn>, NanamiEngine::Module::AnimationTree::IAnimationNode);`
   (the engine's wrapper around `CEREAL_REGISTER_TYPE` + `CEREAL_REGISTER_POLYMORPHIC_RELATION`;
   never call the cereal macros directly, and never register in the header).
   Copy `Node/ClipNode/AnimationClipNode.{h,cpp}` as a starting point.
2. `NanamiEngine.vcxproj` (+ `.vcxproj.filters`) — add `<ClCompile>`/
   `<ClInclude>` entries by hand.
3. `AnimationTreeGraphDelegate.cpp` — add a matching
   `ImGui::MenuItem("Add <Name>")` block to the background context menu shown
   above, constructing and inserting the new node the same way
   `AnimationClipNode` does. If the new type needs its own header colour or
   slot count, add a `TemplateKind` for it (`KindOf` / `GetTemplate`); the
   delegate otherwise treats every non-Entry/AnyState node like a clip
   (1 input, 1 output, deletable).

`.h`/`.cpp` in this repo are **Shift-JIS (CP932)** — after any manual
non-ASCII edit, run `Scripts/convDx.ps1 <file>`.

---

## 4. Verifying changes end-to-end

Point a scratch prefab's `Animator.animationTreeFile_` GUID at the tree,
launch the editor, and confirm it loads in the `AnimatorWindow` graph editor
without errors.
