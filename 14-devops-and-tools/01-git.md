# Git

> **Tier:** T1 (know cold) · **Related:** [GitHub and workflows](02-github-and-workflows.md), [CI/CD and deployment](04-ci-cd-and-deployment.md), [Code quality and agile](../13-architecture-and-practices/10-code-quality-and-agile.md), [Legacy codebases and upgrades](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [Web security and OWASP](../04-web-fundamentals/07-web-security-owasp.md)
> **On your resume:** Git and GitHub in your skills list; daily work in client repositories at Devntech (legacy Rails monoliths, the multi-tenant platform migration), and earlier at PureLogics and RemoteDev.

## Quick revision
- Git stores **snapshots**, not diffs: each commit points to a tree (the full project state), its parent commit(s), the author, committer and message.
- Objects (blobs, trees, commits, tags) are **content-addressed** by hash (SHA-1 by default; SHA-256 repositories exist). Change anything and the hash changes.
- Commits form a **DAG**. A **branch** is a movable pointer to a commit; **HEAD** points to the current branch, or directly to a commit in detached HEAD.
- Three areas: the **working tree** (your files), the **index** or staging area (the next commit), the **repository** (`.git`, committed history).
- `git add -p` stages hunk by hunk; `git diff` compares working tree to index, `git diff --staged` index to the last commit.
- `git switch` (branches) and `git restore` (files) split up the overloaded `git checkout` (Git 2.23+).
- **Fast-forward** merge just moves the pointer; a **merge commit** has two parents (a three-way merge from the merge base).
- Conflicts: resolve the `<<<<<<<` / `=======` / `>>>>>>>` blocks, `git add`, then `git commit` or `git rebase --continue`; `--abort` backs out.
- **Rebase** replays your commits onto a new base as new commits (new hashes) for linear history. **Golden rule:** never rebase commits other people already have.
- Merge keeps true history, rebase linearizes it, squash lands a branch as one commit.
- `reset --soft` moves the branch and keeps changes staged, `--mixed` (default) keeps them unstaged, `--hard` discards them. `revert` adds a commit that undoes another; `restore` changes files, not history.
- Shared history: `revert`. Local, unpushed history: `reset`, `commit --amend`, interactive rebase.
- `git stash` shelves uncommitted work (`-u` includes untracked files); `pop` applies and drops, `apply` keeps it.
- `git cherry-pick <sha>` copies a commit onto the current branch (hotfix backports).
- `git reflog` records where HEAD and branches have been (local, kept about 90 days by default): the way to recover "lost" commits.
- `git fetch` only downloads and updates remote-tracking branches; `git pull` is fetch plus merge (or rebase).
- `git push --force-with-lease` refuses to overwrite remote commits you have not seen; `--force` overwrites anything.
- Tags mark releases; annotated tags (`git tag -a v1.4.0 -m ...`) carry metadata; push them explicitly.
- `.gitignore` only affects untracked files; `git rm --cached` stops tracking a committed one.
- `git bisect` binary-searches for the commit that introduced a bug; `git bisect run <test>` automates it. `git blame` finds who last changed a line, so you can find the PR that explains why.
- Commit messages: imperative subject ("Add bill reminders") of about 50 characters, blank line, a body explaining why; Conventional Commits (`feat:`, `fix:`) where the team uses them.
- A committed secret is compromised: rotate it first, then rewrite history with `git filter-repo` if needed.
- Hooks (`pre-commit`, `commit-msg`, `pre-push`) run local checks; share them with husky, lefthook or pre-commit; CI remains the real gate.

## 1. Git's model: snapshots, hashes and the commit DAG

```text
A---B---C          main
         \
          D---E    feature/split-bills   (HEAD -> feature/split-bills)
```

Each letter is a commit, and every commit records its parent, so history is a directed acyclic graph. `main` and `feature/split-bills` are just pointers (a file under `.git/refs/heads/` containing a hash). HEAD says which branch you are on; committing creates a new commit whose parent is the old tip and moves that branch pointer forward.

| Object | Contains |
|---|---|
| blob | a file's contents (no name) |
| tree | a directory listing: names, modes and the hashes of blobs and subtrees |
| commit | a tree hash, parent hash(es), author, committer and message |
| annotated tag | a target object, the tag name, tagger and message |

You can look inside: `git cat-file -p HEAD` prints the commit, `git cat-file -p HEAD^{tree}` its tree. Because objects are named by the hash of their content, identical files are stored once, corruption is detectable, and changing anything in a commit (its message, parent or content) produces a different commit. That is why amending or rebasing "rewrites history": the old commits still exist, but the branch now points at new ones, and anyone who built on the old ones has diverged. Internally Git packs objects with delta compression, but conceptually every commit is a full snapshot, which is why switching branches is fast and diffs are computed on demand.

## 2. Working tree, index and repository

```text
working tree  --git add-->  index (staging area)  --git commit-->  repository (.git)
      ^                                                                 |
      +------------ git restore / git switch (check files out) --------+
```

A file can be untracked, modified, staged or committed. The index lets you build commits deliberately: commit half of your changes with `git add -p`, keep debugging code out, and split unrelated work into separate commits.

## 3. Everyday commands

```bash
git status -sb                               # short status with branch and ahead/behind
git add -p                                   # stage hunks interactively: y, n, s (split), e (edit)
git commit -m "Add rent reminder job"        # commit what is staged
git commit                                   # editor for a subject plus body
git log --oneline --graph --decorate --all   # the DAG at a glance
git log -p -- app/models/bill.rb             # history of one file, with patches
git log -S "late_fee" --oneline              # commits that added or removed this string
git diff                                     # working tree vs index (unstaged)
git diff --staged                            # index vs HEAD (what you are about to commit)
git diff main...feature/split-bills          # what the branch changed since it forked from main
git show a1b2c3d                             # one commit: message and patch
git show HEAD~2:config/routes.rb             # a file as it was two commits ago
```

`HEAD~2` follows first parents two steps back; `HEAD^2` means the second parent of a merge commit.

## 4. Branches, switching and detached HEAD

```bash
git switch -c feature/split-bills     # create and switch (older: git checkout -b)
git switch main
git switch -                          # back to the previous branch
git branch -vv                        # local branches with upstreams and ahead/behind counts
git branch -d feature/split-bills     # delete if merged; -D forces
git branch -m old-name new-name       # rename
git switch --detach a1b2c3d           # inspect an old commit (detached HEAD)
```

In **detached HEAD**, HEAD points at a commit instead of a branch. You can look around and even commit, but those commits belong to no branch and will eventually be garbage-collected unless you create one (`git switch -c rescue-work`); the reflog can still find them for a while. Switching branches carries uncommitted changes along when they do not conflict; otherwise commit, stash, or open a second checkout with `git worktree add ../hotfix main` to fix a production bug without touching your current work.

## 5. Merging: fast-forward vs merge commits

```text
fast-forward (main has not moved):     A---B  main   +  C---D on feature   ->   A---B---C---D  main, feature

three-way merge (both sides moved):

A---B---E   main                      A---B---E-----M   main
     \                     ->              \       /
      C---D   feature                       C-----D     feature
```

- If `main` has not moved since you branched, `git merge feature` just moves the `main` pointer forward: no new commit.
- Otherwise Git finds the **merge base** (B), combines both sides' changes and records a **merge commit** M with two parents.
- `--no-ff` forces a merge commit even when a fast-forward is possible, keeping a visible "this feature landed here" bubble; `--ff-only` refuses anything but a fast-forward (a safe default for `git pull`).
- `git merge --squash feature` stages the branch's combined changes without committing, so you create one ordinary commit with no merge parent.

## 6. Resolving conflicts step by step

```text
<<<<<<< HEAD
  reminder_days_before = 3
=======
  reminder_days_before = house.reminder_days || 5
>>>>>>> feature/configurable-reminders
```

1. Stay calm: nothing is lost. `git status` lists the "both modified" files, and `git merge --abort` (or `git rebase --abort`) returns you to where you started.
2. Understand both sides before editing: what changed on each branch and why. `git log --merge -p -- <file>` shows the commits involved.
3. Edit the file to the correct combined result (often neither side verbatim) and delete the markers.
4. Run the tests or the app: a textually clean merge can still be logically wrong.
5. `git add <file>` for each resolved file.
6. Finish with `git commit` for a merge, or `git rebase --continue` for a rebase (which may stop again at the next commit).

Tips:
- `git checkout --ours <file>` or `--theirs <file>` takes one side wholesale. During a rebase the meanings flip: "ours" is the branch you are replaying onto, "theirs" is your commit.
- `git config --global merge.conflictstyle zdiff3` also shows the common ancestor, which makes each side's intent much clearer (Git 2.35+).
- For lockfiles and generated files (`Gemfile.lock`, `package-lock.json`, `db/schema.rb`), take one side and regenerate (`bundle install`, `npm install`, `bin/rails db:migrate`) instead of hand-merging.
- `git config --global rerere.enabled true` makes Git remember resolutions and replay them when the same conflict reappears during long rebases.
- Editor merge tools (VS Code's merge editor, RubyMine) show ours, theirs and the result side by side.

## 7. Rebase and the golden rule

```text
before:                          after `git rebase main` while on feature:

A---B---E   main                 A---B---E   main
     \                                    \
      C---D   feature                      C'---D'   feature
```

Rebase replays C and D on top of E as new commits C' and D': the same changes with new parents, so new hashes. The result is linear history with no merge commit, and conflicts are resolved one replayed commit at a time. A branch you already pushed must then be force-pushed with `--force-with-lease`.

**The golden rule:** do not rebase commits that exist outside your machine and that others may have built on: `main`, `develop`, release branches, a teammate's branch. Their copies still contain the old commits, so the histories diverge and the next merge duplicates work. Rebasing your own feature branch before review or merge is normal.

```bash
git rebase -i main      # or HEAD~4: edit the last commits before opening a PR
# the editor lists one commit per line; change the command word, reorder or delete lines:
# pick   a1b2c3d Add bill split model
# fixup  b2c3d4e Fix typo in split model
# reword c3d4e5f Add split UI
# drop   d4e5f6a WIP debug logging

git commit --fixup a1b2c3d          # later: record a fix meant for an earlier commit
git rebase -i --autosquash main     # fixup commits jump to the right place automatically
```

Commands: `pick` keeps a commit, `reword` edits its message, `edit` stops to amend it, `squash` melds it into the previous one and combines messages, `fixup` melds it and discards its message, `drop` deletes it. **Going deeper:** `git rebase --update-refs` (Git 2.38+) also moves the other branches in a stack of dependent branches.

## 8. Merge vs rebase vs squash

| | Merge commit | Rebase | Squash |
|---|---|---|---|
| History | true history, with merge bubbles | linear, rewritten | one commit per branch |
| Rewrites commits | no | yes (new hashes) | yes (branch commits replaced by one) |
| Safe on shared branches | yes | no (golden rule) | yes, as the way a PR lands |
| Conflict resolution | once | once per replayed commit | once |
| Revert and bisect | revert needs `-m 1`; bisect can land on merges | precise | the whole feature reverts at once; bisect cannot look inside it |
| Best for | long-lived branches, preserving context | updating and tidying your own branch | PRs full of WIP commits |

A common team setup: rebase locally to stay current and tidy, then squash-merge or merge on GitHub ([GitHub and workflows](02-github-and-workflows.md)).

## 9. Undoing things: reset vs revert vs restore

```bash
git restore app/models/bill.rb                # discard unstaged changes to a file
git restore --staged app/models/bill.rb       # unstage it, keeping the changes
git restore --source=HEAD~3 config/app.yml    # bring back a file as it was three commits ago
git reset --soft HEAD~1      # undo the last commit, keep its changes staged
git reset HEAD~1             # (--mixed) undo it, keep the changes unstaged
git reset --hard HEAD~1      # undo it and discard the changes (the commit is still in the reflog)
git reset --hard origin/main # make the current branch match the remote exactly
git revert a1b2c3d           # new commit that undoes a1b2c3d: safe on shared branches
git revert -m 1 f00ba47      # revert a merge commit, keeping parent 1 (the mainline)
```

| Command | Moves the branch | Working tree | Index | Rewrites history | Use when |
|---|---|---|---|---|---|
| `restore <file>` | no | file reset (or unstaged with `--staged`) | unchanged | no | discarding or unstaging file changes |
| `reset --soft <commit>` | yes | kept | kept, staged | yes | redoing recent commits as one, fixing a message |
| `reset --mixed <commit>` | yes | kept | reset | yes | uncommitting and restaging selectively |
| `reset --hard <commit>` | yes | overwritten | reset | yes | throwing away local work (careful) |
| `revert <commit>` | adds a commit | updated | updated | no | undoing something already pushed or shared |

`reset --hard` permanently destroys uncommitted changes, because they were never in a commit the reflog could point to. Committed work survives in the reflog. Reverting a merge has a catch: if you later want to merge that branch again, you must first revert the revert.

## 10. Stash

```bash
git stash push -m "wip: split bills form"   # shelve tracked changes (plain git stash also works)
git stash -u                                # include untracked files
git stash list                              # stash@{0}: On feature/x: wip: split bills form
git stash show -p stash@{0}                 # inspect a stash
git stash pop                               # apply the newest and drop it (kept if it conflicts)
git stash apply stash@{1}                   # apply, but keep it in the list
git stash drop stash@{1}
git stash branch fix/from-stash stash@{0}   # new branch at the stash's base, with the stash applied
```

Stash fits short interruptions: an urgent production bug while your work is half done, or a pull blocked by local changes. Stashes are easy to forget, so for longer detours prefer a WIP commit on your branch (amend or squash it later) or a separate worktree.

## 11. Cherry-pick

```bash
git switch release/2.4
git cherry-pick -x a1b2c3d          # copy the fix; -x appends "(cherry picked from commit ...)"
git cherry-pick a1b2c3d^..f6e5d4c   # a range including a1b2c3d (without ^ it starts after it)
git cherry-pick --continue          # after resolving conflicts (or --abort)
```

Use it to backport a hotfix to a release branch or rescue one commit from an abandoned branch. It creates a new commit with the same changes and a different hash, so if both branches later merge you can get duplicate-looking commits or conflicts; when a whole branch should land, merge it instead.

## 12. Reflog: recovering lost work

```bash
git reflog                            # every position HEAD has had, newest first
# 4f2a9c1 HEAD@{0}: reset: moving to HEAD~3
# 9d8e7f6 HEAD@{1}: commit: Add rent reminder job
git branch rescue 9d8e7f6             # put a branch on the "lost" commit
git reset --hard HEAD@{1}             # or move the current branch back to where it was
git reset --hard ORIG_HEAD            # undo the last reset, merge or rebase (Git saves the old tip there)
git reflog show feature/split-bills   # the history of one branch pointer
```

- The reflog records every local movement of HEAD and branch tips: commits, resets, rebases, merges, amends, checkouts.
- Entries expire (by default after 90 days for reachable commits and 30 for unreachable ones), after which garbage collection may delete unreferenced commits.
- A deleted branch is recoverable the same way: find its last commit in the reflog, or in the "Deleted branch ... (was 1a2b3c4)" message, and recreate it.
- The reflog is local: it cannot recover commits that only existed on someone else's machine, nor uncommitted changes wiped by `reset --hard` (only staged content might be found as dangling blobs with `git fsck --lost-found`).

## 13. Tags

```bash
git tag -a v1.4.0 -m "Release 1.4.0: split bills"   # annotated: tagger, date and message (preferred)
git tag v1.4.0-rc1                                   # lightweight: just a pointer
git push origin v1.4.0                               # tags are not pushed by default
git push --follow-tags                               # push commits plus the annotated tags on them
git tag -l "v1.*"
git switch --detach v1.3.2                           # inspect a release
```

Tags usually follow semantic versioning (MAJOR.MINOR.PATCH), and GitHub Releases are built on them. For mobile apps, tagging each store release maps builds to commits ([EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)).

## 14. Remotes: fetch, pull, push, upstream and force-with-lease

```bash
git remote -v                            # where origin points
git fetch origin                         # download new commits; only origin/* refs move
git fetch --prune                        # also drop remote-tracking branches deleted on the server
git pull                                 # fetch, then merge the upstream into the current branch
git pull --rebase                        # fetch, then replay local commits on top
git config --global pull.rebase true     # make rebase the default (or pull.ff only to refuse merges)
git push -u origin feature/split-bills   # first push: create the remote branch and set the upstream
git push --force-with-lease              # after rebasing or amending your own pushed branch
git push origin --delete feature/split-bills
```

- **Remote-tracking branches** (`origin/main`) are your read-only local snapshot of the remote as of the last fetch. `fetch` is always safe; `pull` changes your branch.
- **`--force-with-lease`** overwrites the remote branch only if it still points where your `origin/<branch>` says, so a teammate's freshly pushed commits are not silently destroyed. Caveat: a background fetch (many editors do this) updates `origin/<branch>`, so the lease can pass without you having seen the new commits; `--force-if-includes` (Git 2.30+) adds a check that the remote tip is part of your local history.
- Never force-push shared branches; GitHub branch protection should block it on `main`.

## 15. .gitignore

```text
/node_modules/
/log/*
!/log/.keep
/tmp/
/storage/
/coverage/
.env
.env.*
!.env.example
/config/master.key
*.log
```

- A leading `/` anchors a pattern to the repository root, a trailing `/` matches directories only, `!` re-includes, `**` spans directories, and later rules override earlier ones.
- Ignore rules only affect **untracked** files. Already committed it? `git rm --cached <path>` (the file stays on disk), commit, and rotate it if it was a secret.
- Put personal editor and OS files in a global ignore (`git config --global core.excludesFile ~/.gitignore_global`) instead of every project. `git check-ignore -v <path>` shows which rule matched. Git does not track empty directories, hence `.keep` files.

## 16. bisect and blame

```bash
git bisect start
git bisect bad                      # the current commit has the bug
git bisect good v1.3.0              # this release did not
# Git checks out a commit halfway between: test it, then mark it good or bad; repeat
git bisect reset                    # done: return to where you started

# automated: exit code 0 = good, 125 = skip this commit, other codes up to 127 = bad
git bisect run bundle exec rspec spec/models/bill_spec.rb
```

Bisect is a binary search over history: about 10 steps for 1,000 commits. It is the fastest way to answer "this worked last month; what broke it?", especially with a script that reproduces the bug.

```bash
git blame -w -L 40,60 app/models/bill.rb     # who last changed lines 40 to 60, ignoring whitespace
git log -L 40,60:app/models/bill.rb          # the full history of those lines
git config blame.ignoreRevsFile .git-blame-ignore-revs   # skip mass-formatting commits
```

Use blame to find the commit and its pull request, which explain *why* the code looks the way it does, not to blame people. GitHub's blame view honors `.git-blame-ignore-revs` automatically.

## 17. Commit messages

```text
Add reminder job for unpaid bills

Users only learned about overdue bills from the monthly email.
Enqueue a reminder three days before the due date and skip bills
that are already paid.

Refs: #412
```

- **Subject:** imperative mood (as in "this commit will *add* ..."), about 50 characters, no trailing period. Then a blank line and a body wrapped near 72 characters that explains why, plus trade-offs and references.
- **Atomic commits:** one logical change per commit makes review, revert, bisect and cherry-pick easier. Keep formatting changes out of behavior changes.
- **Conventional Commits:** `type(scope): description`, with types such as `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `ci` and `chore`, and `!` or a `BREAKING CHANGE:` footer for breaking changes. Tools such as release-please and semantic-release generate changelogs and version bumps from them, and commitlint enforces the format in a `commit-msg` hook. Examples: `feat(bills): add percentage-based split`, `fix(api): return 404 for bills outside the user's houses`.
- In client repositories, follow the client's convention.

## 18. Recipes for common mistakes

**Committed to `main` instead of a feature branch (not pushed yet):**

```bash
git branch feature/split-bills      # a new branch pointing at your commits
git reset --hard origin/main        # move main back to the remote state
git switch feature/split-bills      # carry on there
```

- **Committed to the wrong existing branch:** `git switch right-branch && git cherry-pick <sha>`, then remove it from the wrong branch with `git reset --hard HEAD~1` if unpushed or `git revert <sha>` if pushed.
- **Fix the last commit:** `git add forgotten_file.rb && git commit --amend --no-edit`, or `git commit --amend -m "Better message"`. Amending rewrites the commit, so a pushed branch of your own needs `git push --force-with-lease`.
- **Undo a pushed commit on a shared branch:** `git revert <sha>` and push.
- **Squash your branch into one commit:** `git reset --soft $(git merge-base HEAD main) && git commit`, or an interactive rebase.
- **Update your branch with `main`:** `git fetch && git rebase origin/main` (then `--force-with-lease`), or `git merge origin/main` if the team prefers merges.
- **Committed `node_modules` or a huge file (unpushed):** `git rm -r --cached node_modules`, add it to `.gitignore`, and `git commit --amend` (or fix the right commit with an interactive rebase).

**Committed a secret:**
1. **Revoke or rotate it immediately** at the provider. It was compromised the moment it reached a remote: automated scanners find pushed keys within minutes, and clones, forks and CI caches already have copies that no history rewrite can recall.
2. Check the provider's logs for misuse.
3. If the history must be cleaned, use `git filter-repo` (a separate tool that the Git documentation recommends over the old `filter-branch`), for example `git filter-repo --invert-paths --path config/secrets.yml` or `git filter-repo --replace-text replacements.txt`; BFG Repo-Cleaner is an alternative. Force-push every branch and tag, ask collaborators to re-clone, and ask the host to purge cached references if needed.
4. Prevent repeats: `.gitignore` plus a committed `.env.example`, a secret-scanning pre-commit hook such as gitleaks, and GitHub push protection.

## 19. Hooks

Hooks are scripts in `.git/hooks/` that Git runs at certain points: `pre-commit` (lint, format, scan staged files for secrets), `commit-msg` (enforce a message format), `pre-push` (run tests). That folder is not versioned, so teams share hooks with a tool:
- **husky** in Node projects: `npx husky init` creates `.husky/pre-commit`; pair it with lint-staged to run tools only on staged files.
- **lefthook:** a single binary configured in YAML, language-agnostic and common in Rails projects.
- **pre-commit:** a Python-based framework with a shared catalog of hooks.

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    rubocop:
      glob: "*.rb"
      run: bundle exec rubocop --force-exclusion {staged_files}
    eslint:
      glob: "*.{ts,tsx}"
      run: npx eslint {staged_files}
```

Hooks only run where they are installed and can be skipped with `--no-verify`, so CI must run the same checks as the real gate. Keep hooks fast (seconds), or people will bypass them.

## Common pitfalls
- `git add .` sweeping in secrets, debug code or generated files; always review `git diff --staged` first.
- Rebasing or force-pushing shared branches, or using `--force` instead of `--force-with-lease`.
- `reset --hard` with uncommitted work you still needed.
- Long-lived branches that drift for weeks and end in huge, painful conflicts.
- Resolving conflicts by dropping the other side's changes without understanding them, and not re-running tests.
- Hand-merging lockfiles and `schema.rb` instead of regenerating them.
- Commit messages like "fix", "changes" or "WIP" left in shared history; refactors mixed into feature commits.
- Committing directly to `main` in client repositories.
- Expecting `.gitignore` to untrack files that are already committed.
- Scrubbing a secret from history without rotating it.
- Forgetting to push tags.

## How to talk about this in an interview
- Git and GitHub are in your skills, and you use them daily at Devntech in client repositories: feature branches, pull requests, rebasing onto `main`, and resolving conflicts in legacy Rails monoliths where many people touch the same hot files (`schema.rb`, `routes.rb`, shared models).
- Show that you understand the model (snapshots, commits as a graph, branches as pointers), because every "how do I undo X" answer follows from it.
- Demonstrate safety habits: `--force-with-lease`, `revert` on shared branches, the reflog when something goes wrong, small atomic commits with messages that explain why.

> **Fill in:** your team's workflow at Devntech: branch naming, merge vs squash vs rebase on PRs, commit message convention, and who reviews.

> **Fill in:** a time you recovered lost work (reflog, a bad rebase) or untangled a difficult conflict, and what you learned.

## Interview questions

**Q1. How does Git store data? What are a commit, a branch and HEAD?**
Git stores content-addressed objects: blobs for file contents, trees for directories, commits that point to a tree plus parent commits and metadata. Each commit is a full snapshot, and commits form a graph. A branch is a movable pointer to a commit; HEAD points to the current branch (or a commit, in detached HEAD), and committing advances that branch.
*Follow-up:* Why does amending change the hash? The hash covers the commit's content, including its message and parent, so any change creates a new commit.

**Q2. What are the working tree, the index and the repository?**
The working tree is the files on disk, the index (staging area) is the snapshot that the next commit will record, and the repository (`.git`) holds all committed objects and refs. `git add` moves changes into the index, `git commit` records the index as a commit, and `git restore` brings content back out.
*Follow-up:* Why have a staging area? To build focused commits from a messy working tree, for example with `git add -p`.

**Q3. Fast-forward vs merge commit?**
If the target branch has not moved since you branched, merging simply moves its pointer forward (fast-forward). If both sides have new commits, Git does a three-way merge from the merge base and records a merge commit with two parents. `--no-ff` forces a merge commit; `--ff-only` refuses anything else.
*Follow-up:* Why would a team want `--no-ff`? To keep a visible record of when each feature branch landed.

**Q4. git merge vs git rebase: when do you use each?**
Merge combines histories without rewriting anything, which suits shared and long-lived branches. Rebase replays your commits on a new base for a linear history, which suits updating and cleaning up your own branch before review. Rebase rewrites commits, so never use it on history others share.
*Follow-up:* What does a rebase of a pushed branch require? A force-push, ideally `--force-with-lease`.

**Q5. What is the golden rule of rebasing, and why?**
Do not rebase commits that exist outside your repository and that others may have built on. Rebase replaces commits with new ones, so collaborators still holding the old commits end up with diverged histories, duplicate commits and confusing conflicts.
*Follow-up:* Is rebasing a feature branch that only you use fine? Yes, as long as nobody else bases work on it.

**Q6. Merge commit vs squash vs rebase merge for pull requests?**
A merge commit preserves every branch commit plus a merge bubble. Squash turns the whole PR into one commit on `main`: clean history and an easy revert, but fine-grained commits are lost. Rebase-and-merge replays each commit linearly: linear history that keeps individual commits, which then must each be meaningful.
*Follow-up:* Which for messy WIP commits? Squash, unless the author cleans them up first with an interactive rebase.

**Q7. git reset (soft, mixed, hard) vs git revert vs git restore?**
`reset` moves the branch pointer: `--soft` keeps changes staged, `--mixed` unstaged, `--hard` discards them; it rewrites history, so use it locally. `revert` creates a new commit that undoes an earlier one, which is safe on shared branches. `restore` changes files in the working tree or index without touching history.
*Follow-up:* How do you revert a merge commit? `git revert -m 1 <merge>`, choosing the mainline parent.

**Q8. How do you undo the last commit, pushed and not pushed?**
Not pushed: `git reset --soft HEAD~1` to keep the changes staged (or `--hard` to drop them), or `git commit --amend` to fix it. Pushed to a shared branch: `git revert HEAD` and push, never rewrite. Pushed only to your own branch: amend or reset, then `--force-with-lease`.
*Follow-up:* What if you just want to change the message? `git commit --amend -m "New message"`.

**Q9. A reset or rebase went wrong and commits are gone. How do you recover them?**
Run `git reflog`, find the entry from before the mistake, and either create a branch there (`git branch rescue <sha>`) or move back with `git reset --hard HEAD@{n}`. Right after a reset, merge or rebase, `git reset --hard ORIG_HEAD` also works. Unreachable commits stay recoverable until reflog expiry and garbage collection.
*Follow-up:* What can the reflog not recover? Uncommitted changes destroyed by `reset --hard`, and commits that never existed in your repository.

**Q10. git fetch vs git pull?**
`fetch` downloads new objects and updates remote-tracking branches like `origin/main` without touching your branches, so it is always safe. `pull` is fetch followed by merging (or rebasing, with `--rebase` or `pull.rebase true`) the upstream into your current branch.
*Follow-up:* Why fetch first? To inspect incoming changes (`git log HEAD..origin/main`) before integrating them.

**Q11. Why use --force-with-lease instead of --force?**
`--force` overwrites the remote branch unconditionally, destroying commits others pushed since your last fetch. `--force-with-lease` only overwrites if the remote still points where your remote-tracking branch expects, so it fails safely when someone else pushed.
*Follow-up:* When can the lease still bite you? After a background fetch updated the tracking ref without you looking; `--force-if-includes` closes that gap.

**Q12. Walk me through resolving a merge conflict.**
Check `git status` for conflicted files, read both sides and the history behind them, edit each file to the correct combined result, remove the markers, run the tests, `git add` the files and finish with `git commit` or `git rebase --continue`. If it goes badly, `--abort` returns to the starting point. Regenerate lockfiles and schema files instead of hand-merging them.
*Follow-up:* What do "ours" and "theirs" mean during a rebase? They are reversed from a merge: "ours" is the branch you are replaying onto, "theirs" is your commit.

**Q13. You committed to main instead of a feature branch. How do you fix it?**
If unpushed: create a branch at the current commit (`git branch feature/x`), reset `main` to the remote (`git reset --hard origin/main`) and switch to the new branch. If already pushed to a shared `main`, revert the commits there and re-apply them on a branch (cherry-pick), since rewriting shared history is off-limits.
*Follow-up:* How do you prevent it? Branch protection on `main` and creating the branch before you start coding.

**Q14. You committed an API key. What do you do?**
Rotate or revoke it immediately at the provider, because it is compromised as soon as it reached a remote and copies already exist. Check for misuse, then remove it from history with `git filter-repo` or BFG if needed, force-push, and have collaborators re-clone. Prevent repeats with `.gitignore`, secret-scanning hooks and push protection.
*Follow-up:* Why is rewriting history not enough? Clones, forks, CI caches and automated scanners already have the old commits.

**Q15. How do you find which commit introduced a bug?**
`git bisect`: mark a known bad commit and a known good one, and Git checks out the midpoint repeatedly while you mark each as good or bad, finding the culprit in about log2(n) steps. `git bisect run <script>` automates it with a test whose exit code reports good or bad.
*Follow-up:* What if a commit cannot be tested? `git bisect skip`, or exit code 125 from the script.

**Q16. What is git stash, and when do you use it?**
It saves uncommitted changes (and with `-u`, untracked files) on a stack and cleans the working tree, so you can switch context, then reapply with `pop` or `apply`. It suits short interruptions; for longer ones a WIP commit or a separate worktree is harder to lose.
*Follow-up:* `pop` vs `apply`? `pop` applies and removes the stash (unless there is a conflict); `apply` keeps it.

**Q17. What is cherry-pick, and when would you use it?**
It applies the changes of an existing commit onto the current branch as a new commit. Typical uses are backporting a hotfix to a release branch or rescuing a single commit from an abandoned branch; use `-x` to record where it came from.
*Follow-up:* Downside? The copy has a different hash, so later merges of both branches can duplicate or conflict.

**Q18. What makes a good commit message?**
A short imperative subject (about 50 characters) that says what the commit does, a blank line, then a body explaining why the change was needed and any trade-offs, with issue references. Each commit should be one logical change. Teams often add Conventional Commits prefixes for automated changelogs.
*Follow-up:* Why imperative mood? It reads as an instruction that completes "this commit will...", matching Git's own messages like "Merge branch".

**Q19. What are Git hooks, and how do you share them with a team?**
Scripts Git runs at events such as `pre-commit`, `commit-msg` and `pre-push`, used for linting, formatting, message checks and secret scanning. `.git/hooks` is not versioned, so teams use husky with lint-staged, lefthook or the pre-commit framework, and keep the same checks in CI because hooks can be skipped.
*Follow-up:* Why keep hooks fast? Slow hooks get bypassed with `--no-verify`.

**Q20. What is a detached HEAD?**
HEAD points directly at a commit instead of a branch, for example after `git switch --detach <sha>` or checking out a tag. You can explore and commit, but new commits belong to no branch and will be garbage-collected unless you create one with `git switch -c <name>`.
*Follow-up:* Lost commits made in detached HEAD? The reflog still has them until it expires.

## Further reading
- [Pro Git book](https://git-scm.com/book/en/v2)
- [Git reference documentation](https://git-scm.com/docs)
- [Conventional Commits specification](https://www.conventionalcommits.org/)
