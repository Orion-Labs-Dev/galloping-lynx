# Athena Loader — Extensions

Extensions add custom UI and behavior to Athena Loader without touching its
source. They're the supported way for trusted authors to ship their own tabs,
helpers, and automations on top of the loader's runtime.

---

## ⚠ Read this first

Extensions are **JavaScript that runs inside Athena Loader with the same
privileges the app itself has.** That means a malicious or buggy extension
can read your prefs, hit the network, and access anything the loader's
renderer can. Treat them like browser extensions or VS Code extensions —
**only install ones you trust the author of.**

The loader pops a confirmation dialog before any first-time install so you
never end up with custom code running silently.

---

## Installing an extension

1. Open the **Extensions** tab in the left sidebar.
2. Click **Install .aextension**.
3. Read the warning, click *I understand — pick file*.
4. Choose the `.aextension` file from disk.
5. The extension appears in the list, enabled by default. Its tab (if any)
   shows up in the sidebar under an "Extensions" group.

To uninstall, hit the trash icon on its card. To temporarily disable, flip
the toggle off — its files stay on disk but it won't run on next launch.

Click an extension's **Open folder** button to see where it lives on disk.

---

## Building your own

### 1. Turn on Builder mode

Click your avatar (bottom-left) → **Settings** → toggle **Builder mode**
on. The Extensions tab now shows a *Builder mode* panel with **Create new
extension** and **Package folder…** buttons.

### 2. Create the scaffold

Click **Create new extension**. Fill in:

| Field         | Notes                                                                 |
| ------------- | --------------------------------------------------------------------- |
| Display name  | Shown in the loader UI                                                |
| Id            | Lowercase letters / digits / `-` / `_`. 2–64 chars. Auto-derived from name. |
| Author        | Your name / handle                                                    |
| Version       | Semver-ish; must start with a digit (e.g. `0.1.0`)                    |
| Description   | Optional one-liner                                                    |

