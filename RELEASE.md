# waev:outpost release guide

New versions ship as the `waev.outpost` UI plugin: one immutable, version-matched wheel in [the plugin releases](https://github.com/Treehouse-00/waev-outpost-plugin/releases). Published versions and their exact download URLs are retained for catalogue installs and rollback. The standalone channel is frozen at **v0.9.394**; new console archives are rejected.

## Prepare a candidate

Release commands commit, tag and publish. Run them only when those actions are explicitly authorized. Ordinary development and validation use the non-publishing commands below from the private source repository:

```sh
# Use the exact Node in .node-version and its bundled npm.
# Supply MOTION_TOKEN through your environment/secret manager first.
npm run setup
npm run verify
```

An already licensed tarball can instead be supplied through `MOTION_PACKAGE_TARBALL`; its bytes must match the committed lockfile integrity. Setup uses a disposable install/cache directory, preserves tracked manifests and existing dependencies on failure, and never requires injecting a token into source. Do not use the former inject/restore process or put credentials in command examples.

`verify` runs types, lint, correctness, distribution/installer/release/catalogue tests, shell/workflow analysis, dependency audit, both production asset bases, exact wheel verification, built-plugin browser tests and isolated performance budgets. It emits private evidence and an SBOM into `plugin-dist/`. These records are not certification or a public attestation.

The standalone validation bundle goes to `frontend/out/`; the plugin UI goes to `frontend/out-plugin/`, with its wheel in root `plugin-dist/`. Ordinary verification does not modify committed private `frontend/dist/`. A plugin uses absolute `/plugins/waev.outpost/` asset URLs, including when selected as Repeater's default frontend at root.

## Authorized release sequence

1. Finish, test and independently review the intended source changes. Commit them separately through the normal review process. Release requires a clean working tree and refuses to stage unrelated source.
2. Write and commit the complete operator-facing `CHANGELOG.md` section for the intended next version. `frontend/package.json` owns the application version; the root package has no separate version.
3. Confirm hosted source checks, administrator-controlled verification refs, protected credential environments and the two separately scoped GitHub Apps below are configured and tested. Obtain authorization for the concrete release candidate.
4. From that clean, reviewed checkout, run the release command. `patch` is the default; `minor` and `major` are the only other supported bumps. Prereleases are not supported.

```sh
./release.sh "fix: concise release subject" patch --targets=plugin
```

The script requires a clean checkout at the exact current `origin/main`. It prepares a local `release/vVERSION` branch, bumps the app/lock version, supplies a minimal changelog fallback only if the intended heading is absent, and runs the entire verification path against that final version. Each distinct asset base is built once during this gate. It copies the verified standalone output into private dist, checks the plugin's evidence again, and stages **only** the app manifest, lockfile, changelog and private dist. After its local release commit, it binds evidence to that commit without accepting changed source inputs. Preparation does not push a branch or create a tag.

With separate authorization, push the release branch and open its version PR to `main`. An administrator reviews the exact PR head, including workflows and dependency inputs, then promotes that commit to `ci/verified/<full-head-sha>` for the required Linux verification. These refs are administrator-created and immutable. Inspect the successful `verify-source.yml` run on that exact SHA; the status name alone does not establish workflow provenance. Independent PR review and current-base source verification remain required before merging.

After the version PR is merged, use a clean checkout of current `main` and explicitly authorize publication:

```sh
./release.sh --publish --targets=plugin
```

Publication does not bump or commit source. It reruns verification, validates the artifact evidence, sets `PUBLISH_TARGETS=plugin`, rechecks current `origin/main`, and pushes only the version tag. An existing remote source tag is always refused. An unpublished local lightweight tag of the exact current source may be reused after full verification; other local tags are preserved and refused. Never disable source protections to perform a direct main push.

Submission is not proof of publication. Main can advance while a tag is pushed or while its workflow is queued. The script checks again after submission and reports an observed advance as a failure, preserving the immutable source tag. The workflow independently checks current main before public writes. If it rejects that tag as stale, prepare a new reviewed version; never move or delete the remote source tag.

Failures preserve the local candidate for inspection. Inspect the version, changelog, Git status, remote tag and hosted outcome before retrying: a failed network response does not establish that the push failed remotely. Do not blindly rerun a version bump over a failed candidate or reset user work.

## Source and publication gates

`verify-source.yml` provides the required check context **Source verification**, bound to the GitHub Actions App. It runs on protected source branches and administrator-promoted exact commits. Trusted builds use `.console.env` for `MOTION_NPM_TOKEN_V2`, passed to bootstrap as `MOTION_TOKEN`. Ordinary PR events, including forks, do not enter this credential-bound workflow; their required check remains unsatisfied until a maintainer reviews and promotes the exact commit in the private repository. Native environment reviewers are unavailable on the current private-account plan, so administrator-only ref creation is the explicit credential-approval boundary. Required PR review, immutable verification refs and environment allowlists must be enforced in hosted settings; YAML alone does not enforce them. Check-name binding identifies the GitHub App, not the workflow: verify the run path and SHA before merge.

The private `build-ui.yml` validates the intentionally committed dist contract and helper behavior; its public metadata-only path rejects tracked dist and remains independent of private source tooling. `build-ui-artifact.yml` is the private manual candidate lane.

For an administrator-promoted `ci/verified/<sha>` ref, `verify-source.yml` also supports direct manual dispatch with no inputs. It rechecks event/ref/SHA before credentials and reports the required check only after trust and full verification succeed. Ordinary branch dispatches fail before credential access. Both triggers use the same canonical workflow/check; the manual artifact workflow does not certify this gate. Wait for an active push run or rerun an existing failed run instead of starting concurrent verification for the same ref. The private hosted-controls runbook records the command and hosted evidence.

A version tag starts `sync-to-public.yml`. It validates the tag/version and trusted main ancestry, runs the same complete source gate, and verifies the exact wheel, metadata, RECORD, safe paths, allowed assets and secret exclusion **before** public writes. It promotes those same checked wheel bytes. Private provenance records the source revision/content digest, package/lock digests, toolchain, version and wheel SHA-256. The SBOM inventories locked production dependencies, not necessarily every byte retained by tree shaking.

Only allowlisted installer files, metadata, public documentation and `docs/images` cross into the public snapshot. Other internal records and source stay private. Generated UI files belong in the wheel, not public Git history. The supported public snapshot retains one root commit/main branch, the current tag/latest release, and previous immutable plugin releases/tags. Neither a branch push nor a local build publishes a release.

## Credentials and least privilege

Artifact publication uses **`PUBLIC_REPO_APP_ID`** and **`PUBLIC_REPO_APP_PRIVATE_KEY`** in the `publication` environment, restricted to administrator-created `v*` tags. This environment also needs `MOTION_NPM_TOKEN_V2` for its independent source build. Install the publication App only on `Treehouse-00/waev-outpost-plugin`, with Contents write and Workflows write; the latter is needed because the metadata snapshot contains its validation workflow. Metadata read is implicit. A pinned token action mints a short-lived installation token after validation and revokes it at job completion. Retire the old publication PAT after the App cutover is proven. Keep publication credentials out of the build-only `.console.env` environment and repository-level secrets.

Catalogue publication uses the separate `openhop-catalogue-publisher` App (ID `4844131`), installed only on `openhop-dev/openhop-plugin-catalogue`, with Contents read/write, Pull requests read/write and Metadata read-only. Keep **both required catalogue environment secrets** `OPENHOP_CATALOGUE_APP_ID` and `OPENHOP_CATALOGUE_APP_PRIVATE_KEY` in the source repository's `catalogue` environment, restricted to `main`. Remove their repository-level copies after provisioning and verifying the environment. Never print either App's private key. The catalogue workflow validates downloaded release bytes and the catalogue before minting its scoped token.

After public sync, `propose-plugin-catalogue.yml` downloads the published wheel, validates its metadata/RECORD and hashes those exact bytes. It opens or updates a version-specific proposal. The catalogue-owned certification checks and review decide approval; publication success alone does not make a version approved. Retries preserve a human-selected draft/ready state. The proposal workflow can be run manually for an existing release tag without rebuilding its wheel; a tag without a plugin wheel produces no proposal.

## Failures, retries and rollback

| Condition | Required response |
| --- | --- |
| Source verification fails | Repair the concrete failure and rerun the shared gate. Review a failed local version bump before preparing another candidate. Never bypass the gate with a hand-pushed release tag. |
| `origin/main` moves before tag submission | Reconcile the reviewed source with origin, resolve conflicts, rerun verification and obtain the normal release authorization. The preflight does not push a tag on this failure. |
| Main advances during or after tag submission | Inspect the hosted outcome. A stale source tag rejected by the workflow remains immutable; prepare a new reviewed version. A local post-push check cannot make the distributed operation atomic. |
| Tag push fails or its outcome is uncertain | Inspect the remote tag and workflow. If absent remotely and its local lightweight tag still points to exact current main, `--publish` can re-verify and retry the same tag. If present remotely, follow the existing workflow's outcome; never replace or delete that source tag. |
| Locked install returns an authentication error | Refresh the credential at the source supplying the process/CI environment, or use an integrity-matching licensed tarball. Do not edit tracked package URLs, discard the lockfile or delete working dependencies to recover. |
| Release version already published | Stop. Published tags/assets are immutable. Use a new reviewed version for corrections. |
| Unpublished draft from an interrupted run | The workflow stops before public writes. Inspect the draft's target, assets and originating workflow. Only after confirming it belongs to the failed candidate and was never published may an authorized maintainer delete that draft, keep its tag and rerun the same source tag. No automatic draft deletion is performed. |
| Orphan public tag with no release | The supported retry path can remove this unusable tag and recreate it from the same validated candidate. A published version is never treated as an orphan. |
| App permission failure | Check the protected environment secrets, exact installation repository and Contents/Workflows permissions. Do not substitute a broad personal token. |
| Catalogue digest mismatch | Compare the proposed digest against the downloaded immutable wheel. Repair the proposal/evidence; never overwrite the wheel to make it match. |
| Installed update fails | Use the Repeater plugin manager to reinstall the previously verified retained wheel, then verify its version, plugin path and default frontend setting. Preserve configuration and user data. |

## Operator installation

Use **System → Plugins → Catalogue → waev:outpost → Install**, then confirm **UI READY** and open the UI. To make it the default, select it under **System → Configuration → Access → Web Options → Web Frontend**.

Manual **Install wheel** accepts the single `.whl` from a published release. A fresh manual install does not inherit catalogue tracking; upload a newer wheel or adopt a catalogue install. See [INSTALL.md](INSTALL.md) for prerequisites, approved updates and recovery. Existing standalone users should follow [the migration guide](INSTALL.md#migrate-from-the-standalone-install); preserve the old files/configuration while verifying the plugin. `manage.sh upgrade` stays on the final standalone channel.

No cleanup or verification task implicitly authorizes a real Repeater restart, radio transmission, hosted settings change or publication.
