# BehaviourTree: file formats, actions, and editor

This document describes how to **create / edit a behaviour tree** and **add a
new behaviour action** — for either of the two flavors the engine supports,
**Enemy** and **FriendlyNpc** (`Editor::BehaviourTreeType::EnemyNpc` /
`FriendlyNpc`) — including the file-format notes needed to hand-edit safely.

The two flavors share the same composite/control node types and file-format shell
(cereal JSON, `.meta` convention); only the leaf **ActionNode** — each flavor's own
`ActionBase` hierarchy, factory, and concrete action classes — differs.

---

## 1. Where things live

| thing | Enemy | FriendlyNpc |
|---|---|---|
| tree data files | `Assets/Data/EnemyBehaviour/*.enemyBehaviourData` (+ `*.meta`) | `Assets/Data/FriendlyNpcBehviour/*.friendBehaviourData` (+ `*.meta`) |
| runtime tree object | `Assets/Scripts/Core/Game/Npc/Enemy/Behaviour/Enemy_BehaviourTree.{h,cpp}` | `Assets/Scripts/Core/Game/Npc/Friendly/Behaviour/Friendly_BehaviourTree.{h,cpp}` |
| asset wrapper (ScriptableObject) | `Assets/Data/EnemyBehaviour/Data_EnemyBehaviourFile.{h,cpp}` | `Assets/Data/FriendlyNpcBehviour/Data_FriendNpcBehaviourFile.{h,cpp}` |
| ActionNode (wraps one ActionBase) | `Assets/Scripts/Editor/Npc/Enemy/Behaviour/Action/Enemy_Behaviour_ActionNode.{h,cpp}` | `Assets/Scripts/Editor/Npc/Friendly/Behaviour/Action/Friendly_Behaviour_ActionNode.{h,cpp}` |
| action base class | `.../Enemy/Behaviour/Action/Enemy_Behaviour_ActionBase.h` | `.../Friendly/Behaviour/Action/Friendly_Behaviour_ActionBase.h` |
| concrete actions | `.../Enemy/Behaviour/Action/Content/<Category>/<Name>/` | `.../Friendly/Behaviour/Action/Content/<Category>/<Name>/` |
| editor include aggregator | `.../Editor/Npc/Enemy/Behaviour/Action/Enemy_Behaviour_ActionHeaders.h` | `.../Editor/Npc/Friendly/Behaviour/Action/Friendly_Behaviour_ActionHeaders.h` |
| registration macro | `REGISTER_ENEMY_ACTION[_WITH_NAME]` | `REGISTER_FRIENDLY_ACTION[_WITH_NAME]` |

Node types shared by both (Selector, Sequence, RandomSelector, OnceExecute,
OnceSuccess, Entry) live once, flavor-agnostic, under
`Assets/Scripts/Editor/BehaviourTree/Window/Node/**`.

An enemy binds a tree through `EnemyBase::behaviourData_` (a `FIELD(Asset::EnemyBehaviourFile)`);
a friendly NPC through `FriendlyNpc::friendlyNpcBehaviourFile_` (a
`FIELD(Asset::FriendNpcBehaviourFile)`). Either way the prefab stores the asset
**GUID**, which must equal the `guid_` in the tree's `.meta` (e.g.
`Assets/Prefab/Npc/Enemy/Hyena.prefab` ↔ `HyenaBehaviour.enemyBehaviourData.meta`).
`OnAwake` loads the tree; `OnUpdate` ticks it - on both components.
To bind a tree, either drag the asset onto that field in the prefab inspector, or edit the prefab
JSON so the field's `...value0.ptr_wrapper.data.value0.value_` equals the tree's GUID.

### In-engine graph editor (Behaviour Tree window)

Drawn with ImGuizmo's `GraphEditor` through the same engine pieces as the AnimationTree editor
(`Engine/Module/Gui/Graph/Editor/`: `GraphEditorHost` = toolbar / pan / zoom / fit / minimap,
`GraphDelegateBase` = selection, context menus, `Delete` key). The BT side is one adapter shared by both
flavors, `Assets/Scripts/Editor/BehaviourTree/Window/Graph/BehaviourTreeGraphDelegate`; nodes plug in through
`NodeBase`'s graph hooks (`GraphHeaderColor`, `GraphNodeTitle`/`Detail`, `MaxChildren`,
`RemoveChild`/`InsertChild`, `DrawGraphContextMenuItems`). Nodes keep their saved `position_`.

