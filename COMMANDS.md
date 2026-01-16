# COMMAND AUTHORING (engine-first, TS contributors)

Version: 4.0
Last updated: 2025-12-24

Scope
This document is for people who write command files in this repo. It is strict and engine-first: do NOT reimplement parsing, cooldowns, subcommand routing, flag handling, or permission enforcement — the engine (`CommandFiles/plugins/handleCommand.js` and `CommandFiles/modules/*`) already provides those features. Read this before authoring or changing commands.

Core rules (read before coding)
- Use the runtime primitives provided by the engine — don't reimplement them.
- Always declare intent in `meta` (args, permissions, waitingTime, ext_plugins, whiteList). The engine will honor and surface those.
- Use `SpectralCMDHome` for any subcommands / menu flows; the engine exposes `spectralArgs` and structured handlers — don't parse subcommands manually.
- Use `NeaxScript` patterns only for scripting/terminal features; the engine integrates with those generators.
- Prefer TypeScript. All examples below are `.ts` and follow the repo patterns.

Where the engine helps (stop reimplementing these)
- Argument normalization and cooldown handling
- `cancelCooldown()` and `setCooldown()` exposure
- Permission checks (meta.permissions + runtime role checks)
- Alias resolution and `otherNames` handling
- Subcommand routing via `SpectralCMDHome` and `configs` handlers
- Output methods (reply/edit/dispatch/reaction) with consistent return types
- DB handles injection (`usersDB`, `globalDB`, `threadsDB`, `money`, `Inventory`)

Minimal TypeScript command template (engine-first)

```ts
// CommandFiles/commands/example.ts
import { SpectralCMDHome } from '@cass-modules/spectralCMDHome';

export const meta: CommandMeta = {
  name: 'example',
  otherNames: ['ex'],
  author: '@you',
  version: '1.0.0',
  description: 'Engine-first example command',
  usage: '{prefix}{name} <arg>',
  category: 'Utility',
  permissions: [0],
  waitingTime: 3,
  ext_plugins: {},
};

export const style: CommandStyle = {
  title: 'Example',
  titleFont: 'bold',
  contentFont: 'fancy',
};

export async function entry(ctx: CommandContext): Promise<OutputResult> {
  // The runtime has already normalized args, prefix, sender role, etc.
  const { args, input, output, prefix, commandName, cancelCooldown } = ctx;

  // Use meta to declare requirements instead of hand-rolling checks.
  if (meta.permissions?.includes(2) && !input.isAdmin) {
    return output.reply('❌ Bot admin only');
  }

  if (!args.length) {
    // Use cancelCooldown provided by engine instead of reimplementing cooldown logic
    cancelCooldown?.();
    return output.reply(`Usage: ${meta.usage.replace('{prefix}', prefix).replace('{name}', commandName)}`);
  }

  // Business logic here — use injected DB handles and global.utils
  const { usersDB } = ctx;
  const uid = input.sid || input.senderID;
  const user = (await usersDB.getItem(uid)) ?? {};

  await output.reaction('⏳');
  const m = await output.reply('Processing...');
  // do work
  await output.edit('Done ✅', m.messageID);
  return { success: true } as OutputResult;
}
```

Use meta.args and engine argument validators (do not implement your own arg system)
- Declare `meta.args` when you need per-argument UI validation and prompts. The engine uses these descriptors to generate help and to provide automatic fallback/response behavior.

Example `meta.args` snippet:

```ts
meta.args = [
  { degree: 0, fallback: null, response: 'Provide a target user', search: 'uid', required: true },
  { degree: 1, fallback: '1', response: 'Provide amount', search: 'number', required: false },
];
```

Subcommands: use SpectralCMDHome (exact pattern)
- Build a `configs: Config[]` array; each config is a subcommand definition the engine understands.
- Handlers receive `extra.spectralArgs` and `extra.self` for self-reflection.
- The home UI, help pages, cooldowns per subcommand and alias handling are managed by the Spectral engine.

```ts
import { SpectralCMDHome } from '@cass-modules/spectralCMDHome';

const configs: Config[] = [
  {
    key: 'list',
    description: 'List items',
    aliases: ['-l'],
    icon: '📋',
    async handler(ctx, extra) {
      // use ctx.output and extra.spectralArgs instead of parsing args yourself
      return ctx.output.reply('items...');
    },
  },
  {
    key: 'add',
    args: ['<id>'],
    async handler(ctx, extra) {
      const id = extra.spectralArgs[0];
      if (!id) return ctx.output.reply('Missing id');
      // engine will handle cooldowns, permission checks that you declare in meta/config
      return ctx.output.reply('Added ' + id);
    },
  },
];

export async function entry(ctx: CommandContext) {
  const home = new SpectralCMDHome({ isHypen: false }, configs);
  return home.runInContext(ctx); // let the engine run it fully
}
```

