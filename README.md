# vize-runtime-prop-type-null-repro

Reported as [ubugeeei-prod/vize#11252](https://github.com/ubugeeei-prod/vize/issues/11252).

Minimal reproduction for invalid generated code in [vize](https://github.com/ubugeeei-prod/vize)'s content mapper.

A runtime `defineProps()` object with a `type: null` prop makes vize write `typeof` on the object literal itself, which is a syntax error. Compiled component libraries declare props this way: in one project using Nuxt UI 4.12.0, 172 of the 190 `.vue` components it ships failed like this, with 8,525 syntax errors in total.

## Steps

```sh
pnpm install
pnpm check
```

[`src/Box.vue`](./src/Box.vue):

```vue
<script setup lang="ts">
defineProps({ as: { type: null, required: false } });
</script>

<template>
  <div />
</template>
```

`pnpm check` runs `tsc --runExternalCode --noEmit -p tsconfig.json` with `"contentMappers": [{ "package": "vize", "extensions": [".vue"] }]`.

## Expected

No error, as on vize 0.608.0.

## Actual

```
src/Box.vue(14,36): error TS1003: Identifier expected.
  This location is in virtual code produced by the content mapper 'vize@0.627.0' and has no corresponding location in the original file.
src/Box.vue(14,64): error TS1005: ';' expected.
  This location is in virtual code produced by the content mapper 'vize@0.627.0' and has no corresponding location in the original file.
src/Box.vue(14,75): error TS1109: Expression expected.
  This location is in virtual code produced by the content mapper 'vize@0.627.0' and has no corresponding location in the original file.
src/Box.vue(15,1): error TS1128: Declaration or statement expected.
  This location is in virtual code produced by the content mapper 'vize@0.627.0' and has no corresponding location in the original file.
```

## Notes

* The generated code, from `vize check --save-virtual-ts-for src/Box.vue src`:

  ```ts
  "as"?: __RuntimePropShape<typeof { as: { type: null, required: false } }>["as"];
  ```

  [`runtime_prop_shape_member_type`](https://github.com/ubugeeei-prod/vize/blob/v0.627.0/crates/vize_canon/src/batch/virtual_project/setup_props/syntax.rs#L65-L70) expects an identifier for `typeof`, but for an inline object [`merge_imported_runtime_props_into_croquis`](https://github.com/ubugeeei-prod/vize/blob/v0.627.0/crates/vize_canon/src/batch/virtual_project/setup_props.rs#L73-L98) passes the `defineProps()` argument text itself as the binding.
* 0.608.0 passes and 0.609.0 fails. 0.609.0 includes [#11142](https://github.com/ubugeeei-prod/vize/pull/11142), which changed how [`collect_props_from_object`](https://github.com/ubugeeei-prod/vize/blob/v0.627.0/crates/vize_canon/src/batch/virtual_project/setup_props/runtime_props/object.rs#L37-L48) types these props.
* `lang="ts"` or not makes no difference. `{ type: String }` passes.
* `vize check src` hits the same syntax error, warns "Corsa CLI fast path unavailable; degraded to the slower project-session API", and then reports `No type errors found!`.
* `main` (a70648d3) has no change to the props code since v0.627.0.

## Environment

vize 0.627.0, TypeScript 7.1.0-beta, Vue 3.5.44, Node v26.12.1, pnpm 12.16.1, macOS (arm64)
