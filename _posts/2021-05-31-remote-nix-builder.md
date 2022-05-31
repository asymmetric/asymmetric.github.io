---
layout: post
title: Evaluating the security implications of a company-wide Nix remote builder
---

I am trying to test nix's security properties in this scenario:

- A company with a remote Nix builder
- `N` users have ssh write access to the remote builder
- A user's local machine is compromised, with the attacker having root access
- The attacker cannot SSH to the remote builder without user interaction (e.g. because of a YubiKey-backed SSH key)
- The builder is also used as a binary cache
- **Question**: can the attacker distribute tampered store paths to the rest of the company?

To test this scenario, I'm trying to tamper my own store paths :).

What I did:

- remount the store as read-write: `mount -o remount,rw /nix/store`
- `nix build nixpkgs#hello`
- tamper with `nixpkgs#hello`:

```
 echo foo > /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12/bin/hello
```

If I now try to copy this path to a remote store, I get the following:

```sh
nix copy --to ssh://foobar /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 
copying path '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12' to 'ssh://foobar'error: hash mismatch importing path '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12';
         specified: sha256:1047k69qrr209c6jgls43620s53w2gw7gsgwx56j0b180bsn1qhw
         got:       sha256:0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x
error: unexpected end-of-file
```

Where are the 2 hashes coming from? 

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

The key thing here is that in Nix, hashes are stored/displayed in many different formats:

- the hash in the DB is base16
- the hash in the `nix copy` output is base32
- the hash in the `nix verify` output is base32
- the hash in the `nix path-info` output is SRI base64
- the hash in the `nix hash path` output is SRI base64

I have no idea why this is the case, and it is very confusing.

So in the interest of clarity, I'll provide the SRI form (using `nix hash to-sri`) of the two hashes above:

- `specified` : `sha256-HOJg9QIoLCBN6fzpd/gTfBQNhBlE0ycNS0DkjJOZh4A=`
- `got`: `sha256-HQm86KUSV14rqFxgNJsG3JgEqIgmoTP2x7kTiJsKnEY=`

`nix path-info` returns the **expected** `narHash`:

```
❯ nix path-info --json /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 | jq -r .[0].narHash
sha256-HOJg9QIoLCBN6fzpd/gTfBQNhBlE0ycNS0DkjJOZh4A=
```


`nix hash path` returns the **actual** `narHash`:

```
❯ nix hash path /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12
sha256-HQm86KUSV14rqFxgNJsG3JgEqIgmoTP2x7kTiJsKnEY=
```

Two things to note:

- The store path is actually copied over to the remote host, but it is **not** added to its nix db's `ValidPaths` table, thereby making it invisible.
- The `narHash` check is performed by the remote store

An attacker could, at this point, modify the `narHash` (and the `narSize`[^0]) in the local Nix db:

```sql
sqlite> UPDATE ValidPaths
   ...> SET hash = 'sha256:1d09bce8a512575e2ba85c60349b06dc9804a88826a133f6c7b913889b0a9c46',
   ...> narSize = 125720
   ...> WHERE path = '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12';
```

And now:

```bash
❯ nix copy --to ssh://foobar /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12 

❯ echo $?
0
```

### Potential mitigations

The issue here is that paths in the store are input-addressed, meaning that
once they're in the store, there's nothing (apart from signatures, but they're
not relevant here) that prevents them from being tampered with.

Another option (still partly in the works, AFAIU) are proper [content-addressed
paths](https://github.com/tweag/rfcs/blob/cas-rfc/rfcs/0062-content-addressed-paths.md).
If we do:

```
❯ nix store make-content-addressed /nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12
rewrote '/nix/store/hkgpl034l6c5zgzhks2dyp7p41z6qyc4-hello-2.12' to '/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12'

❯ nix path-info --json /nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12 | jq .
[
  {
    "path": "/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12",
    "narHash": "sha256-HQm86KUSV14rqFxgNJsG3JgEqIgmoTP2x7kTiJsKnEY=",
    "narSize": 125720,
    "references": [
      "/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12",
      "/nix/store/ah5d5gjqyv319ir95lm8bc44ikkzm10i-glibc-2.34-115"
    ],
    "ca": "fixed:r:sha256:0ilw1adqh4xrqzv37896i2l0966w0sdk8q2wm0mmwmqjlplbq28x",
    "registrationTime": 1653987573
  }
]
```

we can see that the path now has an additional field, `ca`, which contains the hash of its actual content.

This cannot be tampered with, unless one is able to break the hashing algorithm and produce a collision.

After tampering with the `narHash` and the `narSize` as above, we are now presented with this error:

```
nix copy --to ssh://foobar /nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12
copying path '/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12' to 'ssh://foobar'warning: path '/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12' claims to be content-addressed but isn't
error: cannot add path '/nix/store/9xnc7c495iw6lqk9wxb5r1v8003sp3rp-hello-2.12' to the Nix store because it claims to be content-addressed but isn't
error: unexpected end-of-file
```

### Conclusions

Adding a remote builder opens up the possibility that an attacker might sneak
in malicious store paths, which could be very hard to detect. In effect, the
remote builder's store becomes a db with multiple writers, each of which
has access full read/write access.

CA derivations provide a mitigation for this usecase, and they rely on the security properties of the hashing algorithm.

It's important to be aware of this when evaluating a nix build infrastructure.

[^1]: I haven't shown this for brevity, but the `nix copy` command showed an error for the `narSize` mismatch.
