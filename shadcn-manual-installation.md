# Manual shadcn installation with local component JSON files

Verified on 2026-10-29. Target: an **existing React + Vite + TypeScript + Tailwind CSS v4 project**, using **Radix UI + Nova + Neutral**. Commands below use npm and Windows PowerShell.

## Instructions for the implementing agent

Implement the setup below in the existing project. Inspect its package manager, configuration, and applicable AGENTS.md first. Preserve existing application code, dependencies, Vite plugins, TypeScript options, and unrelated styles. Merge the configuration snippets; do not replace whole existing configuration files. Do not scaffold a new application, run `shadcn init`, create a worktree, or commit/push application changes unless separately instructed.

Keep shadcn source, generated components, helpers, styles, downloaded registry JSON, and the local registry server under `src/shadcn/`. Necessary integration points remain outside that directory: root `components.json`, package manifest/lockfile, TypeScript/Vite alias configuration, and the import in the application's existing global stylesheet. Dependencies remain in the project's normal `node_modules/`.

This guide is intended to be copied in full. All code for the manual setup is included below. The online preparation stage downloads the official component payloads and installs npm packages; those third-party artifacts must be available before using the setup offline.

## Why the local metadata server is included

The CLI supports `shadcn add ./component.json`, but **local component JSON is not a complete offline mode**. In the verified npm release, `shadcn@4.23.0`, adding the official local Nova button still requests `colors/neutral.json` while transforming the source. Registry dependencies can also cause requests for other component JSON files.

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
        ├── components/              # Other generated components/blocks
        ├── lib/
        │   └── utils.ts
        ├── hooks/
        └── registry/
            ├── server.mjs
            ├── index.json           # Downloaded official registry index
            ├── colors/
            │   └── neutral.json     # Downloaded official color metadata
            └── styles/
                └── radix-nova/
                    ├── button.json
                    └── alert-dialog.json
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

## 7. Download the JSON payloads during online preparation

Run the following from the project root while connected. It downloads official JSON directly, validates that each response is JSON, and saves it without copying rendered website text:

```powershell
$ErrorActionPreference = 'Stop'

$downloads = @{
    'index.json' = 'https://ui.shadcn.com/r/index.json'
    'colors/neutral.json' = 'https://ui.shadcn.com/r/colors/neutral.json'
    'styles/radix-nova/button.json' = 'https://ui.shadcn.com/r/styles/radix-nova/button.json'
    'styles/radix-nova/alert-dialog.json' = 'https://ui.shadcn.com/r/styles/radix-nova/alert-dialog.json'
}

foreach ($entry in $downloads.GetEnumerator()) {
    $destination = Join-Path 'src/shadcn/registry' $entry.Key
    New-Item -ItemType Directory -Force -Path (Split-Path -Parent $destination) | Out-Null
    Invoke-WebRequest -Uri $entry.Value -OutFile $destination
    Get-Content -LiteralPath $destination -Raw | ConvertFrom-Json | Out-Null
}
```

Direct links for inspection:

