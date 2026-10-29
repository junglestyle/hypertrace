<!-- junglestyle/.github code-standards/README.md at b3d3bb1eb1b997100738fb5bb9dd2a2b9f45a9d4. Change it there, then run make sync-standards. -->
# Standards

Each principle from [PRINCIPLES.md](https://github.com/junglestyle/.github/blob/main/PRINCIPLES.md) turned into rules you can check in review, arranged by where a reviewer looks. When they pull against each other, pick what works and is easiest to undo.

Each repo keeps this file unchanged as `docs/org-standards.md`. It lists the topics it also follows, such as `python`, in `.org-standards` at its root, and keeps each topic's file here unchanged as `docs/org-standards-<topic>.md`. Every copy is headed by the commit it came from. The repo puts its own rules and examples in `docs/standards.md`, under the heading here they belong to, or under one of its own when none fits; where that differs from a copy, the repo's file wins and names the rule it overrides. Where the repo doesn't follow a copy yet, its `docs/standards.md` says so under Known deviations, as debt to pay down. The repo's checks fail if a copy was edited by hand, this directory has moved on, or a copy's topic isn't listed; resyncing is one command.

## Dependencies

- Use the stdlib first, then what's already in the stack, then the OS, the database, and the init system. A new dependency comes last and gets a one-line reason in the commit.
- Lean on the database's transactions, the OS's process model, and the supervisor's restarts before writing your own.
- A new language, datastore, framework, or runtime gets an ADR saying what it does that the current stack can't.

## Versions and pins

- A deployment pins what it runs to a commit or an image digest. A tag can move, so it can name a pin but isn't one.
- Everything else a build or deployment pulls in is pinned as below. Where a topic file says more, it applies too.

  | What | Pinned to |
  | --- | --- |
  | Libraries | Lower bounds in the manifest, exact versions in the committed lockfile |
  | Container images | A digest where a deployment runs it. [containers.md](https://github.com/junglestyle/.github/blob/main/code-standards/containers.md) covers tags in compose files |
  | OS packages | The distribution release, named by an image's tag or a VM's cloud image. Packages within it, tools included, aren't pinned, and a host a deployment manages installs security updates automatically |
  | Tools and Ansible collections | An exact version or a commit |
  | Other downloads | A version or revision, plus a checksum when the source isn't signed |
  | GitHub Actions | A major tag for `actions/*`. Any other action is pinned to a full commit SHA, with a comment naming its version |
  | These standards | `main`, so a repo's checks fail when the standards move on |

- A tool run through a launcher, such as `uvx --from shellcheck-py==0.11.0.1 shellcheck`, is pinned by its own exact version. The launcher isn't pinned for it, so a repo that isn't Python doesn't take on a Python toolchain pin just to run its tools.
- A release is an annotated `vX.Y.Z` tag on main. A tag in any other form isn't a release.
- Versions follow semver over the interface the repo's architecture names: a published schema or views, an API, or output another program reads. A breaking change to it bumps the major version. Another scheme needs an ADR.
- A repo makes its first release with the first breaking change to an interface that something outside the repo uses. Until then, its manifest's version is `0.0.0`.
- From then on, the release commit sets the manifest's version, and CI fails a `vX.Y.Z` tag that doesn't match it. Nothing else writes the version by hand: a lockfile records it, and code that reports it reads it from the package's metadata.
- A versioned data format or policy, such as a migration or a prompt policy, carries its own integer version, separate from releases.

## Structure

- No plugin systems, factories, registries, base classes, config frameworks, or extension points without two real cases that need them today.
- One code path that the CLI, API, and anything else share.
- A rule has one executable implementation.
- Two blocks that look alike but would change for different reasons stay separate. Duplication is cheaper than the wrong abstraction.
- Extract on the third real use, not the second similar-looking one. A critical invariant is the exception: two implementations of it are already one too many.
- Layers talk through explicit interfaces (an API, a schema, a file format) and don't know each other's internals. The repo's standards name its layers and what each one mustn't know.
- Flat over nested: early returns, shallow control flow, one module per concept.
- No import-time side effects and no hidden globals.
- Delete code that nothing calls. Git keeps it.
- Before removing something you didn't write, find out why it's there. The commit says what it was for.
- Leave each file you touch a little cleaner, within the change's scope.

## Errors and recovery

- Catch only what you can handle, and say what you did with it. Errors never pass silently.
- An impossible state raises. It isn't logged and continued.
- Prefer state the system converges to (expired leases requeue, stale locks clear) over steps someone must remember.

## Config

- No option, flag, or parameter that nothing sets.
- No magic default that differs by environment.

## Comments and names

- Names say what. Comments say why.
- Keep a comment only if a maintainer would lose a fact that the code, types, and tests don't already state. No comments that restate the code, and no commented-out code.
- The facts worth a comment are constraints the code can't show: concurrency and lifecycle rules, security boundaries, protocol quirks, and the tradeoff behind a non-obvious choice.
- A known limit goes in the docs or one comment. A workaround is fine when it's small and says what it works around.
- Project names are dumb and Linuxy: short, lowercase, nothing producty.

## Testing

Few, focused tests of consequential behavior, at boundaries that stay put.

- Test what a user would notice: the behaviors the spec promises, recovery after a crash, and the security boundaries.
- Don't test plumbing, getters, or internal call order. Don't mock half the system to prove a call happened.
- Don't pin prose: check what a message means, not its words. Output another program reads is interface, and a test may pin it.
- Before adding a test, ask what realistic regression it catches and whether it survives a refactor. If the answer is weak, skip it.

## Running it

- `make check` runs every check CI runs, offline. The one exception is `make standards`, which checks the org copies against this directory and needs the network; CI runs it as its own job. `make sync-standards` resyncs the copies, and following another topic means adding it to `.org-standards` and running it.
- Installing takes as few maintained scripts as the job needs. Setup steps don't get scattered across docs.

## Privacy

- Never commit real hostnames, addresses, tokens, personal context, or deployment config. In docs, name roles, not machines: "the NAS" or "a GPU host".
- Commit the shape of config, never its values. `.env.example` holds it unless the repo's standards name another place.
- The values go in the project's deployment repo (see [Deployment repos](#deployment-repos)).
- Examples and test data use made-up values: addresses in the private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, and `fd00::/8` for IPv6) or the documentation ones (`192.0.2.0/24`, `2001:db8::/32`), names under `example.com`, and placeholders for secrets. A public service may appear under its real name and address, but what identifies the owner on it, such as a topic or an account, is a placeholder: `https://ntfy.sh/example-endpoint-0000000000`.

## Secrets

- An app reads its secrets from the environment or a file outside the repo. Only a deployment repo commits them, and only encrypted with sops and age. Another tool needs an ADR.
- `.sops.yaml` at the root encrypts files named `<name>.sops.yaml` with `encrypted_regex: '.*'`. Without a selector, sops leaves keys ending in `_unencrypted` in plaintext. With `encrypted_regex`, it leaves comments in plaintext, so a `<name>.sops.yaml` file has none. `secrets.example.yaml` at the root lists every encrypted key with a placeholder value, and holds any note about one.
- The age private key never sits on disk. sops reads it from the password manager through `SOPS_AGE_KEY_CMD`.
- Edit with `sops edit`, so no decrypted copy is written into the repo. `make check` fails unless each `*.sops.yaml` file has a `sops:` block and no plaintext comment, and every scalar outside the block, at any depth, is `ENC[…]`, an empty string, or null. sops leaves only those two unencrypted. It encrypts `false` and `0` like any other value.
- A secret written to a host is readable only by its owner, and never shows up in logs or command output.
- A secret staged on the controller, such as a rendered file waiting to upload or a capture an upgrade carries across, lives outside the repo and is readable only by its owner. The run removes it when it's done. One kept so a failed run can resume is named in the playbook's header, with what removes it.
- A leaked secret gets rotated. Deleting it from the repo doesn't help, because git keeps it.

## Docs

- The README says what the project is and how to run it, in that order.
- `AGENTS.md` at the root tells an agent what to read before changing the repo (PRINCIPLES.md, the copies of these standards, `docs/standards.md`) and to run `make check` before committing.
- Docs describe the behavior a user can observe (a default, a state name, an env var), so a reader never has to open the code to learn what the project does. They don't carry a second copy of the logic.
- `docs/SPEC.md` says what the product is meant to do. It opens with the problem: who has it, and what goes wrong for them today. Its requirements follow, grouped under a heading per area, each a bullet that opens with an ID and a name, such as `**QUERY-3: Series listing.**` An ID names one requirement for good: it isn't renumbered or reused. What the spec assumes without having checked goes under Unknowns, and what isn't decided yet under Open questions. [examples/SPEC.md](https://github.com/junglestyle/.github/blob/main/code-standards/examples/SPEC.md) shows the shape.
- `docs/ROADMAP.md` says the order it gets built in: numbered slices, each usable on its own. A slice opens with `Status:`, `planned` or `done`, and the change that makes its done-when pass sets it to `done`, so the next slice is the first one planned. Then it says what a user would notice, `Delivers:` lists the requirement IDs it delivers instead of restating them, and `Done when:` says what someone can demonstrate. A done slice stays where it is. The roadmap has no dates, checkboxes, or progress table; git says when. [examples/ROADMAP.md](https://github.com/junglestyle/.github/blob/main/code-standards/examples/ROADMAP.md) shows the shape.
- A deferred requirement stays in the spec and gets a later slice in the roadmap, not a stub in code. When the roadmap changes, check the spec's IDs against its `Delivers:` lines: a requirement no slice delivers is one the roadmap defers on purpose.
- A decision that's hard to undo gets an ADR in `docs/stack.md`.

## Deployment repos

Deployment facts go in a parallel private repo, `<name>-deployment`, kept in the deployer's personal GitHub account, not the org. It holds the values the project repo can't (addresses, scope, secrets, release pins) and the automation that applies them. The project repo's architecture names what it owns. A repo that deploys shared infrastructure rather than one project is a deployment repo with no project repo, named for what it deploys. These rules replace the general ones they name, and everything else applies as written, including the copies of these standards, `AGENTS.md`, and `make check`.

- **Privacy:** hostnames, addresses, and deployment config belong here, since holding them is its job. Secrets follow [Secrets](#secrets), and real scan output and user data are never committed.
- **Docs:** it has no spec or roadmap. The project repo's spec says what it holds, and its roadmap orders the work. Its README says what it deploys, links the project repo's section naming what it owns, then covers first deploy, upgrades, and secrets. A decision about the deployment itself (hosts, hypervisor, layout) gets an ADR in its `docs/stack.md`. A decision about the product belongs in the project repo. Its `docs/standards.md` holds rules about the deployment itself; the project repo's holds the rest. A deployment repo with no project repo says in its README what it owns, and keeps its planned work in its own `docs/ROADMAP.md` and all its rules in its own `docs/standards.md`. Its roadmap's slices name the part of that ownership they deliver, in place of spec requirements.
- **Testing:** its playbooks, roles, and templates have no tests. Before every apply, a dry run against the real hosts (`--check --diff` in Ansible) shows what will change. Code it carries that does more than declare, such as a custom module, gets tests at its boundaries like any other code.
- **Running it:** `make check` runs the deployment tool's offline checks. An Ansible repo lists `ansible` in `.org-standards`, and a repo with scripts lists `shell`.

## Prose

For docs, commit messages, and PR descriptions. Follow the rules in
[agent-style](https://github.com/yzhao062/agent-style/blob/7da99e0b0610aca4e4d340351bd802bfd448c894/RULES.md)
but skip its House Style section, which is for academic papers. These rules take precedence:

- Describe the project as it is meant to be. No dates, approval trails, or "replacing X"; history lives in git.
- Link instead of restating. Each fact has one home, the same as in code.
- Arrange sections in the order a reader needs them.
- Error messages, log lines, and CLI output are interface. Changing their wording is an interface change, not a style edit.
- Contractions are fine, and headings use sentence case.
- "X, not Y" is fine where it prevents a real confusion.
- When writing up a problem, separate evidence, inference, and unknowns, and label which is which.
