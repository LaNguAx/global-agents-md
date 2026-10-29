# Manual shadcn setup: components, blocks, and other registry items offline in Vite

Verified on 2026-10-29. Target: an **existing React + Vite + TypeScript + Tailwind CSS v4 project**, using **Radix UI + Nova + Neutral**. Commands below use npm and Windows PowerShell.

This is a Vite setup. Block page files below are installed as ordinary React components. Do not add a Next.js dependency or create framework page routes as part of this guide.

## Instructions for the implementing agent

Implement the setup below in the existing project. Inspect its package manager, configuration, and applicable AGENTS.md first. Preserve existing application code, dependencies, Vite plugins, TypeScript options, and unrelated styles. Merge the configuration snippets; do not replace whole existing configuration files. Do not scaffold a new application, run `shadcn init`, create a worktree, or commit/push application changes unless separately instructed.

Keep shadcn source, generated components, helpers, styles, downloaded registry JSON, preparation script, and local registry server under `src/shadcn/`. Necessary integration points remain outside that directory: root `components.json`, package manifest/lockfile, TypeScript/Vite alias configuration, and the import in the application's existing global stylesheet. Dependencies remain in the project's normal `node_modules/`.

This guide is intended to be copied in full. All code for the manual setup is included below. The online preparation stage saves the full catalog, downloads selected item payloads and dependencies, prepares their destinations, and installs npm packages; those third-party artifacts must be available before using the setup offline.

## Why the local metadata server is included

The CLI supports `shadcn add ./component.json`, but **local component JSON is not a complete offline mode for the pinned CLI/configuration tested here**. When `shadcn@4.23.0` processes source files, its file updater calls `getRegistryBaseColor(config.tailwind.baseColor)` if the configured base color is nonempty. With this guide's `"baseColor": "neutral"`, that means a request for `colors/neutral.json` before processing the component files. Registry dependencies can also cause requests for other item JSON files.

**The running Vite application does not read neutral.json.** Its actual theme comes from `src/shadcn/styles.css`. The JSON is retained only because this particular stock CLI code path fetches it during installation. This was confirmed both by inspecting the installed updater and by the local server's `GET /colors/neutral.json 200` log. An earlier add attempt failed when that metadata lookup was blocked. A different CLI version, configuration, or custom installer may not need the file; do not generalize this requirement beyond the tested workflow.

This guide uses the official CLI unchanged. A tiny Node server serves the downloaded metadata/component files over `127.0.0.1`, and `REGISTRY_URL` directs CLI registry lookups to it. The component passed to `add` is still a local file. Loopback HTTP does not require an internet connection.

This is a tested workaround for the current CLI behavior, not a claim that the CLI has a documented comprehensive offline mode. It does not mirror npm: package dependencies must already be installed or available in an offline package cache. Remote URLs, GitHub addresses, or third-party namespaces inside registry dependencies also need separate preparation.

## 1. Install the shadcn packages during online preparation

Run from the existing project's root on a connected computer, or in a copy of that exact project on a compatible connected computer:

```powershell
npm install --save-exact shadcn@4.23.0 class-variance-authority@0.7.1 cn@0.5.0 lucide-react@1.64.0 radix-ui@1.8.1 tw-animate-css@1.4.0
```

These are the versions used in the verification project. `shadcn@latest` resolved to **4.23.0** during verification; pinning it makes the guide reproducible. Retain the resulting lockfile. Do not upgrade the existing React, Vite, TypeScript, or Tailwind versions simply to match the verification project. If the project's current versions cause a peer dependency conflict, investigate that conflict rather than forcing installation.

The current official manual guide uses the `cn` package. The live Radix Nova button imports `cn`, `class-variance-authority`, and `radix-ui`, although its own JSON currently lists only `cn` in `dependencies`; this is why the foundation packages are installed explicitly.

The `shadcn` npm package supplies both the local CLI and the `shadcn/tailwind.css` stylesheet used below. No Tailwind or Vite reinstall is needed for the already configured project.

If the existing project uses pnpm or Yarn, translate package-manager commands and preserve its existing lockfile instead of introducing a second package manager.