- [Button JSON](https://ui.shadcn.com/r/styles/radix-nova/button.json)
- [Alert Dialog JSON](https://ui.shadcn.com/r/styles/radix-nova/alert-dialog.json)
- [Neutral color metadata](https://ui.shadcn.com/r/colors/neutral.json)
- [Official registry index](https://ui.shadcn.com/r/index.json)

The metadata is separate from `styles.css`: the stylesheet supplies the runtime theme, while the CLI reads the color JSON during installation. The index is included for resolver paths that need registry index information; the verified button and alert-dialog checks did not request it.

The URLs are mutable. Save the downloaded files with the prepared project so the offline computer uses the same payloads you checked online. To replace or update an existing snapshot later, keep that update deliberate.

For any additional official component:

1. Download `https://ui.shadcn.com/r/styles/radix-nova/NAME.json` into `src/shadcn/registry/styles/radix-nova/NAME.json`.
2. Read its `registryDependencies`. Download each named component the same way, including their own registry dependencies recursively. For example, `alert-dialog.json` references `button`.
3. Read `dependencies` and `devDependencies` and install the additional npm packages while connected. Components such as charts, carousels, calendars, and forms can require additional libraries. Keep the foundation packages from step 1 even when they are not repeated in a component's dependency list.
4. For a URL, GitHub address, namespaced entry, or local path inside `registryDependencies`, prepare that dependency explicitly. `REGISTRY_URL` redirects built-in registry lookups, not arbitrary external URLs. If you rewrite a dependency to a local path, test that path from the project root with the pinned CLI.
5. Exercise the exact add command before transferring the snapshot. A local server `404` identifies a missing mirrored JSON file; an npm offline-cache error identifies a package dependency that was not prepared.

This guide does not claim that downloading only the two example components makes every shadcn component available offline.

Sources: [local JSON file support](https://ui.shadcn.com/docs/changelog/2025-07-local-file-support), [registry item dependency format](https://ui.shadcn.com/docs/registry/registry-item-json).

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

## 10. Add components using the CLI and local JSON

Open **Terminal A** at the project root and start the local metadata/component mirror:

```powershell
npm run shadcn:registry
```

Keep it open. It should print:

```text
Local shadcn registry: http://127.0.0.1:7462
```

Open **Terminal B** at the same project root. Set these variables for this terminal session:

```powershell
$env:REGISTRY_URL = 'http://127.0.0.1:7462'
$env:npm_config_offline = 'true'
```

Use the base URL exactly as shown, without an additional `/r` segment, because the server's root already corresponds to the downloaded registry root.

Add the Button from its local JSON:

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/button.json --yes
```

Expected generated file:

```text
src/shadcn/ui/button.tsx
```

Terminal A should show a successful request for `/colors/neutral.json`. The CLI writes the component; you do not extract or paste its `files[].content` manually.

To add Alert Dialog:

```powershell
npm run shadcn -- add ./src/shadcn/registry/styles/radix-nova/alert-dialog.json --yes
```

The CLI can resolve the `button` registry dependency through the local mirror and install the required source files. If Button already exists, review any overwrite prompt and keep your existing customized file unless replacement is intended. The examples deliberately omit `--overwrite`.

Repeat the environment-variable setup in each new terminal used for offline additions. Do not use `npx shadcn@latest` or a bare `add button` in place of these local-file commands when following this workflow.

`npm_config_offline=true` affects npm dependency installation; it does not itself block every possible network operation in the CLI. The local registry URL and fully prepared dependency files are what keep built-in registry requests local. A missing npm package still needs a prepared cache/package transfer; the JSON server does not supply it.

After adding components, stop Terminal A with Ctrl+C. To clear the two settings in Terminal B when finished:

```powershell
Remove-Item Env:REGISTRY_URL -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_offline -ErrorAction SilentlyContinue
```

The application needs neither the local server nor these environment variables to render installed components.

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

## 12. Prepare the offline computer

Before disconnecting/transferring, ensure the prepared project includes:

- The configuration/source changes described above.
- The downloaded registry JSON, including metadata and the dependency closure for every component you intend to add.
- The installed local CLI and required npm packages, including transitive dependencies.
- The matching package manifest and lockfile.

If npm packages are unavailable on the friend's computer, prepare the same project on an online computer with a compatible OS, CPU architecture, and Node version. Transfer the prepared project/dependency installation without overwriting the friend's unrelated application changes. Copying `node_modules` from an unrelated project or from an incompatible platform is not a reliable substitute.

An offline cache is an alternative, but it must contain the complete dependency graph for the exact lockfile. This guide's tested path uses already installed dependencies; it does not claim to provide a complete npm cache export procedure.

Run step 10 on the disconnected computer. A real disconnected run is the final check for that machine's filesystem, package installation, and environment.

## 13. Verify in the existing application

1. Confirm the pinned CLI is present: `npm run shadcn -- --version`.
2. Start the local registry server and run the exact local-file add command.
3. Confirm source files are generated under `src/shadcn/` and imports resolve correctly.
4. Run the project's existing TypeScript/build command, normally `npm run build`.
5. Run its existing lint/check commands where applicable; report pre-existing failures separately.
6. Start the existing dev server, normally `npm run dev`, and inspect the example in the browser. Check visible styling, variants, and disabled behavior.
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
| Component is unstyled | Confirm the global stylesheet is imported, the theme mappings are present, Tailwind v4 is processing the CSS, and existing app CSS is not overriding the result. |
| CLI reports an invalid registry item | Check that the file contains the actual raw JSON payload rather than HTML, escaped markdown, or a copied text preview. |

## Verification performed for this guide

The guide was researched using Context7 and the official shadcn pages/live JSON through Tavily. An additional Tavily source lookup was rate-limited; the manual/Vite/configuration pages and live button payload had already been retrieved. Exact JSON artifacts were also downloaded directly from the official registry for executable verification.

A disposable project, separate from the user's existing application, used Node 24.19.0, React 19.3.1, Vite 8.3.5, TypeScript 5.9.3, Tailwind 4.3.3, and shadcn 4.23.0.

- **VERIFIED:** Adding the official local Button JSON with external network sockets denied failed when the CLI attempted its color metadata request. This established the missing offline requirement.
- **VERIFIED:** With the local mirror, adding that local JSON created `src/shadcn/ui/button.tsx` while a Node socket guard rejected non-loopback connections.
- **VERIFIED:** Starting without a generated Button, adding local Alert Dialog JSON resolved the Button dependency through the mirror and created both files under `src/shadcn/ui/` with the same guard active.
- **VERIFIED:** TypeScript checking and a Vite production build passed after both files were generated.
- **NOT RUN:** The friend's actual disconnected computer, browser-based visual/interaction verification, every possible shadcn component, and an offline npm cache reconstruction.

The network guard was a verification aid, not a full operating-system network isolation system. These results establish the tested CLI flows, not a blanket guarantee for every registry item or machine. The temporary package install's npm audit summary reported three low-severity findings; the verification did not include a dependency security remediation or security audit.

## Primary references

- [Manual installation](https://ui.shadcn.com/docs/installation/manual)
- [Vite integration](https://ui.shadcn.com/docs/installation/vite)
- [components.json configuration and aliases](https://ui.shadcn.com/docs/components-json)
- [Local JSON file support](https://ui.shadcn.com/docs/changelog/2025-07-local-file-support)
- [Registry item schema and dependencies](https://ui.shadcn.com/docs/registry/registry-item-json)
- [Live Radix Nova Button JSON](https://ui.shadcn.com/r/styles/radix-nova/button.json)
- [Live Radix Nova Alert Dialog JSON](https://ui.shadcn.com/r/styles/radix-nova/alert-dialog.json)
- [Registry URL override implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/registry/constants.ts)
- [File updater implementation](https://github.com/shadcn-ui/ui/blob/main/packages/registry/src/updaters/update-files.ts)
