--
layout: page
title: Evaluating the security implications of a company-wide Nix remote builder
--

I am trying to test nix's security properties in this scenario:

- A company with a remote Nix builder
- N users have ssh write access to the remote builder
- A user's local machine is compromised, with the attacker having root access
- The attackre cannot SSH to the remote builder without user interaction
- The builder is also used as a substituter
- **Question**: can the attacker distribute tampered store paths to the rest of the company?

To test this scenario, i'm trying to tamper my own store paths :slight_smile:.

What I did:

- remount the store as read-write: `mount -o remount,rw /nix/store`
- `nix build nixpkgs#hello`
- tamper with `nixpkgs#hello`:

```
 echo foo > /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12/bin/hello
```

If I now try to copy this path to a remote store, I get the following:

```
nix copy --to ssh://foobar /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 
copying path '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12' to 'ssh://foobar'error: hash mismatch importing path '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12';
         specified: sha256:1047k69qrr209c6jgls43620s53w2gw7gsgwx56j0b180bsn1qhw
         got:       sha256:0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x
error: unexpected end-of-file
```

### Digression: different bases for different outputs

Where is the hash coming from? 

The manpage for `nix store verify` says that the command checks that a store path's

> contents match the NAR hash recorded in the Nix database

and running 

```
nix store verify /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 
path '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12' was modified! expected hash 'sha256:1047k69qrr209c6jgls43620s53w2gw7gsgwx56j0b180bsn1qhw', got 'sha256:0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x'
```

returns the same two hashes, so I went looking in the nix db, and of the 3 tables there:

```sql
sqlite> .tables
DerivationOutputs  Refs               ValidPaths  
```

`ValidPaths` seemed the most promising (as it's the only one with a `hash` column).

But searching for the `specified` or `got` hashes returned no results:

```sql
sqlite> SELECT * from ValidPaths WHERE hash = 'sha256:1047k69qrr209c6jgls43620s53w2gw7gsgwx56j0b180bsn1qhw' OR hash = 'sha256:0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x';
sqlite> 
```

`nix path-info` returns the **expected** `narHash`:

```
❯ nix hash to-base32 (nix path-info --json /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 | jq -r .[0].narHash)
1047k69qrr209c6jgls43620s53w2gw7gsgwx56j0b180bsn1qhw
```


`nix hash path` returns the **actual** `narHash`:

```
❯ nix hash to-base32 (nix hash path /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12)
0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x
```

So it turns out that:

- the hash in the DB is base16
- the hash in `nix verify` output is base32
- the hash in `nix path-info` output is base64
- the hash in `nix hash path` output is SRI base64

I have no idea why this is the case, and it is very confusing.

but I assume with some more searching, i could find it, and also that an attacker with root access could change values in the db. at that point, i think an attacker with write access to a remote builder/binary cache would able to distribute tampered store paths