Sources: [official manual installation](https://ui.shadcn.com/docs/installation/manual), [live Radix Nova button JSON](https://ui.shadcn.com/r/styles/radix-nova/button.json).

## 2. Create the directory structure

```powershell
New-Item -ItemType Directory -Force -Path src/shadcn/ui,src/shadcn/components,src/shadcn/lib,src/shadcn/hooks,src/shadcn/registry/colors,src/shadcn/registry/styles/radix-nova | Out-Null
```

The resulting structure will be:

```text
project-root/
├── components.json
├── package.json
├── package-lock.json
└── src/
    ├── index.css                    # Existing application stylesheet
    └── shadcn/
        ├── styles.css
        ├── ui/                      # CLI-generated UI components
        ├── components/              # Prepared blocks, reusable components, and examples
        ├── lib/
        │   └── utils.ts
        ├── hooks/
        └── registry/
            ├── server.mjs
            ├── prepare.mjs
            ├── prepared-manifest.json
            ├── originals/           # Unmodified downloaded item payloads
            ├── index.json           # Legacy UI-only resolver metadata
            ├── colors/
            │   └── neutral.json     # Downloaded official color metadata
            └── styles/
                └── radix-nova/
                    ├── registry.json     # Full catalog, including blocks
                    ├── button.json       # Prepared installable payload
                    ├── alert-dialog.json
                    ├── login-01.json
                    └── ...               # Selected items plus dependency closure
```

## 3. Ensure the existing alias points to src

The guide uses `@/` to mean `src/`. If this already works in both TypeScript and Vite, keep the existing setup.

Otherwise, merge the following into `compilerOptions` in **both** `tsconfig.json` and `tsconfig.app.json` when the Vite project uses that standard split:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Keep the existing root `files`, `references`, and all other options. If the project has a single TypeScript configuration, apply the mapping there. Existing equivalent aliases do not need to be reconfigured.

If Vite does not already resolve this alias, add this import to `vite.config.ts`:

```ts
import { fileURLToPath, URL } from "node:url"
```

Merge this property into its existing `defineConfig` object:

```ts
resolve: {
  alias: {
    "@": fileURLToPath(new URL("./src", import.meta.url)),
  },
},
```

Preserve any other aliases and plugins, including the project's working React and Tailwind v4 integration. If Node type declarations are missing, install `@types/node` as a development dependency during online preparation.

Source: [official Vite installation instructions](https://ui.shadcn.com/docs/installation/vite#existing-project).

## 4. Create components.json at the project root

Create this complete file. If it already exists, inspect and deliberately merge the settings instead of silently overwriting it.

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "radix-nova",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/shadcn/styles.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/shadcn/components",
    "ui": "@/shadcn/ui",
    "lib": "@/shadcn/lib",
    "hooks": "@/shadcn/hooks",
    "utils": "@/shadcn/lib/utils"
  },
  "iconLibrary": "lucide"
}
```

`radix-nova` selects Radix-based Nova components. `base-nova` would select Base UI instead. `rsc: false` is appropriate for this Vite application. For Tailwind v4, `tailwind.config` is an empty string.

The aliases control generated file locations and imports. For example, a registry UI item is installed under `src/shadcn/ui/` rather than the usual `src/components/ui/`. The original `files[].path` inside an official JSON payload does not need to be manually rewritten.

`components.json` is the configuration the CLI reads; a successful prior `init` command is not required. Its `$schema` URL is an editor schema reference, not a component source. An editor might separately try to download the schema when offline.

Sources: [components.json reference](https://ui.shadcn.com/docs/components-json), [manual configuration example](https://ui.shadcn.com/docs/installation/manual#create-a-componentsjson-file).

## 5. Create the helper

Create `src/shadcn/lib/utils.ts`:

```ts
export { cn } from "cn"
```

Current components may import `cn` directly from its package. The helper also supplies the configured `aliases.utils` location for components/utilities that use that alias.

## 6. Create the full theme stylesheet

Create `src/shadcn/styles.css` with this complete content. This uses the Neutral base tokens from the current manual installation documentation, including light/dark, chart, sidebar, and radius tokens. It retains the application's existing font; it does not download a preset font.

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";

@custom-variant dark (&:is(.dark *));

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);
  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);
  --radius-sm: calc(var(--radius) * 0.6);
  --radius-md: calc(var(--radius) * 0.8);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) * 1.4);
  --radius-2xl: calc(var(--radius) * 1.8);
  --radius-3xl: calc(var(--radius) * 2.2);
  --radius-4xl: calc(var(--radius) * 2.6);
  --color-sidebar: var(--sidebar);
  --color-sidebar-foreground: var(--sidebar-foreground);
  --color-sidebar-primary: var(--sidebar-primary);
  --color-sidebar-primary-foreground: var(--sidebar-primary-foreground);
  --color-sidebar-accent: var(--sidebar-accent);
  --color-sidebar-accent-foreground: var(--sidebar-accent-foreground);
  --color-sidebar-border: var(--sidebar-border);
  --color-sidebar-ring: var(--sidebar-ring);
}

:root {
  --radius: 0.625rem;
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);
  --popover: oklch(1 0 0);
  --popover-foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  --secondary: oklch(0.97 0 0);
  --secondary-foreground: oklch(0.205 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --accent: oklch(0.97 0 0);
  --accent-foreground: oklch(0.205 0 0);
  --destructive: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0 0);
  --input: oklch(0.922 0 0);
  --ring: oklch(0.708 0 0);
  --chart-1: oklch(0.646 0.222 41.116);
  --chart-2: oklch(0.6 0.118 184.704);
  --chart-3: oklch(0.398 0.07 227.392);
  --chart-4: oklch(0.828 0.189 84.429);
  --chart-5: oklch(0.769 0.188 70.08);
  --sidebar: oklch(0.985 0 0);
  --sidebar-foreground: oklch(0.145 0 0);
  --sidebar-primary: oklch(0.205 0 0);
  --sidebar-primary-foreground: oklch(0.985 0 0);
  --sidebar-accent: oklch(0.97 0 0);
  --sidebar-accent-foreground: oklch(0.205 0 0);
  --sidebar-border: oklch(0.922 0 0);
  --sidebar-ring: oklch(0.708 0 0);
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.205 0 0);
  --card-foreground: oklch(0.985 0 0);
  --popover: oklch(0.205 0 0);
  --popover-foreground: oklch(0.985 0 0);
  --primary: oklch(0.922 0 0);
  --primary-foreground: oklch(0.205 0 0);
  --secondary: oklch(0.269 0 0);
  --secondary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);
  --accent: oklch(0.269 0 0);
  --accent-foreground: oklch(0.985 0 0);
  --destructive: oklch(0.704 0.191 22.216);
  --border: oklch(1 0 0 / 10%);
  --input: oklch(1 0 0 / 15%);
  --ring: oklch(0.556 0 0);
  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.17 162.48);
  --chart-3: oklch(0.769 0.188 70.08);
  --chart-4: oklch(0.627 0.265 303.9);
  --chart-5: oklch(0.645 0.246 16.439);
  --sidebar: oklch(0.205 0 0);
  --sidebar-foreground: oklch(0.985 0 0);
  --sidebar-primary: oklch(0.488 0.243 264.376);
  --sidebar-primary-foreground: oklch(0.985 0 0);
  --sidebar-accent: oklch(0.269 0 0);
  --sidebar-accent-foreground: oklch(0.985 0 0);
  --sidebar-border: oklch(1 0 0 / 10%);
  --sidebar-ring: oklch(0.556 0 0);
}

@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

In the existing `src/index.css`, replace its existing `@import "tailwindcss";` with the following, keeping imports before ordinary CSS rules:

```css
@import "./shadcn/styles.css";
```

Keep unrelated application styles and custom Tailwind configuration. Avoid importing Tailwind, `tw-animate-css`, or the shadcn stylesheet twice. If equivalent shadcn theme definitions already exist, consolidate them into the new stylesheet instead of keeping conflicting duplicate `:root`, `.dark`, or `@theme` definitions. Check any relative `@source` paths if moving existing source directives; their location affects how they resolve.

Ensure `src/main.tsx` still imports the application's existing `index.css` once. If the project's global stylesheet has another name, update its import and relative path accordingly. The stylesheet is global despite living under `src/shadcn/`; this directory choice does not scope CSS to those components.

Source: [current official manual stylesheet](https://ui.shadcn.com/docs/installation/manual#configure-styles).

## 7. Save the full catalog and prepare selected items online

Use the **full style-specific catalog**, not the older UI-only index:

**[https://ui.shadcn.com/r/styles/radix-nova/registry.json](https://ui.shadcn.com/r/styles/radix-nova/registry.json)**

At verification it contained 219 items across eight types, including 30 `registry:block` entries. The catalog's `items` array describes available items and file paths; it does not contain all installable source contents. Each selected item still needs its own `NAME.json` payload and recursive dependencies.

Download the catalog and list blocks while connected:

```powershell
$ErrorActionPreference = 'Stop'
$catalogPath = 'src/shadcn/registry/styles/radix-nova/registry.json'
Invoke-WebRequest -Uri 'https://ui.shadcn.com/r/styles/radix-nova/registry.json' -OutFile $catalogPath
$catalog = Get-Content -LiteralPath $catalogPath -Raw | ConvertFrom-Json

