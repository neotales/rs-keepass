# Releasing

Publishing runs from `release.yml`. It is driven by a tag and it authenticates
with a short-lived crates.io token obtained over OIDC. There is no
`CARGO_REGISTRY_TOKEN` secret in this repository.

## The version rule

The tag is `v` plus the version in the manifests. Tag `v0.3.1` publishes the
crates only if `keepass/Cargo.toml` and `keepass-platform/Cargo.toml` both say
`version = "0.3.1"`. The `version` job fails the run when they differ from each
other or from the tag, so a tag can never publish a version that differs from
what is in the tree.

This repository ships two crates that share the tag: `neotales-keepass` from
`keepass/` and `neotales-keepass-platform` from `keepass-platform/`.
`neotales-keepass-platform` depends on `neotales-keepass` by version as well as
by path, so the publish job uploads keepass first and keepass-platform second.

Publishing in the other order fails: crates.io rejects a crate whose dependency
version is not in the index yet. The dry runs go in the same order, and the
second one can still fail while the crates.io index has not caught up with the
keepass upload from the same run. That failure is index lag, not a broken
workspace: confirm with `cargo search neotales-keepass` and re-run the job once
the index lists the version. Do not change the dependency from a version to a
bare path to get past it, because a published crate cannot carry a path.

Bump the version, commit, push, then tag that commit:

```sh
cargo publish -p neotales-keepass --dry-run --locked
cargo publish -p neotales-keepass-platform --dry-run --locked
git tag v0.3.1
git push origin dev
git push origin v0.3.1
```

The one exception is a new crate's first version, which is uploaded by hand and
never from a tag. That is the next section.

## First publish of a new crate

crates.io trusted publishing cannot create a brand-new crate. The OIDC exchange
needs a crate that exists and a trusted publisher configured on it, so a cold
start over OIDC is rejected by crates.io. Do not try to build one.

The first version of a new crate is `0.0.0-alpha.0`. Not `0.0.0-alpha.1` and
not `0.1.0`. The first upload is manual, and a manual upload needs a version
nothing else has claimed.

The first upload goes through `/home/dev/work/rs/publish.sh <dir>`. That script
lives beside this folder, in `~/work/rs`, and is deliberately not copied into
the crate repository: it is how the first version of every crate is uploaded,
not part of any one crate.

1. Set `version = "0.0.0-alpha.0"` in `keepass/Cargo.toml` and
   `keepass-platform/Cargo.toml`, commit it, and leave the working tree clean.
   Do not publish from a tree with uncommitted changes.
2. Run the script with each crate's directory, keepass first because the
   platform crate depends on it:

   ```sh
   ~/work/rs/publish.sh keepass
   ~/work/rs/publish.sh keepass-platform
   ```

   The script reads the token itself and passes it to `cargo` through the
   environment. Do not put a token on the command line, in the repository, or
   in shell history.
3. Add the trusted publisher on crates.io for both crates, in each crate's
   settings: owner `neotales`, repository name, workflow filename
   `release.yml`, and an empty environment. Leave the environment field empty.
   The `publish` job in `release.yml` does not set an environment, and crates.io
   compares the two fields: an environment name here that the job does not set,
   or the reverse, fails the token exchange and nothing else fails with it. If
   someone later sets an environment name on crates.io, the job must set that
   exact name.
4. Bump to the next version and let tags take over. Do not run `publish.sh`
   for these crates again.

The release run for that first version fails, because the version it would
publish is already uploaded. The tag is provenance for the first version, not
its publish trigger. Every later version publishes from its `v*` tag.

Before the first manual upload, `cargo publish -p neotales-keepass-platform
--dry-run` also fails: the platform crate names `neotales-keepass` by version,
and that version is not in the index yet. This is the same dependency rule as a
real publish, not a packaging fault. It clears once keepass is uploaded.

If either crate name is taken on crates.io by someone else, stop. A rename is a
different repository setup, not a flag.

`cargo package` includes every tracked file, so `.github/` and `RELEASE.md`
ship inside the published crate unless the manifest excludes them. That is
harmless. Exclude them if package size matters. The README style guide lives
at `~/work/rs/README.STYLE.md` and is not part of a crate.

## Later releases

1. Bump the version in both manifests and commit it. `Cargo.lock` is committed
   on purpose, so run `cargo build` and commit the updated lockfile with it.
   The lockfile is what the scheduled audit checks.
2. Push the commit.
3. Tag it `vX.Y.Z` and push the tag.
4. The workflow runs the checks, then `cargo publish --dry-run` of both crates,
   then publishes keepass, then keepass-platform, with a token minted for that
   run.

A manual run, `workflow_dispatch`, runs the checks and the dry runs only. It
never publishes, whatever branch it is started from.

After the manual first upload is done, switch each crate to Trusted Publishing
Only mode on crates.io. That rejects token publishes everywhere, so a token
cannot publish these crates later.

## What is not allowed

- `pull_request_target` and `workflow_run` are not used anywhere, and must not
  be added. Both can run untrusted code with publishing credentials in scope.
- No long-lived crates.io token in GitHub secrets for this workflow. The job
  reads `id-token: write` and nothing else.
- No publishing from a branch. The tag check exists so the artifact has a name.
- No republishing a version. crates.io rejects it. If a publish fails after the
  upload was accepted, bump the patch version and release again.

## Failure modes

| Symptom | Cause |
| --- | --- |
| `version` job: tag does not match `vX.Y.Z` | A manifest version and the tag disagree, or the two manifests disagree. Delete the tag, fix, re-tag. |
| Publish job: OIDC exchange fails | The crate does not exist yet, or the trusted publisher does not match: wrong repository, wrong workflow filename, or an environment name that is set on one side and not the other. |
| Publish job: `neotales-keepass-platform`: no matching package named `neotales-keepass` | The index has not caught up with the keepass upload from the same run. That is index lag: confirm the version with `cargo search neotales-keepass` and re-run the job. |
| Publish job: version already uploaded | The version was published before, usually by the manual first release. Bump the version. |
| Publish job: `cargo publish` rejects the token | Trusted Publishing Only mode is on and the crate has no trusted publisher configured. |
