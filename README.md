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

- A launcher is `/Library/PrivilegedHelperTools/my-fda.<NAME>`, root:wheel
  700, and reads as `my-fda.<NAME>` in the Full Disk Access list.
- `create` builds it on the machine with its own `cc` (Command Line Tools);
  the C source lives inside `my-fda`, never a binary in git.
- It spawns the target and waits, so the process holding the grant lives for
  the whole run; it passes on all arguments and SIGTERM/SIGINT/SIGHUP, and
  exits with the target's status.
- The target is compiled in (`MY-FDA-TARGET=<path>`), never taken from the
  command line; `status` reads it back out of the binary.
- `create` never rebuilds an existing launcher: a rebuild loses its grant.

## Commands

    my-fda list                       every grant: an ID, the client, allowed/denied, what to do
    my-fda add <PATH>                 "+": opens the pane, shows PATH in Finder to drag in, checks it
    my-fda remove <ID>...             "-": shows what it would take away
    my-fda remove go <ID>...          ...and takes it away
    my-fda cleanup [go]               the GONE grants: shows them; with go, takes them away
    my-fda reset [go]                 all grants at once, and what to give back
    my-fda create <NAME> <TARGET>     build and install a launcher
    my-fda destroy [go] <NAME>        delete a launcher and its grant
    my-fda status [<NAME>]            each launcher, its target, its grant, the daemon using it
    my-fda test <NAME>                start that daemon through launchd

A destructive command shows first and acts only with `go`. GONE grants --
the one class safe to take away without looking -- go with `cleanup go`;
any other selection goes through the pipe `list` is made for:

    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove        # shows
    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove go     # acts

- **IDs** are 6 hex characters of the client, the same on every run, followed
  by a TAB -- `cut -f 1` returns them.
- **Changes** start with a backup (`sqlite3 .backup` into `BACKUP_DIR`,
  never `cp` -- tccd holds the file open). Each grant then goes the one way
  the machine allows: an app id through `tccutil`, macOS's own command,
  which knows installed apps only; the rest -- paths, and apps tccutil does
  not know, such as a GONE one -- through sqlite in one transaction where
  TCC.db can be written, else my-fda opens the pane with exactly the list
  to take away with "-". Then the grants are read again, and anything
  still there is an error. Even root with Full Disk Access may be refused
  a write (macOS 26 on macado); the probe for it must really write, since
  sqlite opens such a file read-only without a word.
- **`reset`** is `tccutil reset SystemPolicyAllFiles`: every grant at once,
  by macOS itself; it saves the list beside the backup first, so the grants
  to give back are known.
- **`add` only guides**: macOS grants on a click (or a device-management
  profile), and writing grants is what malware does.
- **App ids** are checked with Spotlight: one it cannot find is `GONE`,
  except Apple's own (Spotlight does not find them all) -- and none at all
  when Spotlight finds no apps.
- **`test`** is the only run that proves a grant: a run from a Terminal uses
  the Terminal's.

All commands need root, from a Terminal that has Full Disk Access itself:
the permission database is protected. `my-fda --help` is the reference.

## Config

Plain shell, sourced at startup; `my-fda --create-config` prints the
default. Search order: `$MY_FDA_CONFIG`, `--config <FILE>`,
`/LINKS/default/my-fda.conf`, `/LINKS/default/my-fda`, `~/.my-fda.conf`,
`/etc/my-fda.conf`, `/usr/local/etc/my-fda.conf`.

## Tests

    my-fda --run-tests [<FILTER>]

They run against a TCC.db of their own, with stubs for launchctl, open and
Spotlight -- never against the real one, and without root.