Flags/long options: rely on engine plugins
- The runtime and plugins (see `CommandFiles/plugins/*`) already provide flag parsing utilities where needed. Do not add a global flag parser unless you implement a small local helper strictly for command-local formatting that does not duplicate engine behavior.
- If your command requires flags, declare them in `meta`/Spectral configs and consume `extra.spectralArgs` or `ctx.input.arguments`.

Output & attachments
- Always use `output.reply`, `output.edit`, `output.reaction`, and `output.dispatch`.
- For images use `@napi-rs/canvas` and declare `{ canvas: '^1.0.0' }` in `meta.ext_plugins` so the engine knows the dependency.

DB & economy
- Use injected DB handles. The engine supplies `usersDB`, `globalDB`, `threadsDB`, `money`, `Inventory`. Use read-modify-write and call `cancelCooldown()` on validation failures.
- Example: check balance, deduct, persist. Do not attempt to implement custom transaction locking — the engine's DB abstractions are expected.

NeaxScript & scripts
- If you provide script-level functionality, implement generator-based commands only when integrating with `NeaxScript` module. The engine will call these generators appropriately.

Style & UI hints
- Export `style` to customize title/content fonts. The engine consumes these to render the command pages.
- Do not attempt to render HTML or special markup that the engine does not support.

Security (enforced by engine when you declare meta)
- Put permission constraints in `meta.permissions` and `meta.whiteList`. The engine enforces them; do not replicate checks in ad hoc ways.
- For hazardous commands (exec, eval) set `permissions: [2]` and `botAdmin: true` and still do robust input validation.

Testing & validation (engine-integrated)
- Use engine's test harness patterns: run commands in dev with `input.isAdmin` toggles, and inspect `menu`/`help` output.
- Validate `meta` fields in `menu` and `Spectral` flows; engine uses those fields to build UI.
- For `.ts` files run `tsc`.

Mandatory reading (do not skip)
- `CommandFiles/plugins/handleCommand.js` — how the engine normalizes args, applies cooldowns, and provides `cancelCooldown`, `setCooldown`, `recall`.
- `CommandFiles/modules/spectralCMDHome.ts` — Spectral subcommand system and handler signatures.
- `CommandFiles/modules/NeaxScript.ts` — scripting generator integration.
- Example commands to mimic (engine-first patterns): `admin.ts`, `menu.ts`, `vault.ts`, `quote_canv.ts`, `exec.js` (see guarded exec pattern).

Practical examples (TS-first, engine-driven)

1) Minimal command (engine handles everything):

```ts
export const meta: CommandMeta = {
  name: 'ping',
  description: 'Ping',
  permissions: [0],
  waitingTime: 1,
};
export async function entry({ output }: CommandContext) {
  return output.reply('pong');
}
```

2) Spectral command (do not parse subcommands manually):

```ts
import { SpectralCMDHome } from '@cass-modules/spectralCMDHome';
const configs: Config[] = [
  { key: 'foo', description: 'Foo', async handler(ctx, extra) { return ctx.output.reply('foo ' + (extra.spectralArgs.join(' ') || '')) } }
];
export async function entry(ctx: CommandContext) { return new SpectralCMDHome({ isHypen: false }, configs).runInContext(ctx); }
```

3) Canvas (declare ext_plugins, let engine handle runtime):

```ts
import { createCanvas } from '@napi-rs/canvas';
export const meta: CommandMeta = { name: 'img', ext_plugins: { canvas: '^1.0.0' }, permissions: [0] };
export async function entry({ args, output }: CommandContext) {
  const c = createCanvas(600, 300);
  const ctx = c.getContext('2d');
  ctx.fillStyle = '#000'; ctx.fillRect(0,0,600,300);
  ctx.fillStyle = '#fff'; ctx.fillText(args.join(' ') || 'no text', 20, 50);
  const buf = c.toBuffer('image/png');
  return output.dispatch({ buffer: buf, type: 'image/png' }, { caption: 'Generated' });
}
```

If you want, I will scaffold a strict TypeScript command now that follows the engine-first rules. Reply with: `scaffold <name> <short-purpose>`.