| action | how |
|---|---|
| pan / zoom / fit | middle-drag / wheel / `F` or toolbar *Fit All* (*Fit Selected* for the selection) |
| select | click a node (shows it in the Inspector), left-drag on empty space for a box, Shift to add |
| move | drag selected nodes - their whole subtrees follow |
| connect / re-parent | drag from a parent's output slot (right) to a child's input slot (left); a child has one parent, so this moves it. A single-child node (Entry, OnceExecute, OnceSuccess) drops its old child to *detached* |
| disconnect | drag the link off the child's input slot, right-click the link → *Disconnect*, or the node menu's *Disconnect from Parent*. Re-connecting to the same parent puts it back at its old position (and weight) |
| add | right-click empty space → *Create Node* / *Paste* (detached, at the cursor), or a node → *Create Child* / *Paste as Child* |
| delete | select + `Delete`, or the node menu: *Delete Node* (children become detached) / *Delete Subtree*. Entry can't be deleted |
| action type | right-click an ActionNode → *Action* |
| child order | the Inspector's ↑↓ (the graph shows `#n` on each child of a multi-child node) |
| tidy layout | *整列*: children to the right of their parent, siblings stacked top to bottom in execution order, node widths fitted to their labels; a Sequence whose children are all actions is folded into one list node |
| fold / unfold | double-click a node |

Subtrees not reachable from Entry are **detached**: shaded, labelled, never ticked, and saved in the file's
`detachedNodes_` so they can be wired back later. The *Running BehaviourTree Viewer* shows the same graph
read-only, with each node outlined by its last tick result (green Success, yellow Running, red Failure,
purple Abort).

---

## 2. `.enemyBehaviourData` / `.friendBehaviourData` format

Output of `cereal::JSONOutputArchive` (`BehaviourTree::OnSave`). Two top-level keys,
in this order, plus an optional third:

* `entryNode_` — `shared_ptr<Editor::Npc::Behaviour::EntryNode>`, the graph root.
  Its `nextNode_` holds the actual tree (or `{"polymorphic_id": 0}` when empty).
* `parameters_` — `unique_ptr<ParameterGroup>`, the blackboard.
* `detachedNodes_` — `vector<shared_ptr<NodeBase>>`, the graph editor's detached subtree roots (never
  ticked). Written only when non-empty and optional on load, so older files are unchanged.

On disk: UTF-8, **no BOM**, **CRLF**, no trailing newline, 4-space indent.

### cereal bookkeeping you will see everywhere

| construct | meaning |
|---|---|
| `"polymorphic_id": 0` | null polymorphic pointer |
| `"polymorphic_id": 1073741824` | pointer whose dynamic type == static type (no type name needed) |
| `"polymorphic_id": 0x80000000\|N` + `"polymorphic_name"` | first time type *N* is written; later refs use just `"polymorphic_id": N` |
| `"ptr_wrapper": {"id": 0x80000000\|K, "data": …}` | a `shared_ptr` (K counts every shared_ptr **and** every `Field<T>` inner pointer, in write order) |
| `"ptr_wrapper": {"valid": 1, "data": …}` | a `unique_ptr` (`action_`, `parameters_`) |
| `"cereal_class_version": V` | written **once per type per file**, on that type's first instance; omitted afterwards |
| nested `"value0"` | a serialised base class (`NodeBase` → `IObject`; each action → `ActionBase`) |

The once-per-file version slot matters when hand-editing: `Field<T>::load` reads its `shared_ptr`
positionally, so a stray repeated `"cereal_class_version"` on a later `FIELD(Asset::X)` is consumed as the
pointer and **the engine fails to load the tree** (rapidjson `IsObject()` → `SerializationException`).

Every node object is `{ [cereal_class_version,] value0: <NodeBase header>, <derived fields> }`
where the NodeBase header is `{ [ccv,] value0: <IObject {}>, guid_: {value_: "<UPPER-GUID>"},
position_: {value0: x, value1: y} }` (`position_` is editor-canvas coords).

### Node types

