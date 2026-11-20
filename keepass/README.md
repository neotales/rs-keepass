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

**Nest entries under a group.** `Group::add_group` and `Group::add_entry` return the new child, and
`find_entry` looks an entry up by its `id` after a round trip.

```rust
use std::io::Cursor;

use neotales_keepass::{Entry, Group, KdbxFile, KeePassDatabase, ProtectedValue};

let mut database = KeePassDatabase::default();
let work = database.root.add_group(Group::new("Work"));
let entry = work.add_entry(Entry::new("mail"));
entry.password = ProtectedValue::from_text("hunter2");
let id = entry.id;

let mut bytes = Vec::new();
KdbxFile::save_default(&database, &mut bytes, "vault password").expect("write vault");
let loaded = KdbxFile::open(Cursor::new(bytes), "vault password").expect("open vault");

let found = loaded.find_entry(id).expect("entry survived the round trip");
assert_eq!(found.title, "mail");
assert_eq!(loaded.root.groups[0].name, "Work");
```

**Attach a custom field.** `Entry::set_field` adds or replaces by key, and `Entry::field` reads it
back after the value has been written and re-parsed.

```rust
use std::io::Cursor;

use neotales_keepass::{Entry, Field, KdbxFile, KeePassDatabase};

let mut database = KeePassDatabase::default();
let entry = database.root.add_entry(Entry::new("mail"));
entry.set_field(Field::new("otp", "123456"));
let id = entry.id;

let mut bytes = Vec::new();
KdbxFile::save_default(&database, &mut bytes, "vault password").expect("write vault");
let loaded = KdbxFile::open(Cursor::new(bytes), "vault password").expect("open vault");

let field = loaded
    .find_entry(id)
    .and_then(|entry| entry.field("otp"))
    .expect("custom field survived the round trip");
field
    .value
    .with_revealed_text(|text| assert_eq!(text, "123456"))
    .expect("reveal field");
```

**Derive a TOTP code at a fixed time.** `Totp::generate` takes the Unix timestamp, so a caller owns
the clock; this is the RFC 6238 SHA-1 vector for time step 1, truncated to six digits.

```rust
use neotales_keepass::{OtpAlgorithm, OtpDigits, Totp, TotpParameters};

let parameters = TotpParameters {
    secret: b"12345678901234567890",
    algorithm: OtpAlgorithm::Sha1,
    digits: OtpDigits::Six,
    period: 30,
};

assert_eq!(Totp::generate(&parameters, 59), "287082");
```

**A wrong password is an error, not an empty vault.** `KdbxFile::open` authenticates the header
before decrypting, so a bad password fails closed.

```rust
use std::io::Cursor;

use neotales_keepass::{KdbxFile, KeePassDatabase};

let mut bytes = Vec::new();
KdbxFile::save_default(&KeePassDatabase::default(), &mut bytes, "vault password")
    .expect("write vault");

let opened = KdbxFile::open(Cursor::new(bytes), "not the vault password");
assert!(opened.is_err());
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

**Reveal inside a closure, not into a variable.** `with_revealed_text` hands the plaintext to a
callback and zeroizes its temporary string before returning, so no copy outlives the call.

```rust
use neotales_keepass::{Entry, ProtectedValue};

let mut entry = Entry::new("token");
entry.password = ProtectedValue::from_text("ghp_example_not_a_real_token");

let matches = entry
    .password
    .with_revealed_text(|text| text == "ghp_example_not_a_real_token")
    .expect("reveal inside the closure");
assert!(matches);
assert!(!format!("{entry:?}").contains("ghp_example_not_a_real_token"));
```

**Save what you did not understand.** Elements the model does not parse are retained, and
`KeePassDatabase::can_save_losslessly` says whether a save keeps the document intact. Use
`KdbxFile::save` when you need to choose the KDF, the outer cipher, compression or block size.

**Choose the writer settings explicitly.** `KdbxWriteOptions` starts from `Default`; `save` takes
caller-owned credentials and read limits as well as the options.

```rust
use std::io::Cursor;

use neotales_keepass::{
    KdbxCompression, KdbxCredentials, KdbxFile, KdbxReadLimits, KdbxWriteOptions, KeePassDatabase,
};

let database = KeePassDatabase::default();
let options = KdbxWriteOptions {
    compression: KdbxCompression::GZip,
    ..KdbxWriteOptions::default()
};

let mut credentials = KdbxCredentials::password("vault password").expect("password credentials");
let mut bytes = Vec::new();
KdbxFile::save(
    &database,
    &mut bytes,
    &mut credentials,
    options,
    &KdbxReadLimits::default(),
)
.expect("write vault");

let loaded = KdbxFile::open(Cursor::new(bytes), "vault password").expect("open vault");
assert!(loaded.root.entries.is_empty());
```

The companion crate [`neotales-keepass-platform`](https://github.com/neotales/rs-keepass/tree/dev/keepass-platform)
adds an optional Windows DPAPI provider and a page-aligned Unix `mlock` buffer. Neither is enabled
by default.

## Licence

[MIT](../LICENSE.md)
