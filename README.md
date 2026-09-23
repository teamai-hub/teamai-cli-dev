**English** | [中文](./README.zh-CN.md)

# teamai-cli-dev — Dogfood Team Repo

This is the **dogfood team repository** for [teamai-cli](https://github.com/Tencent/teamai-cli)
contributors. Point your local CLI build at this repo to exercise the real
`init` → `pull` → `push` flow end to end while developing the CLI itself.

This is **not** a template to copy. To bootstrap your own team repo, start from
[`teamai-hub/teamai-template`](https://github.com/teamai-hub/teamai-template) instead.

## Usage

From your teamai-cli clone (not `teamai init .`):

```sh
npm run build && npm link
teamai init https://github.com/teamai-hub/teamai-cli-dev --scope project --role dev
teamai pull
git status   # nothing under .teamai/ or tool dirs should be staged for teamai-cli
```

- Init the **canonical hub URL above**, not a personal fork.
- `Push failed (you can push manually later)` during member registration is
  **expected** without write access. Local config is still saved and `teamai pull`
  still works.

## What's in this repo

### Team skills

**None yet.** Team skills are executable instructions and must pass team review
before they are distributed here. They will be added through the normal
`teamai push` review flow once approved.

The CLI's **built-in skills** are independent of this repo — they ship inside the
`teamai-cli` npm package and deploy automatically on every `init` / `pull`:

- `team-wiki-codebase` — always deployed.
- `teamai-share-learnings` — deployed when recall is enabled.

So after `teamai pull` you will have the built-in skills above, and **zero** team
skills from this repo until reviewed ones are added.

### Access

| Who | Hub repo access | In team digest |
| --- | --- | --- |
| Contributors | read | no |
| Collaborators (after a few PRs) | write, `main` protected | yes |

`digest` / `dashboard` read `stats/`, `sessions/`, and `members/` from the team
repo. Those files are written via git, so **no write access ⇒ not in team stats**.

## License

[MIT](./LICENSE).
