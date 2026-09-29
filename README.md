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
and never rebuilt, that runs one fixed target (a script, or a Homebrew
binary by its stable path such as `/opt/homebrew/bin/rsync`). The
LaunchDaemon starts the launcher; macOS counts the program launchd started
as responsible for everything it runs, so the target gets the launcher's
grant. A launcher is `/Library/PrivilegedHelperTools/my-fda.<NAME>`,
root:wheel 700, and reads as `my-fda.<NAME>` in the Full Disk Access list.

## Commands

    my-fda list                       every grant: an ID, the client, allowed/denied, what to do
    my-fda add <PATH>                 "+": guides the grant (macOS grants only on a click)
    my-fda remove <ID>...             "-": shows what it would take away
    my-fda remove go <ID>...          ...and takes it away (TCC.db backed up first)
    my-fda reset [go]                 all grants at once, with a saved list to give them back
    my-fda create <NAME> <TARGET>     build and install a launcher
    my-fda destroy [go] <NAME>        delete a launcher and its grant
    my-fda status [<NAME>]            each launcher, its target, its grant, the daemon using it
    my-fda test <NAME>                start that daemon through launchd

The pipe `list` is made for:

    my-fda list | grep GONE | cut -f 1 | my-fda remove        # shows
    my-fda list | grep GONE | cut -f 1 | my-fda remove go     # acts

Today (0.1) `list` is built; the rest is designed. `my-fda --help` is the
reference for what the installed version does.

All commands need root, from a Terminal that has Full Disk Access itself:
the permission database is protected.

## Config

Plain shell, sourced at startup; `my-fda --create-config` prints the
default. Search order: `$MY_FDA_CONFIG`, `--config <FILE>`,
`/LINKS/default/my-fda.conf`, `/LINKS/default/my-fda`, `~/.my-fda.conf`,
`/etc/my-fda.conf`, `/usr/local/etc/my-fda.conf`.

## Tests

    my-fda --run-tests [<FILTER>]
