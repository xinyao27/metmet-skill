---
name: metmet
description: Manage the user's Mac through MetMet's metmet command — list listening TCP ports and stop the process holding one, list installed apps and open, quit or uninstall them with their leftovers, list, update or remove Homebrew packages, and scan for disk cleanup and move the chosen items to the Trash. Examples — "what is using port 3000", "free up port 8080", "which apps can I update", "uninstall Slack and its leftovers", "upgrade my outdated brew packages", "clean up my disk".
---

# MetMet (`metmet`)

`metmet` is the command line of the MetMet app on this Mac. Every command is answered by the
running MetMet over a private local socket, with the same checks as its window; MetMet is opened
in the background when it is not running. `metmet help` prints every command and option.

## If the `metmet` command is unavailable

`command not found` only means the launcher is not on this agent's `PATH`. First establish that
commands execute on the user's Mac rather than in a remote machine, container, or cloud runtime;
never claim to operate the user's Mac from anywhere else. Then:

- Ask whether MetMet is installed. If it is not, say why the task needs it and let the user
  install it; do not install it yourself.
- If MetMet is installed, ask the user to open it and turn on **Let local agents use MetMet** in
  MetMet → Settings (it is off by default). While it is on, MetMet keeps the launcher at
  `~/.local/bin/metmet`. If `~/.local/bin` is not on `PATH`, call `~/.local/bin/metmet` by its full
  path, or `/Applications/MetMet.app/Contents/MacOS/MetMet <command>`.
- Verify with `metmet status` before continuing: `enabled` must be `true`.

`status` and `capabilities` always answer; every other command fails with `disabled` while the
switch is off.

## Pick the command

| The user wants… | Run |
| --- | --- |
| to know whether MetMet is ready | `metmet status` |
| which ports are listening and who owns them | `metmet ports list` |
| to stop whatever holds a port | `metmet ports list` → `metmet ports terminate --id <socket-id>` → confirm |
| to pin or watch a port in MetMet | `metmet ports favorite\|watch --id <port> --enabled true\|false` |
| a list of installed apps | `metmet apps list` |
| to open or quit an app | `metmet apps open\|quit --id <application-id>` → confirm |
| to uninstall an app and its leftovers | `metmet apps uninstall --id <application-id>` → choose → confirm |
| Homebrew packages, or which are outdated | `metmet homebrew list` · `metmet homebrew check-updates` |
| to upgrade or remove a Homebrew package | `metmet homebrew upgrade\|uninstall --id <package-id>` → confirm |
| to free disk space | `metmet cleanup scan` → `metmet cleanup trash …` → confirm |
| to empty what a cleanup just moved to the Trash | `metmet cleanup permanent --receipt <id> --select <id> …` → confirm |
| to stop a long scan or cleanup | `metmet cleanup status` → `metmet cleanup cancel --id <operation-id>` |

## Preview, then confirm

Nothing that changes the Mac runs in one step. Every change is a **preview** that returns the
exact targets plus `confirmationToken`, `expiresAt` and `requiresConfirmation: true`; nothing
has happened yet. A separate command carries it out:

```sh
metmet ports confirm --confirm '<token>'      # after ports terminate
metmet apps confirm --confirm '<token>'       # after apps open | quit | uninstall
metmet homebrew confirm --confirm '<token>'   # after homebrew upgrade | uninstall
metmet cleanup confirm --confirm '<token>'    # after cleanup trash | permanent
```

- A token lasts five minutes, works once, and only with its own domain's `confirm`.
- Before confirming anything that removes data or stops a process — terminate, quit, uninstall,
  Homebrew uninstall, cleanup trash, permanent delete — show the user what the preview lists
  (names, paths, sizes, the process) and get their yes. `open`, a Homebrew `upgrade` the user
  asked for, and favorite/watch need no extra check.
- If the user changes the selection, run the preview again; a token is bound to the targets it
  was issued for.
- `busy` means MetMet is doing something else, in its window or for another command; it does
  not consume the token. Retry the same confirm while it is still valid.

## IDs come from earlier output

Never invent an ID, a path, or a PID; each command accepts only IDs MetMet itself returned.

