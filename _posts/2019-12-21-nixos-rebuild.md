---
layout: post
title: What happens when you run nixos-rebuild
date:  2019-12-21 13:40:34 +0100
---

# Table of Contents

1.  [your system configuration is evaluated](#org30df45f)
2.  [packages are downloaded/built](#org34af37a)
3.  [a system profile is created](#org413b089)
4.  [user profiles](#org0b7776a)
5.  [recap](#org5d506bd)

[nixos-rebuild switch~](https://github.com/NixOS/nixpkgs/blob/216f0e6ee4a65219f37caed95afda1a2c66188dc/nixos/modules/installer/tools/nixos-rebuild.sh) is the command you run to bring your system&rsquo;s
configuration in line with `configuration.nix`.

What happens behind the scenes is the following:


<a id="org30df45f"></a>

# your system configuration is evaluated

This typically means evaluating `/etc/nixos/configuration.nix` in the context of
a set of packages.
The set of packages is the one pointed to by the `NIX_PATH` variable
(specifically, the part after `nixpkgs=`).

Ultimately this evaluates to a list of packages to build/download, and a list of
config files/kernel options to set on your system.


<a id="org34af37a"></a>

# packages are downloaded/built

including `nix` itself! That&rsquo;s why you see two building/downloading sections


<a id="org413b089"></a>

# a system profile is created

System profiles are directories in the nix store encapsulating a snapshot of the
system state:

-   system-wide binaries
-   kernel options
-   files in `/etc`
-   manpages
-   shared libraries
-   &#x2026;

A system profile store path could look something like
`/nix/store/a50aw2gj9dsn1yk3dsp33xw5y9zqfih9-nixos-system-asbesto-19.09.1484.84586a4514d`.

These directories are symlinked into well-known, human-readable locations; the
location depends on whether the profile is a *running* or a *boot* one.

The *running* system profile is the one currently active on your system, and is
volatile: it is symlinked in `/run/current-system`, meaning that when your
system reboots, it will be lost.
The *boot* system profile is the one your machine booted from, or will boot
from. It&rsquo;s symlinked at `/nix/var/nix/profiles/system`, and is therefore
persistent.
Your bootloader contains an entry that tells it to load the kernel at
`/nix/var/nix/profiles/system/kernel`.
TODO: is this path correct?

Why do we need this distinction? It is to allow users to test a new system
configuration out, before adding an entry into the bootloader (achieved via
`nixos-rebuild test`); or to add an entry to the bootloader, without changing the
running system (`nixos-rebuild boot`).


<a id="org0b7776a"></a>

# user profiles

User profiles exist alongside system profiles, and encapsulate a user&rsquo;s view of
the system (binaries, manpages, etc.)

They are symlinked under `/nix/var/nix/profiles/per-user/$(whoami)/`.

A user&rsquo;s `$PATH` variable points to their current user profile&rsquo;s `/bin`:

    ❯ echo $PATH
    /run/wrappers/bin /home/asymmetric/.nix-profile/bin /nix/var/nix/profiles/default/bin /run/current-system/sw/bin

As you can see above, the user&rsquo;s profile is also symlinked at `~/.nix-profile`.
Additionally, there&rsquo;s a profile called `default` - this is the &ldquo;global&rdquo; user
profile: if you run `nix-env -i` as root, packages installed will be available
to all users of the system.


<a id="org5d506bd"></a>

# recap

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Name</th>
<th scope="col" class="org-left">Actions</th>
<th scope="col" class="org-left">Locations</th>
<th scope="col" class="org-left">Persistent</th>
<th scope="col" class="org-left">Rollback</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">boot system profile</td>
<td class="org-left">`nixos-rebuild boot~/~switch`</td>
<td class="org-left">`/nix/var/nix/profiles/system`</td>
<td class="org-left">✅</td>
<td class="org-left">✅</td>
</tr>


<tr>
<td class="org-left">running system profile</td>
<td class="org-left">`nixos-rebuild test~/~switch`</td>
<td class="org-left">`/run/current-system`</td>
<td class="org-left">❌</td>
<td class="org-left">❌</td>
</tr>


<tr>
<td class="org-left">user profile</td>
<td class="org-left">`nix-env -i`</td>
<td class="org-left">`/nix/var/nix/profiles/per-user/$(whoami)`<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup></td>
<td class="org-left">✅</td>
<td class="org-left">❌<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup></td>
</tr>


<tr>
<td class="org-left">default user profile</td>
<td class="org-left">`nix-env -i`<sup><a id="fnr.3" class="footref" href="#fn.3">3</a></sup></td>
<td class="org-left">`/nix/var/nix/profiles/default`</td>
<td class="org-left">✅</td>
<td class="org-left">✅</td>
</tr>
</tbody>
</table>


# Footnotes

<sup><a id="fn.1" href="#fnr.1">1</a></sup> Also symlinked at `~/.nix-profile`

<sup><a id="fn.2" href="#fnr.2">2</a></sup> Rollback implemented by Home Manager

<sup><a id="fn.3" href="#fnr.3">3</a></sup> When run as root
