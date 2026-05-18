# Contributing

This repo distributes Agent Skills for [ish](https://ishlabs.io). Where to send changes depends on what you want to change.

## Fixing the `ish` skill

The contents of `skills/ish/` are generated from the `ish-cli` repo and overwritten on every CLI release. Edits made here will not stick.

- Open issues and PRs against <https://github.com/ishlabs/ish-cli>
- Skill source: [`src/lib/skill-content.ts`](https://github.com/ishlabs/ish-cli/blob/main/src/lib/skill-content.ts)

## Proposing a new skill

Open an issue here describing the skill and the workflow it would automate. If accepted, the maintainers will scaffold a new `skills/<name>/` directory and either:

- Add it to the generation pipeline in `ish-cli` (for ish-owned skills), or
- Accept hand-authored content via PR here (for adjacent tooling).

## Editing repo plumbing

PRs welcome for anything outside `skills/<name>/`:

- `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `LICENSE`
- `.claude-plugin/marketplace.json`
- `.github/workflows/`
- `.gitignore`

## Running the validator locally

CI lints frontmatter on every PR — see [`.github/workflows/validate.yml`](.github/workflows/validate.yml). To smoke-test the repo structure locally without pushing:

```bash
npx -y skills add . --list
```

This scans the repo offline and prints every discoverable skill. If it lists what you expect, the structure is sound. The frontmatter lint runs in CI; the workflow file is short and self-contained if you want to read or copy it.

## Reporting bugs

| Bug is in | Open issue at |
|-----------|---------------|
| The `ish` CLI binary, backend, or skill content | <https://github.com/ishlabs/ish-cli/issues> |
| Install commands, marketplace manifest, CI, README accuracy | This repo |

## License

By contributing, you agree your contributions are licensed under the [MIT License](LICENSE).