| type (`polymorphic_name`) | `CEREAL_CLASS_VERSION` | derived fields |
|---|---|---|
| `Editor::Npc::Behaviour::EntryNode` | 0 | `nextNode_` |
| `Editor::Npc::Behaviour::SelectorNode` | 0 | `children_[]` — first child to return Success/Running wins |
| `Editor::Npc::Behaviour::SequenceNode` | 0 | `children_[]` — stops at first Failure/Running |
| `Editor::Npc::Behaviour::RandomSelectorNode` | 1 | `children_[]`, `weights_[]` (one int per child) — picks **one** weighted child per tick and returns exactly what it returns, with **no fallback**: unlike `SelectorNode`, a child that fails makes the whole node fail that tick, it does not try another child. A branch can't be made conditional in isolation inside a `RandomSelector` — if that branch's guard fails, the pick is wasted, not retried — so a variant that's sometimes unavailable needs its own separate weighted pool (see `Editor::Npc::Behaviour::SelectorNode` above, gated by e.g. a blackboard condition, with each pool as one branch), not a guard clause on one child. |
| `Editor::Npc::Behaviour::OnceExecute` | 0 | `child_`, `state_` |
| `Editor::Npc::Behaviour::OnceSuccessNode` | 0 | `child_` |
| `Editor::Npc::Behaviour::BlackBoardGate` | 0 | `child_` (may be null), `conditions_[]`, `writesOnStart_[]`, `writesOnSuccess_[]` (each `{keyName_, value_}`, int blackboard), `once_` — replaces `Seq[ReadBlackBoard..., X, WriteBlackBoard...]`: fails unless every condition matches (checked every tick, like a Sequence), writes `writesOnStart_` when it enters the child, `writesOnSuccess_` when the child succeeds. No child = a pure condition/write node. `once_` latches the result like `OnceExecute`. Shared by both flavors. |
| `Editor::Npc::Enemy::Behaviour::ActionNode` | 1 | `name_` (label), `action_` (`unique_ptr<GameCore::Npc::Enemy::Behaviour::ActionBase>`) |
| `Editor::Npc::Friendly::Behaviour::ActionNode` | 1 | `name_` (label), `action_` (`unique_ptr<GameCore::Npc::Friendly::Behaviour::ActionBase>`) |

