# Syncing with upstream

How to bring this fork up to date with its upstream skill repository and with
the open pull requests there. Nothing here compiles; the validators are plain
Node.js and the only thing run is their test suites.

## 0. Set up

```bash
git remote add up https://github.com/<upstream-owner>/security-audit-skill.git   # once
git fetch up
git rev-list --left-right --count up/main...main      # behind / ahead
git log --oneline up/main..main                       # this fork's own commits
```

Point the clone at the public-repo guard hooks if your setup has them
(`git config core.hooksPath <guard>/hooks`).

## 1. Upstream `main`

When `up/main` has moved:

1. Read every new commit's diff, not its title. Flag anything that weakens a
   validator (drops a check, adds a permissive fallback, loosens a schema),
   changes the findings or ledger contract, adds a network call to a validator,
   or grows `SKILL.md` past its budget (step 4).
2. Merge `up/main` into `main`. Our own commits win on conflict unless upstream
   fixes the same thing better; say which in the merge message.
3. Run step 5.

## 2. Open upstream pull requests

Upstream merges slowly, so most improvements sit in open PRs. Review all of them:

```bash
gh pr list -R <upstream-owner>/security-audit-skill --state open --limit 200 \
  --json number,title,author,additions,deletions
for n in $(gh pr list -R <upstream-owner>/security-audit-skill --state open --limit 200 --json number --jq '.[].number'); do
  git fetch -q up pull/$n/head:refs/pr/$n
done
```

For each PR:

- Read `git diff up/main...refs/pr/<N>`; judge what it does, not what its title says.
- Run its own tests in a throwaway worktree:
  `git worktree add /tmp/wt-<N> refs/pr/<N> && (cd /tmp/wt-<N>/skills/security-audit && node --test *.test.cjs)`.
- Check for conflicts with `main`: `git merge-tree --write-tree main refs/pr/<N>`.
- Group PRs that fix the same problem and pick the single best implementation.
- Verdict: **PORT** (correct, real, best of its group), **MAYBE** (name the
  check that would decide it), or **SKIP** (one line: wrong, duplicate,
  cosmetic, README/LICENSE churn, rebrand, scope creep, or weakens a validator).

Never port a change that makes a validator less strict on POSIX to gain another
platform, adds a fallback that skips `O_NOFOLLOW`/`O_NONBLOCK`, or loosens the
schema without a matching contract reason.

Record the PORT, MAYBE, and SKIP lists in the sync commit message.

## 3. Porting

- Apply each PORT with `git cherry-pick -n <sha>`, one at a time, and run the
  suite after each (step 5). Resolve conflicts by keeping both behaviours, not by
  picking a side: two validator fixes in the same function must both survive,
  with both PRs' tests passing.
- Take only `skills/` from a PR that also carries CI, plans, or README churn:
  `git diff <sha>^ <sha> -- skills/ | git apply -3 --index`.
- When two PRs interact (one adds a bound, another's test uses a value past it),
  fix the test to the combined contract rather than dropping either change.
- Credit every ported PR's author with a `Co-authored-by:` trailer and name the
  PR number in the commit body.

## 4. `SKILL.md` budget

`SKILL.md` loads on every trigger, and its body (after the frontmatter) must stay
under about 5,000 tokens, roughly 20,000 bytes at this file's ~4 bytes per token:

```bash
awk '/^---$/{c++;next} c>=2' skills/security-audit/SKILL.md | wc -c
```

A PR that adds more than a sentence or two to `SKILL.md` goes into a companion
file instead (`BUDGET.md`, `EXECUTION-SAFETY.md`, `SANDBOX-DOCKER.md` are the
pattern), with a one-line trigger in `SKILL.md` that says when to read it. Add
each new companion to the README file table.

## 5. Verify

```bash
cd skills/security-audit
node --test *.test.cjs                 # 0 failures; skips are Windows-only cases
grep -rn '<<<<<<<\|>>>>>>>' . ../../README.md   # no conflict markers
```

Then copy the skill into the installed location and confirm the copy matches:

```bash
diff -r skills/security-audit ~/.claude/skills/security-audit
```

## 6. Commit

Commit on `main`, one commit per sync. Subject `sync: port upstream PRs <list>` or
`sync: merge upstream main`. The body lists what was ported and from which PR,
the MAYBE/SKIP decisions in one line each, and any conflict resolved by
combining two changes. End with one `Co-authored-by:` trailer per ported author.