| Flag | Comes from |
| --- | --- |
| `ports terminate --id` | `ports[].id` in `ports list` (the socket id, not the port number) |
| `ports favorite\|watch --id` | the port number, 1–65535 |
| `apps … --id` | `applications[].id` in `apps list` |
| `apps uninstall --select` | `candidates[].id` from the same app's `apps uninstall` preview |
| `homebrew … --id` | `packages[].id` in `homebrew list` or `check-updates` |
| `cleanup trash --plan` / `--select` | `id` / `candidates[].id` in `cleanup scan` |
| `cleanup permanent --receipt` / `--select` | `id` / `items[].id` in the trash confirm's result, or `cleanup receipt` |
| `cleanup cancel --id` | `activeOperationID` in `cleanup status` |

## Ports

`ports list` returns every listening TCP socket with its `port`, `endpoint`, `processName` and
`owner` (pid and start time), plus `isFavorite`, `isWatched` and `canTerminate`. Only sockets
with `canTerminate: true` belong to this user and can be stopped.

`ports terminate --id <socket-id>` previews a polite `SIGTERM` to the owning process, covering
every socket that process holds (`approvedSocketIDs`). It takes a socket id, not a port number;
when several processes listen on the port you passed, the error's `detail.candidateIDs` lists
their socket ids. `--force` previews a `SIGKILL` instead: use it only after a polite attempt came back
`stillListening` and the user agrees. The confirm reports `outcome`: `released`,
`stillListening`, `rebound` (another process took the port), `identityChanged` (the process was
replaced, so nothing was signalled), `permissionDenied`, or `scanFailed`.

## Apps

`apps list` returns `applications` with `id`, `name`, `bundleID`, `path`, `version`, `bytes`,
`source`, `isRunning` and `isProtected`; protected apps (system apps, MetMet itself) cannot be
opened, quit or uninstalled here. `complete: false` means some folders could not be read.

Uninstalling takes three calls:

1. `metmet apps uninstall --id <app-id>` with no selection only lists `candidates` — the app
   itself and its support files, caches, preferences and launch items — each with `path`,
   `bytes`, `isSelectable`, `defaultSelected` and `isHeuristic`. No token yet.
2. Preview the choice: `--recommended` takes the default selection, or repeat `--select <id>`
   for each item. The app's own id must be among them. Show the user the list before going on.
3. `metmet apps confirm --confirm <token>`. Items move to the Trash, never deleted outright;
   the result lists `movedCount`, `retainedCount` and each item's `trashPath`.

The app must not be running: quit it first (preview and confirm `apps quit`). An app installed
by a Homebrew cask is uninstalled the same way; Homebrew removes the app, then the chosen
leftovers go to the Trash.

## Homebrew

`homebrew list` reads installed formulae and casks; `homebrew check-updates` also refreshes
which are outdated (`isOutdated`, `latest`, `outdatedCount`). `status` is `missing` when Homebrew
is not installed and `partial` when it could not be read completely.

`homebrew upgrade` refuses pinned packages; `homebrew uninstall` refuses a cask that owns an app
and points to `apps uninstall` through `detail.applicationIDs`. Both refuse while the package's
apps are running. Homebrew runs its own uninstall, so those files do not go through the Trash —
say so before confirming.

## Cleanup

1. `metmet cleanup scan` returns a plan: `id`, `freeBytes`, `issues` (paths it could not read)
   and `candidates` with `name`, `path`, `category`, `bytes`, `reason`, `impact`, `isSelectable`
   and `recommended`. A scan can take a while; `cleanup status` shows its `activeOperationID`.
2. `metmet cleanup trash --plan <plan-id>` with either `--recommended` or one or more
   `--select <candidate-id>` — exactly one of the two. Show the user what it lists.
3. `metmet cleanup confirm --confirm <token>` moves them to the Trash and returns a receipt:
   `id`, `freeSpaceDeltaBytes` and `items` with each item's `status` and `trashPath`.

Items stay in the Trash, recoverable. Delete them for good only when the user asks:
`metmet cleanup permanent --receipt <receipt-id> --select <item-id> …`, then confirm. Receipts
are kept for 15 minutes; `cleanup status` gives `latestReceiptID` and `cleanup receipt --id`
reads one again.

## Results and errors

A successful command prints one JSON value on stdout and exits 0. A failure prints
`{"error":{"code","message","detail"}}` on stderr:

| Exit | Codes | What to do |
| --- | --- | --- |
| 64 | `invalidArguments` | fix the command; `detail` may list valid choices |
| 66 | `notFound` | the ID is stale or the token expired: list or scan again |
| 75 | `busy`, `notRunning` | wait a moment and retry the same command |
| 77 | `disabled`, `needsApproval` | ask the user to turn on Agent access, or take the route the message names |
| 70 | `failed` | report the message to the user |

Only one change runs at a time across MetMet's window and every command.