Selector/Sequence keep no state, so an earlier branch that starts succeeding or running takes over
from a later one that was `Running`. For enemy actions the pre-empted action notices this
(`ActionBase::Tick`: it returned Running but wasn't ticked in the previous tree tick,
`TickContext::TickIndex()`), and it calls `Reset()` before its next tick. So an interrupted attack or
wait restarts from the beginning instead of resuming halfway through. Only actions are reset;
composite and decorator state (RandomSelector's pick, OnceExecute) is left alone.

**Flinch / attack cancel (enemies):** every damage source carries a flinch value in
`Damage::PhysicsPower::flinchPower_` (player attacks in `SwordManInitStatus`, spells / items /
cannon in their data). `EnemyStatus::Flinch` returns `Running` for `flinch_secs_` once a hit's flinch
value exceeds `flinchResistance_`, and sets the Animator's `State` to `animatorSetParam_`. Put it in the
root Selector's damage branch, after `OnDamage` and the death sequence. The animation needs an
any-state transition with `hasExitTime_` off (see `docs/AnimationTree.md`) so that it cuts the current
clip. Set `isStopHorizontalMove_` to false when `OnDamage` applies knockback. Hyena is the reference setup.

**Bundling nodes (enemies):** prefer these over long chains of small nodes.

* `Timeline::ActionTimeline` - `cues_[]` of `{at_secs_, waitDone_, keepTicking_, action_}` (`action_` is any
  enemy action, serialised like an ActionNode's), `duration_secs_`, `failOnChildFailure_`, `once_`. Starts each
  cue when its time comes and returns Running until `duration_secs_` has passed and every `waitDone_` cue has
  finished. Replaces `Seq[PlayAnimation, Wait, ShakeCamera, Wait, PlaySE, ...]` and `OnceExecute` around one-shot
  effects: a cue runs **once** unless `keepTicking_` (then it is re-ticked every frame until the timeline ends,
  the way a Sequence re-ticks earlier children - `PlayAnimation`'s delayed sound and a held `SetLinearVelocity`
  need it). Like `WaitSeconds`, a finished timeline ticked again on the next tree tick stays finished, so it can
  sit in the middle of a Sequence.
* `EnemyStatus::PlayAnimation.holdSeconds_` (v3) - Running for that long instead of a following `WaitSeconds`.
* `Other::RandomWriteBlackBoard<Int>` - `keyName_`, `choices_[] {value_, weight_}`; replaces
  `RandomSelector[WriteBlackBoard...]`.
* `Basic::PlayerAngleDispatch` - `keyName_`, `ranges_[] {minDegree_, maxDegree_, useAbsolute_, value_}`,
  `useFallback_`, `fallbackValue_`; writes the first range that contains the angle to the nearest player
  (same angle as `ToPlayerAngle`). Replaces `Selector[Seq[ToPlayerAngle, WriteBlackBoard]..., WriteBlackBoard]`.

A file mixes only one ActionNode flavor; which one it is fixes the flavor of the whole tree.

The action `data` block is
`{ [ccv = <action CEREAL_CLASS_VERSION>,] value0: <ActionBase {}>, <members in save() order> }`.
`CEREAL_NVP(x_)` keeps the key `x_`; a bare `archive(x_)` serialises as `value1`, `value2`, …
`FIELD(T)` members serialise as a versioned wrapper holding an inner
`ptr_wrapper` whose data is `{value0: {value_: "<asset GUID>"}}`.

### `parameters_` (blackboard)

`{ptr_wrapper: {valid: 1, data: {value0: <count>, value1: <param>, value2: <param>, …}}}`.
Only `NanamiEngine::Module::AnimationTree::AnimationParameter<int>` is used today:
`{name_: "<key>", value_: <int>}`.

### `.meta`

`cereal::JSONOutputArchive` of a `shared_ptr<AssetBase>` = `EnemyBehaviourFile` /
`FriendNpcBehaviourFile` with `contentPath_` (backslash dirs, forward slash before
the filename) and the stable asset `guid_`. Created by `File::OnSave`.
**Don't hand-write it.**

---

## 3. Adding a new behaviour action

### The anatomy (Enemy; FriendlyNpc is the same shape with `Friendly` swapped in)

1. **`Enemy_Behaviour_Action_<Name>.h`** under `Content/<Category>/<Name>/`:
   `class <Name> final : public ActionBase` in `namespace GameCore::Npc::Enemy::Behaviour::Action`;
   override `TickStatus DoTick(const TickContext&)` (return `Success` / `Running` /
   `Failure` / `Abort` from `TickStatus.h`); optional `void DoDrawGui()`.
   Parameter members are `[[serialize(0)]]`-tagged; hand-write
   `template<class Archive> void save/load(Archive&, const std::uint32_t version)`
   that first archives `cereal::base_class<ActionBase>(this)` then `CEREAL_NVP(each_)`
   (gate newer members in `load` with `if (version >= N)`).
   After the class: `REGISTER_ENEMY_ACTION_WITH_NAME(<Name>, "<Category>::<Name>")`
   (inside the namespace), then at global scope `CEREAL_CLASS_VERSION` (only if
   versioned). The type registration goes in the `.cpp`, not the header (see 2.).
   Copy `Content/Other/Sample/MoveFront/…SampleMoveFront.{h,cpp}` (no params) or
   `Content/Wait/Seconds/…WaitSeconds.{h,cpp}` (one param) as a starting point.

2. **`Enemy_Behaviour_Action_<Name>.cpp`**: `#include` its own header, implement
   `DoTick` / `DoDrawGui` in `namespace GameCore::Npc::Enemy::Behaviour` with an
   `Action::` qualifier. `stdafx.h` is force-included by the project — no PCH line needed.
   Also `#include` `Engine/Module/Serialization/Engine_Module_SerializationRegistration.h`
   (relative path) and end the file, at global scope, with
   `NANAMI_REGISTER_TYPE(<fqn>, GameCore::Npc::Enemy::Behaviour::ActionBase);` (the engine's wrapper
   around `CEREAL_REGISTER_TYPE` + `CEREAL_REGISTER_POLYMORPHIC_RELATION`; it also records the
   registering module for hot reload — never call the cereal macros directly).
   Registering in the header would re-instantiate the type's serialisers in every file that
   includes it, which is what used to dominate the build time.

3. **Three wiring points** (no code-gen / globbing exists):
   * `Assets/Scripts/Editor/Npc/Enemy/Behaviour/Action/Enemy_Behaviour_ActionHeaders.h`
     — add one `#include "../../../../../Core/Game/Npc/Enemy/Behaviour/Action/Content/<Category>/<Name>/Enemy_Behaviour_Action_<Name>.h"`.
   * `EnviroHunter.vcxproj` — add `<ClCompile Include="Assets\Scripts\…\<Name>.cpp" />`
     and `<ClInclude Include="Assets\Scripts\…\<Name>.h" />` (bare entries inherit the
     cereal include dirs + `/bigobj` + forced `stdafx.h` from the `Debug|x64`
     `ItemDefinitionGroup`). **Mandatory.**
   * `EnviroHunter.vcxproj.filters` — mirror the two entries with `<Filter>Source Files</Filter>`
     / `<Filter>Header Files</Filter>`. Optional (Solution Explorer only).

Then build:

```
MSBuild.exe EnviroHunter.sln -p:Configuration=Debug -p:Platform=x64 -p:PreferredToolArchitecture=x64 -m:12
```

`.h`/`.cpp` in this repo are **UTF-8 with a BOM**.

---

## 4. Verifying changes end-to-end

1. For a new action: build with the MSBuild line above and confirm the new TU compiles.
2. Point a scratch enemy prefab's `behaviourData_` (or friendly NPC prefab's
   `friendlyNpcBehaviourFile_`) GUID at the tree, launch the editor, confirm it
   loads and ticks.
