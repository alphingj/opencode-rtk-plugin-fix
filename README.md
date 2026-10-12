# opencode-rtk-plugin-fix
• Troubleshooting guide for RTK plugin load failures in OpenCode v2. Documents API incompatibilities, error logs, recovery steps, version checks, and the upstream compatibility fix.


# Fixing RTK Plugin Load Errors in OpenCode v2

A troubleshooting guide for the **RTK (Rust Token Killer) OpenCode plugin failing to load after upgrading from OpenCode v1 to OpenCode v2**.

This guide documents the error sequence, the root cause found in the local installation, the failed intermediate workarounds, and the supported path to a compatible plugin. It is written as a searchable technical incident report for developers and AI coding assistants.

> **Important:** The `ctx.tool || ctx` / `ctx.hooks.on` fallback shown during the original debugging chat was not verified and is not a reliable fix. Do not publish or use it as a working implementation. The RTK-specific compatibility patch described below was merged into the RTK `develop` branch on **October 12, 2026**; a stable release containing that patch was not verified when this README was written.

## At a glance

- **Affected component:** global RTK OpenCode plugin
- **Plugin path:** `~/.config/opencode/plugins/rtk.ts`
- **Log path:** `~/.local/share/opencode/log/opencode.log`
- **Observed environment:** OpenCode `v2.0.27`, RTK `0.44.2`, `@opencode-ai/plugin` `1.18.32`
- **Symptoms:** `Export named 'Plugin' not found`, `Missing key at ["default"]`, `Plugin must export a default definition with an id and an effect or setup function`, and `TypeError: undefined is not an object (evaluating '$.hook')`
- **Underlying issue:** the installed RTK plugin was written for the OpenCode v1 API, while OpenCode v2 expects a different plugin entry point and hook context.
- **Recommended fix:** install an RTK build that includes the upstream OpenCode compatibility change, regenerate the global plugin with that build, and verify that the installed RTK binary supports `rtk hook opencode`.

## What happened

### 1. The original plugin failed during module loading

The log included an error similar to:

```text
failed to load plugin
...
SyntaxError: Export named 'Plugin' not found in module
'@opencode-ai/plugin/dist/index.js'
```

The plugin had been changed to use a value import such as:

```ts
import { Plugin } from "@opencode-ai/plugin";
```

In the inspected local package version (`@opencode-ai/plugin@1.18.32`), `Plugin` was a TypeScript type, not a runtime value with a `Plugin.define()` method. Importing it as a value therefore failed at runtime.

### 2. Changing the export shape exposed a second error

OpenCode v2 then reported:

```text
Plugin must export a default definition with an id and an effect or setup function.
(cause: SchemaError(Expected object at ["default"]))
```

The old plugin used the v1 named-export style. The v2 loader expected a compatible default export containing the required plugin metadata and lifecycle entry point.

### 3. Wrapping the v1 function in a v2-shaped object exposed a third error

The next attempt failed with:

```text
TypeError: undefined is not an object (evaluating '$.hook')
```

The implementation still expected the OpenCode v1 context, where the plugin function could receive a Bun shell helper as `$`. That helper is not available in the OpenCode v2 plugin context. Merely putting a v1 function inside `setup(ctx)` does not migrate its runtime behavior.

### 4. The problem was global, not project-specific

The failing file lived in the global plugin directory:

```text
~/.config/opencode/plugins/rtk.ts
```

That means the failure could affect every OpenCode project for this user. The affected project did not need an `opencode.json` or `opencode.jsonc` file for the global plugin to fail.

The investigation also found a backup file, `~/.config/opencode/plugins/rtk.ts.bak`, containing the previous v1-style implementation. That backup may be useful for forensic comparison or a v1 environment, but restoring it alone does **not** make the plugin compatible with OpenCode v2.

## Root cause

This was an API compatibility problem, made worse by mixing v1 and v2 plugin patterns:

