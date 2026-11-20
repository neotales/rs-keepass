# neotales-keepass

Reads, writes and edits KeePass KDBX vaults without a UI, a clipboard or a platform prompt.

By [x0craft](https://github.com/x0craft).

[![Neotales](https://github.com/neotales/brand/raw/main/assets/logo.png)](https://github.com/neotales/rs-keepass)

<!-- badges: crates.io, docs.rs, CI. Add once the first version exists. -->

## Install

```sh
cargo add neotales-keepass
```

## What you can do 🔐

**Round-trip a vault in memory.** `save_default` writes KDBX 4.1 with Argon2id and AES-CBC;
`open` reads KDBX 3.0 through 4.1 and checks the header, HMAC and block checksums before decrypting.

```rust
use std::io::Cursor;

use neotales_keepass::{Entry, KdbxFile, KeePassDatabase, ProtectedValue};

let mut database = KeePassDatabase::default();
let entry = database.root.add_entry(Entry::new("mail"));
entry.username = "x0craft".into();
entry.password = ProtectedValue::from_text("correct horse battery staple");

let mut bytes = Vec::new();
KdbxFile::save_default(&database, &mut bytes, "vault password").expect("write vault");

let loaded = KdbxFile::open(Cursor::new(bytes), "vault password").expect("open vault");
assert_eq!(loaded.root.entries[0].username, "x0craft");
```

**Open with a key file.** `KdbxCredentials::key_file` builds the composite key from a key file;
`load` takes caller-owned streams and an explicit resource policy, so a caller decides what the
input may cost.

```rust,no_run
use std::fs::File;

use neotales_keepass::{KdbxCredentials, KdbxFile, KdbxReadLimits};

let key_file = std::fs::read("vault.key").expect("read key file");
let mut credentials = KdbxCredentials::key_file(key_file).expect("parse key file");
let vault = File::open("passwords.kdbx").expect("open vault file");
let database =
    KdbxFile::load(vault, &mut credentials, &KdbxReadLimits::default()).expect("load vault");

assert!(!database.root.entries.is_empty());
```

**Read a secret only when you need it.** `ProtectedValue` does not implement `Display`, its `Debug`
output is redacted, and a plaintext copy arrives in a `SecretString` that is zeroized on drop.

```rust
use neotales_keepass::{Entry, ProtectedValue};

let mut entry = Entry::new("mail");
entry.password = ProtectedValue::from_text("hunter2");

assert_eq!(entry.password.reveal_text().unwrap().as_str(), "hunter2");
assert!(!format!("{:?}", entry.password).contains("hunter2"));
```

**Save what you did not understand.** Elements the model does not parse are retained, and
`KeePassDatabase::can_save_losslessly` says whether a save keeps the document intact. Use
`KdbxFile::save` when you need to choose the KDF, the outer cipher, compression or block size.

The companion crate [`neotales-keepass-platform`](https://github.com/neotales/rs-keepass/tree/dev/keepass-platform)
adds an optional Windows DPAPI provider and a page-aligned Unix `mlock` buffer. Neither is enabled
by default.

## Licence

[MIT](../LICENSE.md)