$catalog.items | Group-Object type | Select-Object Name,Count
$catalog.items | Where-Object { $_.type -eq 'registry:block' } | Select-Object -ExpandProperty name
```

To list a different type, change the filter to `registry:hook`, `registry:font`, or one of the types below. The same listing works offline from the saved catalog.

| Catalog item type | Example | Offline installation behavior |
|---|---|---|
| `registry:ui` | `button` | Writes source under `src/shadcn/ui/`. |
| `registry:block` | `login-01` | Installs multiple files and dependencies; prepared pages and companion files stay together under `src/shadcn/components/blocks/NAME/`. |
| `registry:component` | `example` | Writes source under `src/shadcn/components/`; block components retain their block subdirectories. |
| `registry:hook` | `use-mobile` | Writes hooks under `src/shadcn/hooks/`; any library companion files go under `lib/`. |
| `registry:lib` | `utils` | Writes utilities under `src/shadcn/lib/`. |
| `registry:example` | `input-example` | Writes a demonstrative React component under `src/shadcn/components/examples/`; also installs its helper components and primitives. |
| `registry:style` | `style` | Updates `src/shadcn/styles.css`, may install utilities/packages, and can change existing global styling. It need not generate a TSX file. |
| `registry:font` | `font-inter` | For Vite, uses a local font package such as `@fontsource-variable/inter` and updates the configured stylesheet. The font package must be installed before going offline. |

All eight types were covered by the representative verification below. That does not establish that every item in the catalog is compatible with every Vite project.

### Prepare file destinations before installing blocks

The stock `login-01.json` contains a page targeting `app/login/page.tsx`. Aliases alone do not contain that target. The preparation script below saves the original payload, then produces a separate installable copy with an explicit target for **every source file**:

- UI files use `@ui/...`.
- Libraries use `@lib/...`.
- Hooks use `@hooks/...`.
- Other source files use `@components/...`, retaining block/example subdirectories.
- A `registry:page` file becomes a plain `registry:component` file in the prepared copy. Its original registry source path remains unchanged, so the CLI can rewrite imports against the planned destinations. It does not become a Vite route automatically.

For example, the prepared login page has:

```json
{
  "path": "registry/radix-nova/blocks/login-01/page.tsx",
  "type": "registry:component",
  "target": "@components/blocks/login-01/page.tsx",
  "content": "...the existing source content is preserved..."
}
```

The example above illustrates the changed fields; the script preserves the actual complete `content` string. Its generated companion form goes to `src/shadcn/components/blocks/login-01/components/login-form.tsx`. Keeping the original block subtree also preserves relative companion-file relationships, such as a page importing `./data.json`.

### Create the preparation script

Create `src/shadcn/registry/prepare.mjs` with this complete code:

```js
import { mkdir, writeFile } from "node:fs/promises"
import { dirname, join } from "node:path"
import { fileURLToPath } from "node:url"

