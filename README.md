# desktop-starter-app (EsnaadM desktop shell)

This repository is the starting point for "EsnaadM", a Windows desktop application from 2019 that packages an Adobe Flex (Flash) user interface inside Electron so it can run offline, without a browser plug-in or a central server. `main.js` starts a local Express server on port 3000 (or 3001 if that port is busy) and opens a frameless Electron window whose `<webview>` loads the compiled `esnaadm.swf` through the bundled Pepper Flash plug-in. The Flex app talks to that local server over HTTP to read and write a single JSON "catalog" file on disk, which acts as its database, and to run a query and save API over it. The same server can also act as a distribution point: one machine uploads a zipped build, and other installations download the latest version and unpack it. The Flex source in `project/` contains a reusable component and skin library, login and first-run setup pages, and admin modules for the database, queries and encryption. Packaging uses electron-builder to produce NSIS installers for 32- and 64-bit Windows. It is an unmaintained prototype that depends on Flash, which is no longer supported.

> Prototype, built in September 2019. Not actively maintained. Adobe Flash Player reached end of life in December 2020, so the UI will not run on current systems without a Flash-capable Electron build.

## Features

- Frameless Electron window with custom minimize, maximize and close buttons and a draggable header
- Loads the Flex UI (`esnaadm.swf`) through the bundled Pepper Flash plug-in (`pepflashplayer-32.dll` / `pepflashplayer-64.dll`)
- Local Express API used by the Flex app:
  - `POST /query`: evaluates a JavaScript expression against the catalog, with helpers such as `filterBy`, `to`, `except`, `by`, `flat` and `describe`
  - `POST /save`: updates or appends data at a dotted path in the catalog; `POST /commit` writes the catalog to disk
  - `POST /saveDb`, `POST /deleteDb` (moves the current database folder to a timestamped backup)
  - `POST /encrypt`: AES-CBC encryption followed by an MD5 digest, keyed by a caller-supplied salt
  - `POST /browser`: forwards a function call from the Flex app to the Electron page
- Distribution and update server: `GET /distribute` (upload page), `POST /distribute` (upload a zip), `GET /distribute/latest`, `GET /distribute/download`, `GET /distribute/downloadDb`, `POST /distribute/uploadDb`, and `POST /auto-update` to fetch and unzip the latest build from another machine
- On first run, copies the Flex build into `C:\Users\Public\AppData\Local\esm`; data lives in `esm\active\`
- Optional AES encryption of the database file (off by default: `encryptDb = false`)
- Starting a second instance shuts down the one already running on the other port

## Tech stack

Electron 4 · Node.js · Express · multer · crypto-js · Adobe Flex / ActionScript 3 (MXML) · Pepper Flash · electron-builder (NSIS)

## Getting started

Prerequisites: Windows (the data paths and Flash plug-ins are Windows-specific), Node.js and npm, and Adobe Flash Builder or the Flex SDK to compile the UI.

1. Compile the Flex project in `project/` (main application `app.mxml`). Its output folder is `project/bin-debug/`, which is ignored by git and must contain `esnaadm.swf`.
2. Install and run:

```bash
npm install
npm start          # electron .
npm run pack       # electron-builder --dir (unpacked build in dist/)
npm run dist       # NSIS installers for x64 and ia32
```

electron-builder copies the Flash DLLs and `project/bin-debug/` into the app's resources. On first launch the app copies them to `C:\Users\Public\AppData\Local\esm\app\`. When running from source with `npm start`, the app looks for a previous build in `dist/win-unpacked/resources/app`, so run `npm run pack` first.

## Project structure

```text
main.js              Electron main process, local Express API, distribution/update server
index.html           frameless window with a <webview> that hosts the Flex app
distribute.html      upload form for the distribution point
pepflashplayer-*.dll Pepper Flash plug-ins (32- and 64-bit)
build/icon.png       application icon
project/             Flash Builder project for the Flex UI
  src/app.mxml       application shell: header, login, setup, router
  src/lib/component/ custom Spark components and skins (grid, lookup, date/time fields, ...)
  src/lib/page/      login, setup, navigator, router, dialogs, notifications
  src/lib/utils/     promises, fetch, binding helpers, date utilities, crypto library
  src/modules/       home page and config modules (database, query, encrypt)
```

## Limitations

- Requires Flash, which is end-of-life, and Electron 4, which is long out of support.
- `POST /query` runs the request body as JavaScript in the main process (`new Function`), and the server accepts requests from any local process, so any program on the machine can execute code through it.
- Certificate errors are accepted for every URL (`certificate-error` handler).
- The database encryption key is hard-coded in `main.js`, and encryption is disabled by default.
- The auto-updater downloads over plain HTTP without verifying the package.
- Only Windows is supported in practice; Linux and macOS plug-in names are referenced but the plug-ins are not included.
