# Decisions

## Why not Crafty Controller?

Crafty is a real, well-built multi-user web panel (own auth system, roles,
scheduler, file manager, console) and was tried first. It was dropped for
this repo's structure for one concrete reason:

**Split ownership of the running process.**

The original attempt ran the Minecraft server in its own `itzg`-based
container (bind-mounted to `./bedrock-data`), and *also* mounted that same
folder into a separate Crafty container, hoping Crafty would pick it up as
a managed server. It didn't — Crafty tracks servers in its own internal
database, not by scanning folder contents, so nothing showed up in the
dashboard until a server was explicitly created/imported through its UI.

Even after importing the folder as an "existing server," a deeper problem
remained: Crafty's Start/Stop/Console controls assume *Crafty itself* spawned
the server process. When the real process is actually owned by a separate,
independently-running `itzg` container, Crafty's controls don't map onto
that real process — you end up with two systems that each think they're in
charge, and neither fully is.

The fix Crafty itself expects is **Option B: let Crafty fully own the
server** (create a brand-new server through its UI, let it download and
run the binary itself, configure everything through its UI instead of
Docker env vars). That's a legitimate way to use Crafty — but it moves the
entire configuration out of the git-trackable `docker-compose.yml` and into
Crafty's own SQLite state, which conflicts with the goal of this repo: a
fully reproducible, version-controlled stack.

## Why the `itzg` images instead?

- Every setting (`server.properties` equivalents, version, memory, RCON,
  plugin/mod auto-install) is an environment variable in one YAML file —
  reproducible, diffable, and trackable in git.
- Plugin/mod automation (`MODRINTH_PROJECTS`, `CURSEFORGE_FILES`, etc. on
  the Java image) is more capable than manually uploading jars through a
  panel's file manager.
- Actively maintained, widely used, handles graceful shutdown/save-on-stop,
  healthchecks, and version switching cleanly.

The tradeoff — no built-in web console, backup scheduler, file browser, or
multi-user permissions — is solved by bolting on small, independent sidecar
containers instead of one monolithic panel. See the table in
[`README.md`](../README.md) and the compose file for the specific tools
chosen for each gap. Each sidecar is optional and removable independently,
and none of them claim ownership of the Minecraft process itself — they
either read the same volume, talk over RCON, or just watch the server from
the outside. That keeps `mc` as the single, unambiguous owner of the actual
game server.

## When Crafty (or a similar panel) *would* be the better choice

If the priority is ease of management over reproducibility — e.g. running
servers for a non-technical group, wanting one login for everything, not
caring about git-tracked config — Option B (Crafty fully owns the server)
is a perfectly reasonable choice. It's a different tradeoff, not strictly a
worse tool. This repo just optimizes for the opposite priority.

## Why RCON Web Admin (`itzg/rcon`) was removed

`rcon-web` was originally added as the web-console sidecar. It never
worked correctly even after fixing the config (see `docs/operations.md`'s
"Known issues" section for the specifics — wrong env var for username,
missing `RWA_ENV`/`RWA_RCON_*` vars, wrong host), and even once fixed, it
stayed a thin layer most of the value of which (console access, player
list) is already covered by:
- `docker attach mc` / `docker exec mc rcon-cli <command>` for direct
  console access, or
- `docker exec -it bedrock send-command <command>` on Bedrock.

Given the extra moving part wasn't earning its keep, it was dropped
entirely rather than kept half-used. `ENABLE_RCON` stays on for `mc`
itself since `mc-backup` still depends on it internally (to pause
world-saving during backups) — RCON just no longer has any UI in front of
it.