The scaffold lands at `%APPDATA%\AthenaLoader\extension-builder\<id>\` and
contains:

- `manifest.json` — your extension's metadata
- `index.js` — the entry script (a working starter that adds a tab and a
  toast button so you can sanity-check the flow immediately)

### 3. Iterate

Edit `index.js` in your editor of choice. The renderer reloads the entry
on every install, so the dev loop is:

1. Edit `index.js`
2. **Package folder…** → save somewhere as `<id>.aextension`
3. **Install .aextension** → pick the file you just saved
4. Watch it run

(Auto-reload-on-save is on the roadmap; today every change needs a
re-package + re-install.)

### 4. Package + share

When you're happy:

1. Click **Package folder…** in the Extensions tab.
2. Pick *any* file inside the source folder (the loader treats the parent
   directory as the source).
3. Pick the output path. By convention `<id>.aextension`.
4. Send the resulting file to anyone — they install it the same way you did.

Inside, `.aextension` is just a zip. Anyone can unzip it to read the
contents.

---

## Manifest format

```json
{
  "id":          "my-extension",
  "name":        "My Extension",
  "author":      "Your Name",
  "version":     "0.1.0",
  "entry":       "index.js",
  "description": "Optional one-liner shown in the Extensions tab."
}
```

Validation rules the loader enforces at install time:

- `id` is required, must match `/^[a-z0-9][a-z0-9_-]{1,63}$/`. The folder
  in `%APPDATA%\AthenaLoader\extensions\` uses this id verbatim.
- `name`, `author`, `version`, `entry` are required and non-empty.
- `entry` must point to a file that actually exists in the archive.

---

## The `loaderApi`

Every extension's entry script is invoked CommonJS-style:

```js
module.exports = function init(loaderApi) {
  // your code here
}
```

`loaderApi` is a frozen object. The full surface today:

### `loaderApi.log(level, message)`

Drop a line into the loader's own log feed. Visible in the **Loader Log**
sidebar tab.

```js
loaderApi.log('info',    'Hello from my extension')
loaderApi.log('warn',    'Something unusual happened')
loaderApi.log('error',   'Bad thing')
loaderApi.log('success', 'Did the thing')
```

### `loaderApi.toast(level, message)`

Bottom-right toast notification. Auto-dismisses after a few seconds.

```js
loaderApi.toast('ok',    'Saved.')
loaderApi.toast('error', 'Couldn’t reach the server.')
loaderApi.toast('info',  'Heads up: …')
```

### `loaderApi.alert({ title, body?, danger? }) → Promise<boolean>`

Themed confirmation dialog. Resolves `true` on the OK button, `false` on
Cancel / Esc / backdrop click.

```js
const ok = await loaderApi.alert({
  title:  'Wipe cache?',
  body:   'This deletes everything in the cache folder. It cannot be undone.',
  danger: true
})
if (ok) doTheThing()
```

### `loaderApi.addTab({ id, label, icon?, render(rootEl) })`

Add a new top-level tab to the sidebar. The tab shows up under the
"Extensions" group.

| Param  | Type     | Notes                                                                            |
| ------ | -------- | -------------------------------------------------------------------------------- |
| `id`   | string   | Unique within your extension. Re-using the same id replaces the previous tab.    |
| `label`| string   | Shown in the sidebar                                                             |
| `icon` | string?  | A FontAwesome class (e.g. `fa-solid fa-rocket`). Defaults to `fa-puzzle-piece`. |
| `render(root)` | function | Receives an empty `HTMLDivElement`. Set `innerHTML` or imperatively build DOM. |

```js
loaderApi.addTab({
  id: 'cool-tools',
  label: 'Cool Tools',
  icon: 'fa-solid fa-wrench',
  render(root) {
    root.innerHTML = `
      <div class="p-6">
        <h1 class="text-2xl font-black tracking-tighter mb-3">Cool Tools</h1>
        <button class="btn btn-primary" id="ping">Ping</button>
      </div>`
    root.querySelector('#ping')
        .addEventListener('click', () => loaderApi.toast('ok', 'Pong'))
  }
})
```

### `loaderApi.removeTab(id)`

Remove a tab you previously added. Useful when the user toggles a feature
inside your extension.

```js
loaderApi.removeTab('cool-tools')
```

---

## Styling — use the loader's theme

The loader ships a Tailwind v4 + custom CSS theme. Your tab's content has
the same classes available, so anything you build can match the rest of
the app for free. The most useful ones:

| Class              | What it gives you                                            |
| ------------------ | ------------------------------------------------------------ |
| `.card`            | Padded card with the dark background + border-radius         |
| `.card-soft`       | A flatter variant with a quieter background                  |
| `.snap-input`      | The yellow-focus text input field                            |
| `.btn .btn-primary`| Yellow primary button                                        |
| `.btn .btn-secondary` | Gray secondary button                                     |
| `.btn .btn-danger` | Red danger button                                            |
| `.btn .btn-ghost`  | Borderless ghost button                                      |
| `.tab-row` / `.tab` | Tab strip that matches the sub-tab look                     |
| `.tile` / `.tile-label` / `.tile-value` | Stat tiles                              |
| `.badge .badge-good` / `.badge-warn` / `.badge-bad` / `.badge-dim` | Pill labels |
| `.h-section`       | Yellow section heading                                       |
| `.hr`              | Inline-block 1px divider                                     |
| `.log-line` / `.log-info` / `.log-warn` / `.log-error` | Mono log row colors      |
| `.anim-fade-in` / `.anim-pop-in` | Reusable entry animations                       |

CSS variables you can pull into inline styles:

```
--color-bg, --color-card, --color-card-soft, --color-border, --color-border-hot,
--color-accent (yellow), --color-text, --color-text-dim, --color-text-muted,
--color-good, --color-warn, --color-bad, --color-info,
--font-sans
```

Example using both:

```js
root.innerHTML = `
  <div class="p-6">
    <h1 class="text-2xl font-black tracking-tighter">Stats</h1>
    <div class="hr" />
    <div class="flex gap-3">
      <div class="tile">
        <div class="tile-label">Players</div>
        <div class="tile-value" style="color: var(--color-accent)">42</div>
      </div>
      <div class="tile">
        <div class="tile-label">Uptime</div>
        <div class="tile-value">3d 7h</div>
      </div>
    </div>
  </div>`
