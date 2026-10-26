# my-fda

Full Disk Access for root jobs on macOS, kept stable across Homebrew upgrades.

## Why

macOS ties a Full Disk Access grant to a binary's path and contents. A
Homebrew binary changes both on every `brew upgrade`, so its grant silently
stops applying and the root job using it fails with "Operation not
permitted". And the list in System Settings grows with every version ever
granted.

## How

A *launcher* holds the grant instead: a tiny compiled program, built once
and never rebuilt, that runs one fixed target -- a script, or a Homebrew
binary by its stable path such as `/opt/homebrew/bin/rsync`. The
LaunchDaemon starts the launcher; macOS counts the program launchd started
as responsible for everything it runs, so the target gets the launcher's
grant.

- A launcher is `/Library/PrivilegedHelperTools/my-fda-<NAME>`, root:wheel
  744, and reads as `my-fda-<NAME>` in the Full Disk Access list. No dot in
  the name: macOS takes what follows a dot for an extension and files the
  launcher as a Document, which the pane refuses -- dragged in or chosen with
  +. A launcher from before, `my-fda.<NAME>`, still runs its job; `status`
  marks it, and `add` refuses it with the fix (destroy, create). Readable by
  everyone, because System Settings, which runs as the screen's user, reads
  the file to add it (a file only root can read is refused there without a
  word); run by root only, because the grant is that path's. `add` makes an
  older 700 launcher readable first, and refuses any other file the screen's
  user cannot read, saying why.
- `create` builds it on the machine with its own `cc` (Command Line Tools);
  the C source lives inside `my-fda`, never a binary in git.
- It spawns the target and waits, so the process holding the grant lives for
  the whole run; it passes on all arguments and SIGTERM/SIGINT/SIGHUP, and
  exits with the target's status.
- The target is compiled in (`MY-FDA-TARGET=<path>`), never taken from the
  command line; `status` reads it back out of the binary.
- `create` never rebuilds an existing launcher: a rebuild loses its grant.
- `create --user <USER>`: for a job that runs as a user. launchd starts the
  launcher as root (its LaunchDaemon names no `UserName`), and the launcher
  becomes USER -- groups, group, user, HOME/USER/LOGNAME/SHELL -- before it
  starts the target; when it cannot, the target does not run at all. The
  user is compiled in too (`MY-FDA-USER=<user>`), and `test` refuses a
  LaunchDaemon that would start such a launcher as anyone but root.

- `add` puts the program into the Full Disk Access list switched OFF -- a
  row in TCC.db with auth 0 and the program's code requirement, TCC.db
  backed up first -- and opens the pane: one click there grants it, so the
  grant stays a person's act. Where TCC.db cannot be written, it puts the
  path on the clipboard for +, Cmd-Shift-G, Cmd-V. A drag into the list can
  be lost: the pane asks for the password and drops the item, without an
  error (ada, macOS 27).

## Commands

    my-fda list                       every grant: an ID, the client, allowed/denied, what to do
    my-fda denied [<LAST>]            every program refused Full Disk Access in the system log (default: the last 1d)
    my-fda add <PATH>                 "+": lists PATH switched OFF, opens the pane -- you switch it on -- checks it
    my-fda remove <ID>...             "-": shows what it would take away
    my-fda remove go <ID>...          ...and takes it away
    my-fda cleanup [go]               the GONE grants: shows them; with go, takes them away
    my-fda reset [go]                 all grants at once, and what to give back
    my-fda create [<NAME>] <TARGET>   build and install a launcher; NAME defaults to the target's file name (--user <USER>: the target runs as USER)
    my-fda destroy [go] <NAME>        delete a launcher and its grant
    my-fda status [<NAME>]            each launcher, its target, its grant, the daemon using it
    my-fda test <NAME>                start that daemon through launchd

A destructive command shows first and acts only with `go`. GONE grants --
the one class safe to take away without looking -- go with `cleanup go`;
any other selection goes through the pipe `list` is made for:

    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove        # shows
    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove go     # acts

- **denied** finds the jobs that need a grant or a launcher: a job refused
  Full Disk Access just fails, often with a misleading error, and only
  tccd's lines in the system log say so. One line per program, as a grant
  names it, with how often, when last, the binary that touched the file
  (`via`), and what to do -- `/bin/dash ... via /usr/sbin/diskutil  a
  shell` is a root shell script running diskutil, and needs a launcher.
- **IDs** are 6 hex characters of the client, the same on every run, followed
  by a TAB -- `cut -f 1` returns them.
- **A path is granted as its real path:** macOS records the file with its
  symlinks resolved, so `add` resolves PATH first and says so when it
  differs (`/LINKS/local/sbin/x is /Users/.../local/sbin/x`); `list` shows
  that real path, which is what to `grep` for.
- **Changes** start with a backup (`sqlite3 .backup` into `BACKUP_DIR`,
  never `cp` -- tccd holds the file open). Each grant then goes the one way
  the machine allows: an app id through `tccutil`, macOS's own command,
  which knows installed apps only; the rest -- paths, and apps tccutil does
  not know, such as a GONE one -- through sqlite in one transaction where
  TCC.db can be written, else my-fda opens the pane and names each one as
  the pane does (a path's file name, an app's `.app` name) to take away
  with "-". A GONE app neither tccutil nor sqlite can take stays, said
  once: the pane does not show it either, and it applies to nothing. Then
  the grants are read again, and anything else still there is an error.
  Even root with Full Disk Access may be refused a write (macOS 26 on
  macado); the probe for it must really write, since sqlite opens such a
  file read-only without a word. As root, the pane and Finder open in the
  session of the user at the screen (`launchctl asuser`): plain `open`
  from root is refused.
- **Listed, never removed:** an Endpoint Security extension's grant
  (TCC.db, `kTCCServiceEndpointSecurityClient` -- the pane shows it with the
  others) and a configuration profile's (`MDMOverrides.plist` -- the pane
  does not show it; it goes with the profile).
- **`reset`** is `tccutil reset SystemPolicyAllFiles`: every grant at once,
  by macOS itself; it saves the list beside the backup first, so the grants
  to give back are known.
- **`add` only guides**: macOS grants on a click (or a device-management
  profile), and writing grants is what malware does.
- **App ids** are `GONE` only when neither Spotlight nor LaunchServices
  knows them: Spotlight misses helper apps inside other apps, and
  LaunchServices, asked as root, may miss apps in a user's
  `~/Applications`. Apple's own are never `GONE`, and none at all when
  Spotlight finds no apps.
- **`test`** is the only run that proves a grant: a run from a Terminal uses
  the Terminal's.

All commands need root, from a Terminal that has Full Disk Access itself:
the permission database is protected. `my-fda --help` is the reference.

## Config

Plain shell, sourced at startup over the defaults: a key the file does not
set keeps its default, so an older config needs no edit when a new key
arrives. `my-fda --create-config` prints the defaults. Search order: `$MY_FDA_CONFIG`, `--config <FILE>`,
`/LINKS/default/my-fda.conf`, `/LINKS/default/my-fda`, `~/.my-fda.conf`,
`/etc/my-fda.conf`, `/usr/local/etc/my-fda.conf`.

## Tests

    my-fda --run-tests [<FILTER>]

They run against a TCC.db of their own, with stubs for launchctl, open and
Spotlight -- never against the real one, and without root.
