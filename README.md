# codex-web

a browser frontend for codex desktop, running on a machine you control.

https://github.com/user-attachments/assets/0a33cbd8-741c-412c-9e75-46dfe9324596

This fork extends [0xcaff/codex-web](https://github.com/0xcaff/codex-web) with
persistent shared Linux sessions, Desktop tool compatibility, and configurable
LAN access. [中文安装与使用说明](docs/shared-sessions.zh-CN.md).

## motivation

the agents were never meant to stay trapped in a terminal window for long.
codex desktop brought the power of agents to your local computer, where your
files, credentials, and tools already live.

codex-web brings codex desktop to the browser while keeping the backend on a
machine you control (a linux box in the cloud, your home lab, or a desktop / mac
mini). agents keep running after your laptop closes. you can reconnect from any
device with a browser.

this project aims to be as thin a wrapper as possible to ensure upstream changes
to the codex desktop app can be integrated quickly.

## usage

`codex-web` serves the browser client and hosts the desktop-side bridge. by
default, it listens on `127.0.0.1:8214`.

it will use `codex` from `PATH` if available, or `CODEX_CLI_PATH` if you set
it.

run it with `npx`:

```bash
npx --yes github:qq1018408006/codex-web
```

or with nix:

```bash
nix run github:qq1018408006/codex-web
```

then open <http://127.0.0.1:8214> in a browser.

### sign in

ensure the codex cli on the host machine is signed in before starting the
server.

```bash
codex login --device-auth
```

### shared Linux sessions and LAN setup

Web and the `codex-shared` CLI can connect to the same long-lived app-server.
Restarting Web leaves that server and its sessions running. The installer creates
user services and a versioned snapshot; it retains the original `codex` command
and does not edit the global Codex configuration.

Requirements: Linux with `systemd --user`, Python 3, Node 22.12+, Codex CLI
0.160.1+, and `websocat` on `PATH`. Sign in to Codex first, then build and install
from a checkout:

```bash
git clone https://github.com/qq1018408006/codex-web.git
cd codex-web
npm ci
python3 scripts/install-services.py --access local --restart
```

To allow a phone on the same trusted LAN, choose `--access lan` during installation:

```bash
python3 scripts/install-services.py --access lan --port 8214 --restart
```

Open `http://<computer-LAN-IP>:8214` on the phone. The Web server has no built-in
authentication; read the security section below before enabling LAN access.
The saved host and port are retained on subsequent installs when those flags
are omitted. `--host <IP>` selects a specific interface; `--dry-run` previews the
configuration without changing services.

Use `~/.local/bin/codex-shared` for a shared interactive session, or
`~/.local/bin/codex-shared resume <session-id>` to reopen one. Use the original
`codex` for login, plugins, and other standalone commands. Ordinary Desktop and
VS Code sessions remain independent unless explicitly attached to this server.

Desktop tools use the user's existing official `codex-app-tools` 0.1.5 plugin,
loaded from its installed cache. Supply another installation location with
`--app-tools /path/to/server.mjs`. Without it, shared sessions still work and the
orphan Desktop-only tool whitelist is removed locally. The plugin is not bundled
in this repository.

Already loaded CLI threads may retain their original tool configuration. Finish
the current turn, close its clients, and reopen it in Web, or create a new Web
thread to obtain Desktop tools. Those tools require an open, initialized Web tab.
This compatibility layer does not provide the unfinished browser/computer-use
features listed in the roadmap.

For a manually managed shared server:

```bash
node src/server/main.js --shared-socket /path/to/app.sock --host 127.0.0.1
```

The default mode still starts its own Codex process. Plain HTTP LAN pages and
runtime workers receive a UUID fallback backed by `crypto.getRandomValues`,
avoiding the blank page caused by missing `crypto.randomUUID` outside a secure
context. Browser features that require HTTPS still require HTTPS.

Run `npm test` for compatibility, installer, and supervisor checks.
`npm run test:install` exercises first installation and upgrades in a temporary
home with service commands intercepted. After a build,
`npm run test:integration` uses an isolated app-server and mobile-sized browser to
check real Desktop tool calls across a Web restart. It requires Google Chrome,
the official plugin, and signed-in Codex, and consumes some model quota. Set
`CODEX_TEST_CODEX`, `CODEX_TEST_BROWSER`, `CODEX_TEST_APP_TOOLS`,
`CODEX_TEST_HOST`, or `CODEX_TEST_PORT` to override test dependencies and address.
The integration test uses port 8220 by default and never restarts user services.

### proxying to app-server (advanced usage)

it’s often useful to run the app server separately, so a crash or restart of
codex-web doesn’t interrupt the codex process executing commands.

it's possible to hook codex-web up to an already-running app server using the
`codex_remote_proxy` script.

start a long-lived app server somewhere:

```bash
mkdir -p /tmp/codex-app-server
cd /tmp/codex-app-server
codex app-server --listen unix://codex-app-server.sock
```

then run `codex-web` with the proxy helper:

```bash
nix shell github:0xcaff/codex-web github:0xcaff/codex-web#codex_remote_proxy -c bash -lc '
  export CODEX_UNIX_SOCKET=/tmp/codex-app-server/codex-app-server.sock
  export CODEX_CLI_PATH="$(command -v codex_remote_proxy)"
  codex-web
'
```

`codex app-server proxy --sock ...` is a raw stdio protocol bridge for another
program to use; when run directly in a terminal it will wait for protocol input
rather than opening an interactive prompt.

## security

run `codex-web` only on trusted networks. treat anyone who can reach the
`codex-web` server as someone who can operate codex on the host machine as the
same user running the server.

if you need authn or authz, implement it outside of `codex-web`: proxy it through
wireguard, tailscale, or an ssh tunnel and put an authentication gateway or
reverse proxy in front.

someone with access to the web ui may be able to:

- run commands on the host, limited only by the permissions of the `codex-web`
  server process.
- read or modify files, environment variables, credentials, ssh keys, and other
  local resources that are accessible to that process.
- use the codex / chatgpt account already signed in on the host. this may
  consume usage quota or billing credits, and may expose account metadata shown
  by the app or cli, such as name or email address.

## features

- hostable on macOS, Linux (and anything codex cli + node will run on)
- reachable from the browser
- thin wrapper, so updates should land fast
- working today:
  - subagents
  - inline images
  - editor sidepanel
  - transcription

## roadmap

some parts of the desktop experience are not wired up yet:

- browser panel support, likely rebuilt around iframes
- computer use on linux, which could become a very powerful feature
- terminal support
- git worker integration
- whatever else people find and file issues for

## issues welcome

if something is broken, missing, or rough around the edges, please file an
issue.

using `codex-web` in an interesting way? post about it on x and tag me
[@0xcaff](https://x.com/0xcaff).

using this at a company and need something more tailored? email me and we can
talk.

## alternatives

* [davej/pocodex](https://github.com/davej/pocodex) i used this until the wheels fell off. i needed subagents
  and an inline image viewer. this didn't have them and was having a hard time
  keeping up with upstream codex updates.
* the native codex remote feature (behind a feature flag) is great for
  connecting to remote codex hosts over ssh to manage long running tasks but
  this only works if you have codex desktop on your client device. this means it
  doesn't work on mobile.
* upcoming first party mobile app from openai. `codex-web` exists and works
  today. i can't wait for the mobile app but judging by the other openai mobile
  apps, i'm a little bit skeptical about the quality of the mobile experience.
  time will tell.