```

---

## Examples

### Example 1 — A button that hits an API

```js
module.exports = function init(api) {
  api.addTab({
    id: 'weather',
    label: 'Weather',
    icon: 'fa-solid fa-cloud-sun',
    render(root) {
      root.innerHTML = `
        <div class="p-6">
          <button class="btn btn-primary" id="fetch">Fetch weather</button>
          <div id="out" class="card mt-3" style="display:none"></div>
        </div>`
      const out = root.querySelector('#out')
      root.querySelector('#fetch').addEventListener('click', async () => {
        api.log('info', 'fetching...')
        try {
          const res = await fetch('https://wttr.in/?format=%t')
          const txt = await res.text()
          out.style.display = 'block'
          out.textContent = `It is ${txt}`
          api.toast('ok', 'Weather fetched.')
        } catch (e) {
          api.toast('error', e.message)
        }
      })
    }
  })
}
```

### Example 2 — A confirmation flow

```js
module.exports = function init(api) {
  api.addTab({
    id: 'admin-tools',
    label: 'Admin tools',
    icon: 'fa-solid fa-shield',
    render(root) {
      root.innerHTML = `
        <div class="p-6 max-w-2xl">
          <div class="card">
            <div class="h-section mb-2">Danger zone</div>
            <p class="text-[var(--color-text-dim)] text-sm">
              Use this if the server has been misbehaving for more than five minutes.
            </p>
            <button class="btn btn-danger mt-3" id="kick-everyone">Kick everyone</button>
          </div>
        </div>`
      root.querySelector('#kick-everyone').addEventListener('click', async () => {
        const ok = await api.alert({
          title: 'Kick all players?',
          body: 'They get a generic disconnect message. They can rejoin immediately.',
          danger: true
        })
        if (!ok) return
        // …do the work…
        api.toast('ok', 'Done.')
      })
    }
  })
}
```

### Example 3 — Polling and removing a tab

```js
module.exports = function init(api) {
  let polling = null

  api.addTab({
    id: 'live-ticker',
    label: 'Live ticker',
    icon: 'fa-solid fa-tower-broadcast',
    render(root) {
      root.innerHTML = `
        <div class="p-6">
          <button class="btn btn-primary" id="start">Start polling</button>
          <button class="btn btn-secondary" id="stop">Stop</button>
          <div id="lines" class="card-soft mt-3" style="height: 240px; overflow:auto"></div>
        </div>`
      const lines = root.querySelector('#lines')
      root.querySelector('#start').addEventListener('click', () => {
        if (polling) return
        polling = setInterval(() => {
          const div = document.createElement('div')
          div.className = 'log-line log-info'
          div.textContent = new Date().toLocaleTimeString()
          lines.appendChild(div)
          lines.scrollTop = lines.scrollHeight
        }, 1000)
      })
      root.querySelector('#stop').addEventListener('click', () => {
        if (polling) { clearInterval(polling); polling = null }
        api.toast('info', 'Stopped.')
      })
    }
  })
}
```

---

## Where things live on disk

| Path                                                                | Purpose                                       |
| ------------------------------------------------------------------- | --------------------------------------------- |
| `%APPDATA%\AthenaLoader\extensions.json`                            | Registry of installed extensions              |
| `%APPDATA%\AthenaLoader\extensions\<id>\`                           | Unzipped contents of each installed extension |
| `%APPDATA%\AthenaLoader\extension-builder\<id>\`                    | Builder-mode scaffolds before packaging       |

You can open any of these from inside the loader: Settings shows shortcuts
in the Debug panel for **Open user data folder**, and the Extensions list
exposes **Open folder** per extension.

---

## Troubleshooting

**My extension installed but its tab isn't there.**
The tab is registered when the entry script runs. Open the **Loader Log**
tab — if the entry threw, you'll see a `[ext:<id>]` error toast and a
matching log line. Re-package after fixing.

**Toast says "Extension entry not found or disabled."**
The toggle on the card is off, or you removed the underlying folder
manually. Toggle it back on, or reinstall the `.aextension`.

**`alert()` never resolves.**
You forgot to `await` the call. It returns a Promise; treat it like any
other async function.

**Builder mode toggles don't appear in the Extensions tab.**
You turned on Builder mode but didn't reload — the panel reads the
current pref on mount. Switch tabs and back, or close+reopen the loader.

**The extension throws because of `require('fs')` etc.**
Extensions run inside the renderer, not Node. Browser-style APIs only
(`fetch`, `localStorage`, the DOM, …). `loaderApi` is your bridge to the
loader's privileged actions — request more there if something's missing.

---

## What's stable, what's not

The `loaderApi` surface listed above is the supported API today. The
following are planned but **not yet** present:

- `loaderApi.addSubTab(parentId, …)` — sub-tabs under built-in pages
- `loaderApi.addButton({ where, … })` — injection into specific toolbar
  slots (Control toolbar, server-detail header, etc.)
- `loaderApi.servers.list()` / `.start()` / `.stop()` — typed server
  control surface
- `loaderApi.events.on('server-started' | 'mod-injected' | …)` — event
  subscriptions

Until these land, lean on `addTab` + `fetch` for everything.

---

If you build something cool, tell us. We're happy to feature good
extensions in-app once that piece of the docs ships.