const root = dirname(fileURLToPath(import.meta.url))
const style = "radix-nova"
const sourcePrefix = `registry/${style}/`
const origin = "https://ui.shadcn.com/r/"
const visited = new Set()
const runtime = new Set()
const development = new Set()
const preparedItems = []
const review = []

async function save(relative, data) {
  const destination = join(root, relative)
  await mkdir(dirname(destination), { recursive: true })
  await writeFile(destination, JSON.stringify(data, null, 2) + "\n")
}

async function download(relative) {
  const response = await fetch(new URL(relative, origin))
  if (!response.ok) throw new Error(`${relative}: HTTP ${response.status}`)
  return response.json()
}

function safeRelative(value) {
  if (!value || value.includes("\\") || value.includes(":") || value.startsWith("/") || value.split("/").some((part) => !part || part === "." || part === "..")) {
    throw new Error(`Review unexpected registry path: ${value}`)
  }
  return value
}

function prepareFile(file, itemName) {
  if (!file.path.startsWith(sourcePrefix) || typeof file.content !== "string") {
    throw new Error(`Expected built source content and ${sourcePrefix} path: ${file.path}`)
  }
  const relative = safeRelative(file.path.slice(sourcePrefix.length))
  let target
  if (file.type === "registry:ui") target = `@ui/${relative.replace(/^ui\//, "")}`
  else if (file.type === "registry:lib") target = `@lib/${relative.replace(/^lib\//, "")}`
  else if (file.type === "registry:hook") target = `@hooks/${relative.replace(/^hooks\//, "")}`
  else target = `@components/${relative.replace(/^components\//, "")}`

  if (/(?:from\s*|import\s*\(|import\s*)["']next(?:[\/"'])/.test(file.content)) {
    review.push(`${itemName}: adapt Next.js imports in ${file.path}`)
  }
  if (/(?:https?:\/\/|["']\/(?:images|placeholder|avatars|assets)[^"']*)/.test(file.content)) {
    review.push(`${itemName}: inspect URLs/public asset references in ${file.path}`)
  }
  return { ...file, type: file.type === "registry:page" ? "registry:component" : file.type, target }
}

async function prepare(name) {
  if (!/^[a-z0-9][a-z0-9-]*$/.test(name)) throw new Error(`Only built-in item names are automated: ${name}`)
  if (visited.has(name)) return
  visited.add(name)
  const raw = await download(`styles/${style}/${name}.json`)
  await save(`originals/styles/${style}/${name}.json`, raw)
  if (raw.type === "registry:base" || raw.config || Object.keys(raw.envVars ?? {}).length) {
    throw new Error(`${name}: review design-system/config/environment changes separately`)
  }
  const item = { ...raw, files: (raw.files ?? []).map((file) => prepareFile(file, name)) }
  for (const dependency of raw.dependencies ?? []) runtime.add(dependency)
  for (const dependency of raw.devDependencies ?? []) development.add(dependency)
  if (raw.font?.dependency) runtime.add(raw.font.dependency)
  for (const dependency of raw.registryDependencies ?? []) await prepare(dependency)
  await save(`styles/${style}/${name}.json`, item)
  preparedItems.push({ name, type: raw.type })
}

const catalog = await download(`styles/${style}/registry.json`)
await save(`styles/${style}/registry.json`, catalog)
await save("index.json", await download("index.json"))
await save("colors/neutral.json", await download("colors/neutral.json"))

const requested = process.argv.slice(2)
const names = requested.includes("--all") ? catalog.items.map((item) => item.name) : requested
if (!names.length) throw new Error("Pass item names, for example: button login-01 use-mobile utils input-example font-inter style")
for (const name of names) {
  if (!catalog.items.some((item) => item.name === name)) throw new Error(`Not in ${style} catalog: ${name}`)
  await prepare(name)
}

await save("prepared-manifest.json", {
  style,
  items: preparedItems,
  dependencies: [...runtime].sort(),
  devDependencies: [...development].sort(),
  review: [...new Set(review)],
})
console.log(`Prepared ${preparedItems.length} items. Read prepared-manifest.json and review Vite compatibility/assets before installation.`)
for (const note of new Set(review)) console.log(`REVIEW: ${note}`)
```

The script fetches selected **official Radix Nova items only**, recursively prepares named registry dependencies, keeps originals under `registry/originals/`, and records package requirements plus review notes in `prepared-manifest.json`. It preserves item-level types, CSS, theme variables, and font metadata. It changes file destinations and converts page files into plain component files for this Vite layout.

Unexpected paths, external/namespaced dependency addresses, and design-system/config/environment changes stop automatic preparation. Do not treat a partially completed run or a manifest from an earlier run as a newly complete snapshot. Correct the issue and complete preparation before installing. The manifest describes the last requested dependency closure; it is not an inventory of every previously downloaded file.

Run the following online to prepare the guide's examples across all eight item types:

```powershell
node src/shadcn/registry/prepare.mjs button alert-dialog login-01 use-mobile utils input-example style font-inter
```

For a different block, substitute its name or add it to the list. Examples of raw payloads for inspection:

- [Radix Nova Login 01](https://ui.shadcn.com/r/styles/radix-nova/login-01.json)
- [Radix Nova Sidebar 01](https://ui.shadcn.com/r/styles/radix-nova/sidebar-01.json)
- [Radix Nova Inter font item](https://ui.shadcn.com/r/styles/radix-nova/font-inter.json)

The script also supports `--all` to **download and prepare** the full catalog. That is optional, potentially large, and not an instruction to install all items. Do not install every style/font at once: they can overwrite the same styling and font variables. Full-catalog preparation and full-catalog Vite compatibility were not executed in this verification.

### Install the selected items' npm dependencies while connected

Read the manifest, including its `review` entries, before proceeding:

```powershell
$manifest = Get-Content -LiteralPath 'src/shadcn/registry/prepared-manifest.json' -Raw | ConvertFrom-Json
$manifest.review
$manifest.items | Format-Table name,type

$project = Get-Content -LiteralPath 'package.json' -Raw | ConvertFrom-Json
$declared = @()
if ($project.dependencies) { $declared += @($project.dependencies.PSObject.Properties.Name) }
if ($project.devDependencies) { $declared += @($project.devDependencies.PSObject.Properties.Name) }

function Get-RegistryPackageName([string]$spec) {
    if ($spec -notmatch '^(?:@[^/]+/)?[^@]+') { throw "Review package specification: $spec" }
    return $Matches[0]
}

$runtimePackages = @($manifest.dependencies | Where-Object {
    $name = Get-RegistryPackageName $_
    $name -ne 'shadcn' -and $name -notin $declared
})
$developmentPackages = @($manifest.devDependencies | Where-Object {
    $name = Get-RegistryPackageName $_
    $name -ne 'shadcn' -and $name -notin $declared
})

if ($runtimePackages.Count -gt 0) { npm install --save-exact @runtimePackages }
if ($developmentPackages.Count -gt 0) { npm install --save-dev --save-exact @developmentPackages }
```

Keep the CLI pinned to `shadcn@4.23.0` from step 1 and retain the lockfile. Do not let an unversioned manifest entry upgrade it. The commands above add missing declarations and preserve the classification of packages already declared in the project. They do not verify that existing versions satisfy every requested version range: inspect the manifest's requirements and resolve any conflicts while connected. Likewise, a declared package still needs to be installed in node_modules. Run the exact add command while connected before transferring, then verify again with internet disconnected. The temporary verification preinstalled all needed packages before the offline add commands.

Font requirements are included even when they are expressed through `font.dependency` rather than the normal `dependencies` array. For the sample font, this includes `@fontsource-variable/inter`. Package files remain in normal `node_modules/`; generated CSS and source remain under `src/shadcn/`. A production Vite build emits its normal bundled assets under `dist/`.

### Review Vite compatibility and runtime assets

Changing a file's destination does not adapt framework-specific source automatically. Review the selected payloads and dependency closure before installation:

1. Resolve any Next.js imports such as `next/link`, `next/image`, server-only utilities, server actions, or framework-specific routing. Adapt the **prepared copy** to your existing Vite application's router and data layer. Preserve the original payload for reference. Keep original `files[].path` identifiers for CLI import mapping; edit `files[].content` and prepared targets as needed. Re-running preparation overwrites prepared copies, so retain deliberate adaptations separately before refreshing them.
2. Download any actually required images/media while online into `src/shadcn/assets/`, then replace public-path or remote references with Vite asset imports such as `import illustration from "@/shadcn/assets/illustration.svg"`. Create that directory only if needed. Preserve asset licenses. Do not leave `/placeholder.svg`, remote images, CSS `url(...)` references, or API calls unresolved and claim runtime offline completeness.
3. Inspect `css`, `cssVars`, `dependencies`, `devDependencies`, and font metadata as well as file contents. The script's review list is a hint, not an exhaustive compatibility/network audit. For example, the inspected `input-example` contains an `https://example.com` **placeholder string**, which does not itself make a request.
4. Inspect styles/plugins and imported packages for requirements beyond source placement. Downloading a font item without its actual package/font files does not make it available offline.
5. Apply source adaptations in prepared JSON before CLI installation, or adapt the generated files under `src/shadcn/` afterward and verify the result. For a real login block, the form is UI; this guide does not implement authentication, a backend, or a new route.

The tested `login-01` source was already plain React and needed only destination adaptation. Its page and form compiled in Vite after the CLI rewrote their imports to the prepared paths.

### Catalog versus resolver metadata

`styles/radix-nova/registry.json` is the full catalog used for **discovery and selection**. The older `/r/index.json` contains only UI items and is retained in the local mirror solely for pinned-CLI resolver compatibility. It is not the full inventory and should not be used to enumerate blocks. `colors/neutral.json` remains separate CLI metadata; `styles.css` supplies the application's runtime theme.

The preparation script saves all three files. The catalog and individual payload URLs are mutable: retain the actual downloaded snapshot with the lockfile and prepared project. Downloading the catalog alone does not download its 219 payloads, transitive packages, or runtime assets.

Sources: [full Radix Nova catalog](https://ui.shadcn.com/r/styles/radix-nova/registry.json), [target placeholders and registry examples](https://ui.shadcn.com/docs/registry/examples), [item schema and dependencies](https://ui.shadcn.com/docs/registry/registry-item-json), [local JSON support](https://ui.shadcn.com/docs/changelog/2025-07-local-file-support).

## 8. Create the local JSON server

Create `src/shadcn/registry/server.mjs` with this complete content. It uses Node's standard library and needs no additional package:

```js
import { createServer } from "node:http"
import { readFile } from "node:fs/promises"
import { dirname, resolve, relative, isAbsolute } from "node:path"
import { fileURLToPath } from "node:url"

const root = dirname(fileURLToPath(import.meta.url))

createServer(async (req, res) => {
  try {
    const pathname = decodeURIComponent(new URL(req.url, "http://localhost").pathname)
    const file = resolve(root, "." + pathname)
    const localPath = relative(root, file)

    if (isAbsolute(localPath) || localPath.startsWith("..") || !file.endsWith(".json")) {
      res.writeHead(403).end("Forbidden")
      return
    }

    const content = await readFile(file)
    res.writeHead(200, { "Content-Type": "application/json" }).end(content)
    console.log(`${req.method} ${pathname} 200`)
  } catch {
    res.writeHead(404).end("JSON not found in local registry")
    console.log(`${req.method} ${req.url} 404`)
  }
}).listen(7462, "127.0.0.1", () => {
  console.log("Local shadcn registry: http://127.0.0.1:7462")
})
```

It serves only JSON from the registry directory and binds only to loopback. It is an installation aid, not an application runtime server. Keep it running only while adding components. If port 7462 is already in use, choose a free port and use the same value in `REGISTRY_URL` below.

The `REGISTRY_URL` override is implemented in the registry package's [constants](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/registry/constants.ts). The color metadata request is made in the [file updater](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/updaters/update-files.ts).

## 9. Add local npm scripts

Merge these into the existing `scripts` object in root `package.json`:

```json
{
  "scripts": {
    "shadcn": "shadcn",
    "shadcn:registry": "node src/shadcn/registry/server.mjs"
  }
}
```

Keep existing scripts. `npm run shadcn` uses the project's installed executable; it does not invoke `npx` to obtain a CLI from the internet.

## 10. Install prepared items from local JSON while offline

Open **Terminal A** at the project root and start the local metadata/component mirror:

```powershell
npm run shadcn:registry
```

Keep it open. It should print:

```text
Local shadcn registry: http://127.0.0.1:7462
```

Open **Terminal B** at the same project root and set these variables for this terminal session:

```powershell
$env:REGISTRY_URL = 'http://127.0.0.1:7462'
$env:npm_config_offline = 'true'
```

Use the base URL exactly as shown, without an additional `/r` segment. Install the **prepared** files under `registry/styles/radix-nova/`, not the original payloads under `registry/originals/`.

### UI components and blocks

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/button.json --yes
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/login-01.json --yes
```

The block's missing registry dependencies are read through the local mirror. Its prepared page and form are generated at:

```text
src/shadcn/components/blocks/login-01/page.tsx
src/shadcn/components/blocks/login-01/components/login-form.tsx
```

Primitives go under `src/shadcn/ui/`. The block does **not** create `app/login/page.tsx`, change your Vite entry point, or register a route. Import its exported page/component into your existing application deliberately, as shown in step 11.

### Hooks, utilities, reusable components, and examples

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/use-mobile.json --yes
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/utils.json --yes
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/example.json --yes
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/input-example.json --yes
```

`example.json` is available because it is a dependency of `input-example`; it is the reusable gallery wrapper, not the catalog itself. Expected locations include:

```text
src/shadcn/hooks/use-mobile.ts
src/shadcn/lib/utils.ts
src/shadcn/components/example.tsx
src/shadcn/components/examples/input-example.tsx
```

The CLI can also accept multiple local JSON paths in one `add` invocation. Review overwrite prompts and keep existing customized files unless replacement is intended. The examples deliberately omit `--overwrite`. Installing an example can install many primitives; examples are optional demonstrations, not required infrastructure for every component.

### Styles and fonts: optional, deliberate changes

The manual baseline in step 6 is already sufficient to style components. To apply the official prepared style item, inspect its CSS/package changes first, then run:

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/style.json
```

The CLI may ask for confirmation because styles can replace or modify global styling. Changes are directed to `src/shadcn/styles.css`; utility files use the configured aliases. Do not install a different design-system base as if it were a harmless component.

To apply Inter after its font package has been prepared and installed online:

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/font-inter.json
```

For the verified Vite case, this adds a local stylesheet import such as `@import "@fontsource-variable/inter";` and configures `--font-sans` plus the font class in `src/shadcn/styles.css`. The font binaries come from the installed package and Vite bundles them locally; no Google Fonts runtime request is needed for this tested item. Apply one intended font per shared font variable. Preparing 52 font descriptors does not mean all should be applied simultaneously.

### Item types supported by the schema but not present in this catalog

The current catalog contains the eight types listed in step 7. The broader registry schema also supports items such as `registry:theme`, `registry:file`, `registry:page`, `registry:item`, and `registry:base`.

- A reviewed local theme JSON can update the configured `src/shadcn/styles.css` through the same CLI command. Its npm/plugin dependencies and any URLs still need offline preparation.
- File/page entries **inside downloaded items** are assigned explicit local targets by the preparation script. Page entries become plain React component files, not framework routes. Any custom standalone file/page/item JSON also needs explicit destinations under the configured aliases or `~/src/shadcn/...`, complete source content, and locally prepared dependencies; do not blindly install its original root/config targets. This preparer intentionally automates only the official catalog's source-path format.
- `registry:base`, config changes, and environment-variable changes are outside this contained-source workflow. The preparer stops rather than allowing an entire design-system migration or root `.env`/configuration changes implicitly. Adapt such an item as a separate task. The guide does not claim it can place an entire application's configuration beneath `src/shadcn/`.

### Session handling and missing resources

Repeat the environment-variable setup in each new terminal used for offline additions. Use the local npm script, not `npx shadcn@latest`. Terminal A logs each requested JSON path; a `404` means that item/metadata file was not prepared at the requested location. Do not substitute the catalog for the missing individual payload.

`npm_config_offline=true` affects npm dependency installation; it does not itself block every possible CLI network operation. The local registry URL, local dependency closure, installed packages, and adapted runtime assets together make the selected workflow usable offline. The JSON server does not supply npm packages.

After adding items, stop Terminal A with Ctrl+C. To clear the two settings in Terminal B:

```powershell
Remove-Item Env:REGISTRY_URL -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_offline -ErrorAction SilentlyContinue
```

The application needs neither the installation server nor these variables to render installed source. Normal source changes stay under `src/shadcn/`; package manifest/lockfile changes, installed dependencies, build outputs, and the explicit integration points described at the start retain their normal project locations.

## 11. Use the generated component

Import it from its generated location in an existing application component:

```tsx
import { Button } from "@/shadcn/ui/button"

export function Example() {
  return (
    <div className="flex flex-wrap gap-3 p-6">
      <Button>Default</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="destructive">Delete</Button>
      <Button disabled>Disabled</Button>
    </div>
  )
}
```

Render the example temporarily in the application's existing UI to verify styling and disabled behavior. Do not replace the application's existing root component or add a permanent demo route unless requested. Dark tokens are used when the existing app places the `dark` class on an ancestor such as the HTML element; this guide does not add a separate theme-switching feature.

### Render a prepared block or example in Vite

After step 10, a login block can be imported as an ordinary React component:

```tsx
import LoginPage from "@/shadcn/components/blocks/login-01/page"
import InputExample from "@/shadcn/components/examples/input-example"

export function LoginPreview() {
  return <LoginPage />
}

export function InputPreview() {
  return <InputExample />
}
```

Use the relevant export in the existing application's view or existing router configuration. These are alternative examples, not instructions to add a new route or replace the root application. The login block is presentation only until you wire it to your application's authentication behavior.

Hooks and utilities use their configured paths, for example `@/shadcn/hooks/use-mobile` and `@/shadcn/lib/utils`. Do not import from `@/registry/radix-nova/...`: those are upstream source identifiers that the CLI should rewrite to the planned local destinations. A remaining unresolved upstream import is a verification failure to fix before handoff.

## 12. Prepare the offline computer

Before disconnecting/transferring, ensure the prepared project includes:

- The configuration/source changes described above.
- The full saved catalog, prepared item payloads, resolver metadata, and complete registry dependency closure for every item you intend to add; keep originals for later review.
- The installed local CLI and required npm packages, including transitive dependencies and the font packages recorded in the prepared manifest.
- The matching package manifest and lockfile, plus any runtime image/media assets saved under src/shadcn/assets/.

If npm packages are unavailable on the friend's computer, prepare the same project on an online computer with a compatible OS, CPU architecture, and Node version. Transfer the prepared project/dependency installation without overwriting the friend's unrelated application changes. Copying `node_modules` from an unrelated project or from an incompatible platform is not a reliable substitute.

An offline cache is an alternative, but it must contain the complete dependency graph for the exact lockfile. This guide's tested path uses already installed dependencies; it does not claim to provide a complete npm cache export procedure.

Run step 10 on the disconnected computer. A real disconnected run is the final check for that machine's filesystem, package installation, and environment.

## 13. Verify in the existing application

1. Confirm the pinned CLI is present: `npm run shadcn -- --version`.
2. Read the full catalog and prepared manifest, resolve applicable review notes, then start the local registry server and run the exact prepared local-file add command.
3. Confirm every prepared file has an explicit target under the configured shadcn directories; verify generated source and companion assets stay under `src/shadcn/`. In particular, installing a block must not create an `app/` page tree or modify the Vite entry point. Check that upstream registry imports were rewritten and block-relative data imports still resolve.
4. Run the project's existing TypeScript/build command, normally `npm run build`.
5. Run its existing lint/check commands where applicable; report pre-existing failures separately.
6. Start the existing dev server, normally `npm run dev`, and inspect the installed block/example in the browser. Check visible styling, variants, disabled behavior, and any expected local images/fonts. Inspect runtime requests while disconnected; a successful source installation alone does not prove runtime offline completeness.
7. Inspect the diff and ensure application code was preserved, no duplicate theme/Tailwind setup was introduced, and only necessary root integration files plus `src/shadcn/` changed.
8. Repeat the component-add operation with internet disconnected before claiming offline success for the target computer.

Common failures:

| Symptom | Check |
|---|---|
| CLI asks to initialize | Root `components.json` is missing, invalid, or the command is running from the wrong project directory. |
| Request goes to `ui.shadcn.com` | Set `REGISTRY_URL` in the terminal actually running the CLI; inspect dependencies for absolute remote URLs. |
| Local server returns 404 | Download the missing JSON at the exact path printed in the server log. |
| Connection to 127.0.0.1 is refused | Start the server and confirm the port matches `REGISTRY_URL`. |
| npm reports an offline/cache error | A required npm dependency is not installed or cached. Prepare it online; do not silently fall back to online installation. |
| Component imports fail | Confirm both TypeScript and Vite resolve `@/`, and `components.json` aliases point to the intended directories. |
| Block produces app/ pages or unresolved registry imports | Confirm you installed the prepared copy, not the original payload. Review its explicit targets and CLI import rewriting. |
| Font or image is missing offline | Check the prepared font package and local asset imports, not just the descriptor JSON. |
| Component is unstyled | Confirm the global stylesheet is imported, the theme mappings are present, Tailwind v4 is processing the CSS, and existing app CSS is not overriding the result. |
| CLI reports an invalid registry item | Check that the file contains the actual raw JSON payload rather than HTML, escaped markdown, or a copied text preview. |

## Verification performed for this guide

Updated on 2026-10-29. Context7 and Tavily supplied current official registry schema, target-placeholder, font-updater, and destination-resolution documentation/source. Exact catalog and payload JSON were downloaded directly from the official registry for executable checks. The catalog contained 219 items across the eight types listed in step 7, including 30 blocks; these counts describe the checked snapshot, not a permanent guarantee.

A disposable project, separate from the user's existing application, used Node 24.19.0, React 19.3.1, Vite 8.3.5, TypeScript 5.9.3, Tailwind 4.3.3, and shadcn 4.23.0.

- **VERIFIED:** The original local Button install attempted a color-metadata network request; the local metadata mirror resolves that requirement.
- **VERIFIED:** Local Button and Alert Dialog installs, including registry dependency resolution, with a Node socket guard rejecting non-loopback connections.
- **VERIFIED:** The embedded preparation script fetched selected items recursively, preserved original payloads, assigned explicit local targets, and recorded packages plus font dependencies in its manifest.
- **VERIFIED:** A multi-item CLI install of prepared Login 01, use-mobile, input-example, font-inter, and style JSON, with the external socket guard active. The dependency closure also exercised registry:ui, registry:lib, and registry:component items, covering all eight types currently present in the catalog.
- **VERIFIED:** The login page and form were generated under src/shadcn/components/blocks/login-01/, the example under src/shadcn/components/examples/, hooks under src/shadcn/hooks/, and libraries/UI under their configured directories. The login page's import was rewritten to its local block companion component. No app/ route tree was created.
- **VERIFIED:** Vite font handling added a local @fontsource-variable/inter import and font configuration to src/shadcn/styles.css; the production build emitted local WOFF2 font assets.
- **VERIFIED:** TypeScript checking and a Vite production build passed after the representative item install, with the login block and input example included in the application bundle.
- **NOT RUN:** The friend's actual disconnected machine, browser visual/interaction verification, full --all preparation, every catalog item, all external/runtime URLs or asset types, an offline npm cache reconstruction, and custom standalone base/theme/file/page/item schemas outside the current catalog. The document provides handling/review instructions for those schema cases; it does not label them all executed.

The socket guard was a verification aid, not full operating-system network isolation. These checks establish the selected CLI flows and representative Vite builds, not a universal framework converter or a blanket guarantee for every registry item. The temporary npm install's audit summary reported three low-severity findings; this task did not perform dependency security remediation.

## Primary references

- [Full Radix Nova registry catalog, including blocks](https://ui.shadcn.com/r/styles/radix-nova/registry.json)
- [Registry target placeholders and examples](https://ui.shadcn.com/docs/registry/examples)
- [Target destination resolver implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/resolve-file-path.ts)
- [Vite/local font handling implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/updaters/update-fonts.ts)

- [Manual installation](https://ui.shadcn.com/docs/installation/manual)
- [Vite integration](https://ui.shadcn.com/docs/installation/vite)
- [components.json configuration and aliases](https://ui.shadcn.com/docs/components-json)
- [Local JSON file support](https://ui.shadcn.com/docs/changelog/2025-07-local-file-support)
- [Registry item schema and dependencies](https://ui.shadcn.com/docs/registry/registry-item-json)
- [Live Radix Nova Button JSON](https://ui.shadcn.com/r/styles/radix-nova/button.json)
- [Live Radix Nova Alert Dialog JSON](https://ui.shadcn.com/r/styles/radix-nova/alert-dialog.json)
- [Registry URL override implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/registry/constants.ts)
- [File updater implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/updaters/update-files.ts)