| OpenCode v1 pattern | OpenCode v2 pattern |
|---|---|
| Plugin commonly exported as a named async function | Plugin loader expects a default plugin definition/object |
| Hooks returned under keys such as `tool.execute.before` | Tool hooks registered through `ctx.tool.hook("execute.before", callback)` |
| Hook callbacks use the v1 input/output shape | Hook callback receives a v2 event object, including fields such as `event.tool` and `event.input` |
| Legacy plugin may use the Bun shell helper `$` | Do not assume `$` exists in the v2 context; use a compatible subprocess mechanism when needed |
| `@opencode-ai/plugin` v1 package/type contract | The documented v2 API uses `@opencode/plugin` for `Plugin.define()` |

The exact API should always be checked against the OpenCode version being run. See the official [OpenCode v1-to-v2 migration guide](https://dev.opencode.ai/v2/docs/build/plugins/migrate-v1/) and [v2 plugin documentation](https://dev.opencode.ai/v2/docs/build/plugins/).

## Recommended resolution for RTK

### Step 1 — Inspect the installed versions

Run:

```bash
opencode --version
rtk --version
rtk hook opencode --help
```

The third command is particularly important for the current RTK compatibility implementation. If the installed binary says that `hook opencode` is an unknown subcommand or otherwise cannot run it, that binary does not provide the interface expected by the newer plugin. **Do not copy the current upstream plugin over an older RTK binary and assume it will work.**

### Step 2 — Back up the current global plugin

```bash
cp "$HOME/.config/opencode/plugins/rtk.ts" \
   "$HOME/.config/opencode/plugins/rtk.ts.before-fix" 2>/dev/null || true
```

Inspect existing files before restoring or replacing anything:

```bash
ls -l "$HOME/.config/opencode/plugins/rtk.ts"* 2>/dev/null
```

### Step 3 — Upgrade to an RTK build that contains the compatibility fix

RTK's compatibility change was merged into the upstream [`rtk-ai/rtk` `develop` branch](https://github.com/rtk-ai/rtk/pull/4187) on **October 12, 2026**. It replaces the incompatible generated plugin with an implementation intended to support OpenCode v2 and compatible OpenCode v1 versions.

The installed version in this incident was RTK `0.44.2`, which predates that merged change. Use a stable RTK release that includes the patch when available, or follow the RTK project's instructions for using a build containing the merged commit. The merge date alone does not prove that a packaged release has already been published.

### Step 4 — Regenerate the global plugin with the updated RTK binary

After installing a compatible RTK build, run:

```bash
rtk init -g --opencode
rtk init --show
```

Check that the global plugin exists and that RTK reports OpenCode as installed. Review any message printed by `rtk init`; do not assume that a failed or skipped initialization replaced the broken file.

### Step 5 — Check ownership and permissions if edits fail

The incident file was reported as owned by `root`, which prevented normal user-level edits. If the file is still root-owned, correct its ownership and permissions after verifying the path:

```bash
sudo chown "$(id -un):$(id -gn)" "$HOME/.config/opencode/plugins/rtk.ts"
chmod 644 "$HOME/.config/opencode/plugins/rtk.ts"
```

Use `sudo` only where required. Do not change ownership of the entire OpenCode configuration directory unless there is separate evidence that it is necessary.

### Step 6 — Reload OpenCode and verify the logs

Restart the OpenCode app/server using the method appropriate for your installation, or allow its plugin hot reload to pick up the file if supported. Then inspect recent logs:

```bash
tail -n 100 "$HOME/.local/share/opencode/log/opencode.log" \
  | grep -E 'failed to load plugin.*rtk.ts|err_9effe22d|err_8bd3ce63|TypeError.*\$\.hook'
```

A clean grep result is useful, but not sufficient by itself: also verify that the plugin is installed and that a shell command is actually rewritten as expected.

## Temporary workaround

Until a compatible plugin and RTK binary are installed, RTK can still be invoked explicitly for supported commands, for example:

```bash
rtk git status
rtk ls -la
```

The issue affects automatic command rewriting from OpenCode, not necessarily direct RTK invocations. The explicit commands are a workaround, not proof that the OpenCode plugin has been repaired.

## Why the earlier attempted fixes were incomplete

The following changes are **not sufficient by themselves**:

- Changing `import { Plugin } from "@opencode-ai/plugin"` to `import type { Plugin } ...` fixes the value-import mistake but does not convert a v1 plugin into a v2 plugin.
- Exporting a raw v1 async function as `default` does not satisfy the v2 loader's required plugin definition shape.
- Exporting `{ id, setup }` while retaining `async ({ $ }) => { $.hook(...) }` still calls a legacy API. In v2, the setup context is different and does not provide that `$` helper.
- Guessing between `ctx.tool`, `ctx.hooks`, or an event-emitter fallback is not a substitute for using the documented API. The v2 tool hook is registered with `ctx.tool.hook("execute.before", ...)`, with the v2 event-object contract.
- Copying only a generic `Plugin.define()` skeleton does not recreate RTK's command rewriting logic or account for RTK's subprocess/exit-code behavior.

For these reasons, use the RTK project's compatible implementation rather than the placeholder callback shown in the original chat. The upstream merged implementation can be reviewed here: [RTK OpenCode plugin source](https://github.com/rtk-ai/rtk/blob/develop/hooks/opencode/rtk.ts).

## Verification checklist

- [ ] `opencode --version` confirms the OpenCode version.
- [ ] `rtk --version` confirms which RTK binary is on the terminal's `PATH`.
- [ ] `rtk hook opencode --help` succeeds on a build expected to support the new plugin.
- [ ] `rtk init -g --opencode` completes successfully with that build.
- [ ] `rtk init --show` reports the OpenCode plugin as installed.
- [ ] `~/.config/opencode/plugins/rtk.ts` is readable and owned by the expected user.
- [ ] Recent logs contain no new plugin-load/schema/runtime failures.
- [ ] A representative shell command is automatically rewritten by OpenCode.
- [ ] Explicit commands such as `rtk git status` continue to work.

## FAQ and searchable terms

**What does `Missing key at ["default"]` mean?**  
The OpenCode v2 plugin loader could not find a valid default plugin definition in the loaded module. A v1 named export can trigger this failure.

**Why does `Plugin.define is not a function` or `Export named 'Plugin' not found` happen?**  
The code imported a v2-style runtime symbol from a package version that did not export that symbol as a runtime value. The package name and API version must match the installed OpenCode/plugin API.

**Why is `$.hook` undefined?**  
The code is using a legacy v1 context/helper from inside a v2 lifecycle function. A new export wrapper does not make the old context available.

**Is this a project configuration issue?**  
Not in the reported case. The failing file was under `~/.config/opencode/plugins/`, so it was a global plugin problem.

**Does restarting OpenCode fix it?**  
Restarting can reload a corrected plugin, but it cannot make an incompatible plugin API compatible. Fix the plugin and binary first, then reload and verify.

**Can I restore `rtk.ts.bak`?**  
Keep it as a backup. It contains the old v1 implementation and is not, by itself, a v2 fix.

## References

- [OpenCode v2 plugin documentation](https://dev.opencode.ai/v2/docs/build/plugins/)
- [Official OpenCode v1-to-v2 migration guide](https://dev.opencode.ai/v2/docs/build/plugins/migrate-v1/)
- [RTK compatibility pull request #4187 — merged October 12, 2026](https://github.com/rtk-ai/rtk/pull/4187)
- [RTK issue #4311: Plugin failed in OpenCode v2](https://github.com/rtk-ai/rtk/issues/4311)
- [RTK OpenCode plugin source on `develop`](https://github.com/rtk-ai/rtk/blob/develop/hooks/opencode/rtk.ts)

---

**Scope note:** This is a community troubleshooting report based on one local incident and the linked upstream documentation/changes. It is not an official RTK or OpenCode support statement. Exact commands and compatibility depend on the installed versions.
