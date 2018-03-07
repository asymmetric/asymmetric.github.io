---
layout: post
title: Implementing a Blockchain in Rust
date:  2018-02-11 13:40:34 +0100
---

This is another one of those blog posts teaching you how to implement a simple,
Bitcoin-like blockchain. It has been heavily inspired by Ivan Kuznetsov's
[awesome series](https://jeiwan.cc/posts/building-blockchain-in-go-part-1/).
What makes it different from previous examples is that this one is written in
Rust.

So if these two things happen to interest you, read along.

### What's in a blockchain

A blockchain is a data structure resembling a singly-linked list, i.e. a list
  where each element has one (and only one) pointer to the preceding one. This
  pointer is, in the case of a blockchain, the hash of the header of the
  preceding block in the chain.

We just introduced 3 new terms: hash, block and header.

What's a hash? A hash is a function that produces a fixed size output for any
given input. For example, SHA-256 always produces a 256 bit output, which is
usually displayed to humans in hexadecimal. The SHA-256 of the string "I am
Satoshi Nakamoto", displayed in hex, is
`a756b325faef56ad975c1bf79105bfc427e11102aa159828c8b416f5326a8440`.

If a blockchain is a list, a block is an element in the list. It is composed of:

* a header
* a body

In real blockchains, the body contains the transactions that represent the
exchange of value across accounts, but in our case, the body will just be a
string.

**A note about structure:** The code examples in this blog post are not meant to
exemplify the best practices on how to structure a Rust project. All the code
is in one file; the code snippets are not meant to be run in isolation; and we
don't separate library and binary crates.
Check out [this
chapter](https://doc.rust-lang.org/book/second-edition/ch07-00-modules.html)
the Rust book for info on all that and more.

So with that out of the way, let's take a first stab at implementing a block:

```rust
const HASH_BYTE_SIZE: usize = 32;

pub type Sha256Hash = [u8; HASH_BYTE_SIZE];

#[derive(Debug)]
pub struct Block {
    // Headers.
    timestamp: i64,
    prev_block_hash: Sha256Hash,

    // Body.
    // Instead of transaction, blocks contain data.
    data: Vec<u8>,
}
```

The block has a header, containing an array of 32 bytes (for the 256-bit SHA-256
hash) and a timestamp, and a body, with a `data` field of type `Vec<u8>` for variable length strings.

Now that we know what a block is, we can proceed to the next step, i.e.
concatenating them in a chain.

In order to do that, we need to have a way to create a new block, and set its
`prev_block_hash` to the hash of the headers of  for our blockchainanother block. Let's implement the `new`
function:

```rust
use chrono::prelude::*;

impl Block {
    // Creates a new block.
    pub fn new(data: &str, prev_hash: Sha256Hash) -> Self {
        Self {
            prev_block_hash: prev_hash,
            data: data.to_owned().into(),
            timestamp: Utc::now().timestamp(),
        }
    }
}
```

So far, so good. We pass a hash and a string to the `new` function, and it gives
us an initialized block. But how do we get the hash in the first place?

### Proof of Work

As we saw previously, in our blockchain a hash is `prev_block_hash: SHA(header)` (in Bitcoin, it's actually
`SHA(SHA(header)`).

In Bitcoin-like blockchains, miners are the nodes that take new transactions,
put them in a block, and then compete with each other for the right to have
the block they created added to the blockchain.

What does the competition consist of? The race is to be the first one in the
network to find a hash with a numeric value lower than the current **target**.

The target is a hexadecimal number with an amount *x* of leading zeroes. *x* is
the difficulty.

For example, if the difficulty is 5, that means we need to produce a hash that,
when expressed as hex, has a value lower than
`0x0000010000000000000000000000000000000000000000000000000000000000` (notice
that the target has 5 leading zeroes).

This is a type of calculation that can only be brute-forced, i.e. there's no
other way to find such a hash than to go through all possible iterations.

But if the header doesn't change, there is only one iteration to
perform: `SHA(header)`, for a header that doesn't change, will obviously always output
the same value, no matter how many times you run it!

For this reason, we need to add another field to our header, the so-called
**nonce**, which is incremented on each iteration. 

```rust
const HASH_BYTE_SIZE: usize = 32;

pub type Sha256Hash = [u8; HASH_BYTE_SIZE];

#[derive(Debug)]
pub struct Block {
    // Headers.
    timestamp: Utc::now().timestamp(),
    prev_block_hash: Sha256Hash,
    nonce: u64,

    // Body.
    // Instead of transactions, blocks contain data.
    data: Vec<u8>,
}

impl Block {
    // Creates a new block.
    pub fn new(data: &str, prev_hash: Sha256Hash) -> Self {
        Self {
            timestamp: Utc::now().timestamp(),
            prev_block_hash: prev_hash,
            data: data.to_owned().into(),
            nonce: 0,
        }
    }
}
```

A miner will run the hashing function, check if the hash is below the target, 
and if it isn't, increase the nonce (thereby changing the header) and
run the hashing function again, until it finds a winning hash (or until someone
else finds one before them).

In a real blockchain, the difficulty is adjusted dynamically, in order to
maintain the number of blocks produced per minute more-or-less stable: if
there's an increase in hashing power, the difficulty will be raised; if there's
a decrease, it will be lowered. In Bitcoin, the block-mining rate is 10 minutes.

In our case, the difficulty is a hard-coded constant.

We can now proceed to implementing Proof of Work!

```rust
extern crate crypto;
use crypto::digest::Digest;
use crypto::sha2::Sha256;
use num_bigint::BigUint;
use num_traits::One;

const DIFFICULTY usize = 5;
const MAX_NONCE: u64 = 1_000_000;

impl Block {
  fn try_hash(&self) -> Option<u64> {
      // The target is a number we compare the hash to. It is a 256bit binary with DIFFICULTY
      // leading zeroes.
      let target = BigUint::one() << (256 - 4 * DIFFICULTY);

      for nonce in 0..MAX_NONCE {
          let hash = calculate_hash(&block, nonce);
          let hash_int = BigUint::from_bytes_be(&hash);

          if hash_int < target {
              return Some(nonce);
          }
      }

      None
  }

  pub fn calculate_hash(block: &Block, nonce: u64) -> Sha256Hash {
      let mut headers = block.headers();
      headers.extend_from_slice(convert_u64_to_u8_array(nonce));

      let mut hasher = Sha256::new();
      hasher.input(&headers);
      let mut hash = Sha256Hash::default();

      hasher.result(&mut hash);

      hash
  }

  pub fn headers(&self) -> Vec<u8> {
      let mut vec = Vec::new();

      vec.extend(&util::convert_u64_to_u8_array(self.timestamp as u64));
      vec.extend_from_slice(&self.prev_block_hash);

      vec
  }

}

// This transforms a u64 into a little endian array of u8
pub fn convert_u64_to_u8_array(val: u64) -> [u8; 8] {
    return [
        val as u8,
        (val >> 8) as u8,
        (val >> 16) as u8,
        (val >> 24) as u8,
        (val >> 32) as u8,
        (val >> 40) as u8,
        (val >> 48) as u8,
        (val >> 56) as u8,
    ]
}
```

In order to perform the hashing we need the headers and the nonce as an array of
bytes, so we create a little helper function (`convert_u64_to_u8_array`) that
does just that (alternatively, you could decide to use the [byteorder
crate](https://docs.rs/byteorder/1.2.1/byteorder/index.html))

The `try_hash()` function iterates over all the possible nonces sequentially (the limit is
`MAX_NONCE`) and performs the calculation. It either returns the nonce or, if it
can't find any, `None`.

Notice that how this method uses the
[`num-bigint`](https://rust-num.github.io/num/num_bigint/index.html) crate for
handling large numbers, and bit-wise operations to create a target: since each
hex digit encodes 4 bits, we want to multiply `DIFFICULTY` by 4.

The `calculate_hash()` function is where the actual hashing happens. It retrieves
the headers, adds the nonce, and hashes the whole thing.

Let's update our `new` function to perform hashing when a new block is created:

```rust
impl Block {
    pub fn new(data: &str, prev_hash: Sha256Hash) -> Result<Self, MiningError> {
        let mut s = Self {
            prev_block_hash: prev_hash,
            nonce: 0,
            data: data.to_owned().into(),
        };

        s.try_hash()
            .ok_or(MiningError::Iteration)
            .and_then(|nonce| {
                s.nonce = nonce;

                Ok(s)
            })
    }
}
```

Notice that we introduced an error type, `MiningError`:

```rust
use std::error;
use std::fmt;

#[derive(Debug)]
pub enum MiningError {
    Iteration,
    NoParent,
}

impl fmt::Display for MiningError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            MiningError::Iteration => write!(f, "could not mine block, hit iteration limit"),
            MiningError::NoParent => write!(f, "block has no parent"),
        }
    }
}

impl error::Error for MiningError {
    fn description(&self) -> &str {
        match *self {
            MiningError::Iteration => "could not mine block, hit iteration limit",
            MiningError::NoParent => "block has no parent",
        }
    }

    fn cause(&self) -> Option<&error::Error> {
        None
    }
}
```

We now have a way to create blocks that are linked to a parent by way of a hash.

A pretty important missing piece is the first block, i.e. the one that has no
parent. In blockchain-speak, this is called a **genesis block**:

```rust
impl Block {
    // Creates a genesis block, which is a block with no parent.
    //
    // The `prev_block_hash` field is set to all zeroes.
    pub fn genesis() -> Result<Self, MiningError> {
        Self::new("Genesis block", Sha256Hash::default())
    }
}
```

### The Blockchain

The next step is creating a `Blockchain` struct:

```rust
pub struct Blockchain {
    blocks: Vec<Block>,
}

impl Blockchain {
    // Initializes a new blockchain with a genesis block.
    pub fn new() -> Result<Self, MiningError> {
        let blocks = Block::genesis()?;

        Ok(Self { blocks: vec![blocks] })
    }

    // Adds a newly-mined block to the chain.
    pub fn add_block(&mut self, data: &str) -> Result<(), MiningError> {
        let block: Block;
        {
            match self.blocks.last() {
                Some(prev) => {
                    block = Block::new(data, prev.hash())?;
                }
                // Adding a block to an empty blockchain is an error, a genesis block needs to be
                // created first.
                None => {
                    return Err(MiningError::NoParent)
                }
            }
        }

        self.blocks.push(block);

        Ok(())
    }

    // A method that iterates over the blockchain's blocks and prints out information for each.
    pub fn traverse(&self) {
        for (i, block) in self.blocks.iter().enumerate() {
            println!("block: {}", i);
            println!("hash: {:?}", block.pretty_hash());
            println!("parent: {:?}", block.pretty_parent());
            println!("data: {:?}", block.pretty_data());
            println!()
        }
    }
}
```

Ok, that's pretty cool. We can tie all this together in an example. In a
properly structured Rust crate, this would be the
`main.rs` file of a binary crate making use of the library crate:

```rust
// this would be the library crate
extern crate rusty_chain;

use std::process;

use rusty_chain::blockchain::Blockchain;
use rusty_chain::error::MiningError;

fn main() {
    println!("Welcome to Rusty Chain");

    run().
        unwrap_or_else(|e| {
            println!("Error: {}", e);
            process::exit(1)
        })
}

fn run() -> Result<(), MiningError> {
    let mut chain = Blockchain::new()?;
    println!("Send 1 RC to foo");
    chain.add_block("enjoy, foo!")?;

    println!("Traversing blockchain:\n");
    chain.traverse();

    Ok(())
}
```

In this silly example, we:

* create a blockchain
* create a block, with the string `enjoy, foo!`
* traverse and pretty print the contents of the block chain
