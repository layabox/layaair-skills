# Scene and prefab serialization

Read this reference when inspecting or editing `.ls` scene or `.lh` prefab JSON. It explains relationships and serialization conventions; use `schema-get <type> --inherited` for concrete node/component properties, defaults, enums, and inheritance. Select the project's UI implementation before selecting node types.

For ordinary scene or prefab operations, use the available LayaAirIDE MCP tools. For JSON inspection and edits, use `Laya_ReadLayaAsset` and `Laya_EditAsset`. Explicit validation uses `Laya_ValidateLayaAsset`, which delegates to the IDE's `IEditorEnv.HierarchyValidator`. Treat validator failures as errors; do not implement a parallel set of format checks from this reference.

## Document and object structure

Keys beginning with `_$` carry serialization instructions. Other keys represent serialized properties interpreted using the registered type and its serialization metadata. A TypeScript member's existence alone does not establish that it is a writable serialized property.

A hierarchy document header uses the numeric value `"_$ver": 1`. Preserve the scene structure produced by the IDE, including any embedded 3D hierarchy; do not flatten it into a prefab root. The following is a minimal ordinary prefab shape, not a template for every scene:

```json
{
  "_$ver": 1,
  "_$id": "k3m8q1vx",
  "_$type": "Sprite",
  "name": "Root",
  "_$child": [
    { "_$id": "7n2c9w4b", "_$type": "Sprite", "name": "Child" }
  ]
}
```

- `_$id` identifies a node within its owning hierarchy. New node IDs, including prefab instance roots and added nodes, must be random 8-character strings from `0123456789abcdefghijklmnopqrstuvwxyz`, matching the IDE generator. Prefer IDE allocation; when authoring JSON directly, run a random generator and check for collisions in the owning hierarchy. Do not derive IDs from names, paths, roles, counters, or copy the example IDs. Semantic labels belong in `name`.
- Preserve existing IDs when editing, even if they do not match the new-ID convention: references and prefab overrides depend on them. Keep the actual target IDs in `_$ref`, `_$override`, and `_$parent`, including nested paths; these references must not receive newly generated IDs.
- `_$type` identifies a registered node or component type, or a registered custom script UUID. Do not invent script UUIDs; obtain them from the actual asset metadata.
- `_$child` contains child node records; `_$comp` contains attached component records.
- `_$runtime` on a root identifies its runtime script, commonly by a `res://` script UUID. It is distinct from attaching a component through `_$comp`.
- `_$var` marks named nodes for exposure through the root runtime script; it is not a general-purpose component field.
- Root `_$preloads` lists resource URLs to load with the hierarchy. The corresponding `_$preloadTypes` entries disambiguate loader types when needed, such as Spine data stored in a `.json` resource. Preserve their positional correspondence and use actual loader type names.

IDs are local object identities; asset UUIDs identify resources. They are not interchangeable.

## Prefab instances and overrides

An instance uses `_$prefab` with the prefab asset UUID instead of an ordinary `_$type`. Its local `_$id` identifies the instance in the containing hierarchy. The source prefab retains its own internal IDs.

The instance's `_$child` can contain two distinct forms:

- An override record uses `_$override` to identify an existing source node and supplies changed properties.
- An added node uses its own `_$id` and node type (or prefab instance identity), plus `_$parent` to identify the source parent. Optional `_$index` specifies its insertion position among the parent's children.

Illustration only: the asset UUID and referenced source IDs must exist in the target project.

```json
{
  "_$id": "f9r2x6ka",
  "_$prefab": "3b15da0e-f2e4-4c83-92d2-bee8c233b1f7",
  "_$child": [
    { "_$override": "2v8m5q0s", "name": "InstanceLabel" },
    {
      "_$id": "c4z7n1hp",
      "_$type": "Sprite",
      "_$parent": "w6b0j9te",
      "_$index": 0,
      "name": "AddedChild"
    }
  ]
}
```

`_$override` and `_$parent` accept a source node ID or a path array. For nested prefabs, each intermediate array element identifies the nested prefab instance root at that level; the last element identifies the target node. For example, `["a5t9d3ru", "2v8m5q0s"]` targets `2v8m5q0s` inside `a5t9d3ru`. Read the source prefab before forming a path; a display name or a containing scene's unrelated node ID is not a substitute.

Editing an instance override changes that instance. Editing the source prefab changes the reusable resource. Choose the intended scope before writing.

## Components and component overrides

A normal component record uses `_$type`. A singleton component is identified by type; a non-singleton component also has its own `_$id` so multiple instances can be distinguished. Generate new component IDs as random 4-character strings from `0123456789abcdefghijklmnopqrstuvwxyz`, matching the IDE component generator, and check for collisions in the owning hierarchy. Preserve existing component IDs when editing. Obtain singleton behavior from the registered component metadata rather than guessing from its name; do not add IDs to singleton components.

A component override appears in the prefab instance's `_$comp`, or in the `_$comp` of an overridden source node. Its `_$override` identifies the source singleton's type or the source non-singleton component's ID. Preserve this identity instead of adding a new component record when the intent is to override an existing component.

Component override identity is distinct from the node-plus-type reference described below.

## Resource and object references

The destination property's serialized type determines the representation:

| Relationship | Value shape | Meaning |
| --- | --- | --- |
| String resource URL | `"res://<asset-uuid>"` | A string property such as an image URL |
| Object resource | `{ "_$uuid": "<asset-uuid>", "_$type": "Material" }` | A loaded resource object; use the type appropriate to the property |
| Node reference | `{ "_$ref": "7n2c9w4b" }` | A node in the hierarchy's reference scope |
| Component reference | `{ "_$ref": "7n2c9w4b", "_$type": "MeshFilter" }` | A component of the given registered type on the referenced node |
| Controller reference | `{ "_$ref": "7n2c9w4b", "_$ctrl": "c1" }` | A controller reference associated with the node and controller name |

A material property may use a `Material` object reference; a texture property needs its appropriate texture resource type. Do not replace a string resource URL with an object wrapper just because both point to assets. Resolve UUIDs through project assets; example UUIDs are placeholders, not usable resources.

For nested reference scopes, preserve the reference path emitted by the IDE and verify its target. Do not convert it into a guessed flat ID. A component reference by node and type does not select a particular non-singleton component by its `_$id`.

## Inline objects and template nodes

Registered value objects can carry `_$type` and their serialized fields, for example:

```json
{ "_$type": "Color", "r": 0.5, "g": 0.5, "b": 0.5, "a": 1 }
```

`{ "_$type": "Record", "entry": 1 }` represents a dictionary object. Its allowed value types still come from the destination property's metadata; it does not make arbitrary values valid.

Typed arrays use a registered array type and a `value` array, for example `{ "_$type": "Float32Array", "value": [1, 2, 3] }`. Use the array type expected by the property.

Template references combine a node reference with the destination template property name:

```json
{
  "_templateNode": { "_$ref": "u8g2s5ny", "_$tmpl": "itemTemplate" }
}
```

During decoding, the referenced node is assigned to `_templateNode`, and a prefab wrapper built from that node's serialized data is assigned to `itemTemplate`. This is a specialized serialization pattern used by controls such as lists; use the actual control's schema and IDE-produced data instead of adding these fields to arbitrary nodes.

## Source of truth

The current IDE hierarchy writer/parser, object decoder, registered property metadata, and official hierarchy validator govern actual behavior. Component schemas describe serialized properties, not the hierarchy identity keys above. The validator does not verify how an ID was generated; a validation pass does not replace random ID allocation. This reference explains how to read and combine these representations; it is not an exhaustive schema or a replacement validator.
