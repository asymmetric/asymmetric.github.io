---
layout: post
title: What happens when you run nixos-rebuild
date:  2019-12-21 13:40:34 +0100
---

[nixos-rebuild](https://github.com/NixOS/nixpkgs/blob/216f0e6ee4a65219f37caed95afda1a2c66188dc/nixos/modules/installer/tools/nixos-rebuild.sh)
is the command you run to bring your system's configuration in line with
`/etc/nixos/configuration.nix`.

What happens behind the scenes is the following:

# Your system configuration is evaluated

This typically means evaluating `/etc/nixos/configuration.nix` in the
context of a set of packages. The set of packages is the one pointed to
by the `NIX_PATH` variable – specifically, the part after `nixpkgs=`:

``` bash
❯ echo $NIX_PATH
/home/asymmetric/.nix-defexpr/channels nixpkgs=/nix/var/nix/profiles/per-user/root/channels/nixos nixos-config=/etc/nixos/configuration.nix /nix/var/nix/profiles/per-user/root/channels
```

# Packages are downloaded/built

including `nix` itself\! That's why you see two building/downloading
sections

# A system profile is created

System profiles are directories in the nix store encapsulating a
snapshot of the system state:

  - system-wide binaries
  - kernel options
  - files in `/etc`
  - manpages
  - shared libraries
  - …

A system profile store path could look something like
`/nix/store/a50aw2gj9dsn1yk3dsp33xw5y9zqfih9-nixos-system-asbesto-19.09.1484.84586a4514d`.

These directories are symlinked into well-known, human-readable
locations; the location depends on whether the profile is a *running* or
a *boot* one.

  - The *running* system profile is the one currently active on your
    system, and it's volatile: it is symlinked in `/run/current-system`,
    meaning that when your system reboots, it will be lost.
  - The *boot* system profile is the one your machine booted from, or
    will boot from. It's symlinked at `/nix/var/nix/profiles/system`,
    and is therefore persistent. Your bootloader contains an entry that
    tells it to load the kernel pointed to by the
    `/nix/var/nix/profiles/system/kernel` symlink.

Why do we need this distinction? It is to allow users to test a new
system configuration out, before adding an entry into the bootloader
(achieved via `nixos-rebuild test`); or to add an entry to the
bootloader, without changing the running system (`nixos-rebuild boot`).

## User profiles

User profiles exist alongside system profiles, and encapsulate a user's
view of the system (binaries, manpages, etc.)

They are symlinked under `/nix/var/nix/profiles/per-user/$(whoami)/`.

A user's `$PATH` variable points to their current user profile's `/bin`:

``` bash
❯ echo $PATH
/run/wrappers/bin /home/asymmetric/.nix-profile/bin /nix/var/nix/profiles/default/bin /run/current-system/sw/bin
```

As you can see above, the user's profile is also symlinked at
`~/.nix-profile`. Additionally, there's a profile called `default` -
this is the "global" user profile: if you run `nix-env -i` as root,
packages installed will be available to all users of the system.

# Recap

| Name                   | Actions                       | Locations                                       | Persistent | Rollback |
| ---------------------- | ----------------------------- | ----------------------------------------------- | ---------- | -------- |
| boot system profile    | `nixos-rebuild boot~/~switch` | `/nix/var/nix/profiles/system`                  | ✔          | ✔        |
| running system profile | `nixos-rebuild test~/~switch` | `/run/current-system`                           | ❌          | ❌        |
| user profile           | `nix-env -i`                  | `/nix/var/nix/profiles/per-user/$(whoami)`\[1\] | ✔          | ❌\[2\]   |
| default user profile   | `nix-env -i`\[3\]             | `/nix/var/nix/profiles/default`                 | ✔          | ✔        |

1.  Also symlinked at `~/.nix-profile`

2.  Rollback implemented by Home Manager

3.  When run as root
