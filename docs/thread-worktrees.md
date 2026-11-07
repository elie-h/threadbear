# A worktree per thread

Mike, 2026-10-27: several threads work on Threadbear at once, all in the
same checkout. One thread's `git add -A` has swept another's half-done work
into a commit, merges and test runs collide, and every agent has to
remember to make its own worktree. Instead, a repo can ask for each new
thread to get a worktree of its own, and a thread lands its branch on main
when its checks pass.

## Opting in

Only repos that ask. A patch (docs/protocol.md, "Patches") can say:

```ts
export default {
  worktree: {
    // Run in the new worktree before the first turn. Optional. One
    // script in the repo and plain words, as it runs by itself
    // (docs/protocol.md, "Trusting a folder's patches").
    setup: "./scripts/setup.sh",
    // Run by `tb land` before main moves. Optional; a failure stops it.
    check: "./check.sh --quick",
    // Run in the worktree's root when the thread is archived or deleted,
    // before cleanup. Optional. Held to the same rules as setup.
    teardown: "./scripts/teardown.sh",
    // "pr": tb land pushes the branch and opens a pull request instead
    // of moving main (see "Landing as a PR"). Optional; "main" is the
    // default.
    land: "pr",
  },
  // actions, panels, hooks as before
};
```

The first patch in the thread's folder chain with a `worktree` wins (home's
`~/.threadbear/patches/` never counts: worktrees are a repo's choice). The
repo is the git toplevel of the folder the patch is in, and its main
checkout is the one with `main` checked out.

On a Mac, `setup`, `check` and turn hooks run at background priority
(`taskpolicy -b`, `runner/src/background.ts`), as do the runner's and the
app's own update builds, so a land's test suite stays off the cores you're
using. Action buttons you press run as normal.

## A new thread

`threads.create` (and `create_thread`, and schedules that start a thread)
for a folder inside an opted-in repo:

1. Brings main up to origin's first, when the main checkout has an
   `origin`: `git fetch origin main`, then a fast-forward of main if it's
   only behind (Elie, 2026-11-06: PRs merged on GitHub never reached the
   local main, so threads started 111 commits behind). If main has forked
   from origin's, or the fetch fails (20 s at most), main is left as it
   is and the thread's "Working in a worktree" note says why.
   Then makes `git worktree add -b thread/<id8> <parent>/<repo>-threads/<id8>
   main` from the main checkout, where `<id8>` is the first 8 characters of
   the thread id. A leftover folder or branch of that name is an error
   shown in the thread, and the thread starts in the main checkout as
   before.
2. The thread's `cwd` is the same subfolder of the worktree as the folder
   asked for (asking for `~/dev/me/threadbear/runner` gives
   `.../threadbear-threads/<id8>/runner`).
3. `setup` runs in the worktree's root before the first turn starts. Its
   output goes in the transcript as an `action.run` event (`trigger:
   "worktree.setup"`). A failure is shown and the turn starts anyway.
   It's the worktree's own copy of the patch that's read, and only when
   that copy (with the scripts it names, as they are in the worktree) is
   trusted; otherwise a note says the setup was skipped (docs/protocol.md,
   "Trusting a folder's patches").
4. The record keeps `worktree: { repo, mainCheckout, path, branch, base }`
   (`base` is main's commit when it was made; `land: "pr"` too, from the
   patch); the summary carries
   `worktree?: { branch, mainCheckout }` so the app can show it.
5. `threads.create` takes `worktree: false` to start in the folder as
   before.

The agent is told, in the appended system prompt: this thread has its own
worktree of `<repo>` at `<path>` on branch `thread/<id8>`, made from main;
commit there; other threads use other worktrees; when the work is done and
checked, `tb land` puts it on main (or, with `land: "pr"`, pushes it and
opens or updates its PR).

## Landing

`tb land`, run in the thread's worktree (an agent's Bash, or Mike's Land
chip, which is a command action every opted-in thread gets):

It first takes an OS lock in the repo's common git directory
(`threadbear-land.lock`), held through the check and fast-forward. Other
landings wait before rebasing, so each checks against the main left by the
previous landing. The lock releases when the command exits, even on a
crash; the empty lock file stays. Separate repos do not block one another.

1. Refuses with uncommitted changes in the worktree (says which files).
2. `git rebase main` in the worktree. A conflict aborts the rebase and
   says which files; the agent sorts it out and runs `tb land` again. A
   branch with a merge commit on it (Mike's release brought in,
   docs/bringing-in-updates.md) is never rebased, since that would flatten
   the merge and rewrite Mike's commits: if main has moved, main is merged
   into it instead, with the same undo-and-name-the-files on a conflict.
3. Reads the patch's `check` after rebasing, and runs it in the worktree,
   if there is one. A patch load error or failing check stops here, with
   the last lines of its output. Changing the branch or leaving changed
   or untracked files during the check also stops it.
4. `git -C <mainCheckout> merge --ff-only <checked commit>`. If main's working
   tree has uncommitted changes in files the merge touches, git refuses;
   `tb land` says so and which files, and leaves everything as it was.
5. Prints what landed (`git log --oneline main@{1}..main`).

It never pushes (unless the patch says `land: "pr"`, below). The runner and the app follow the local main by
themselves.

The Land chip has an hour for waiting and checking; other command actions
keep their usual ten-minute limit. In an agent tool with its own shorter
command limit, run `tb land` in the background and collect its result.

After landing, the branch stays and the thread can carry on: the next
`tb land` lands only what's new. `tb land --dry-run` does steps 1 to 3 and
says what would land.

### Landing as a PR

Elie, 2026-11-06: rawa's branches reach main through PRs squash-merged on
GitHub. Fast-forwarding the local main to a thread's commits meant it
forked from origin's main after every land (the squash is a new commit),
and new threads started from a stale main. With `land: "pr"`, the local
main only ever follows origin's, and `tb land`:

1. Refuses with uncommitted changes, as above.
2. Fetches origin's main and rebases the branch on `origin/main` (the
   same undo-and-name-the-files on a conflict, the same merge for a
   branch with a merge on it). A local main that's only behind origin's
   is fast-forwarded; one that has forked is left, with a line saying so.
   A failed fetch stops it.
3. Runs `check`, as above.
4. Pushes the branch to origin, with `--force-with-lease` against what
   origin had for it a moment before, as a later land's rebase rewrites
   it.
5. Opens a PR with `gh pr create` (title from the first commit, body from
   its message, or a list of the commits when there are several) unless
   the branch has one open, which the push has just updated. Prints the
   PR's URL.

`--dry-run` does steps 1 to 3 and says what it would push. The Land chip
is "Open PR" for such a thread (threads made before the patch said so
keep "Land"; `tb land` reads the patch either way). Once the PR is
squash-merged, the branch's own commits no longer match main's, so a
thread carrying on afterwards should start again from `origin/main`
rather than land more on the same branch. Cleaning up still counts a
branch as landed only when it's on the local main, which a squash-merged
one never is, so its worktree stays until it's removed by hand.

## Cleaning up

Elie, 2026-11-06: a repo whose setup makes something outside the folder
(a Convex deployment per worktree, say) needs it gone when the thread is
done, and its branches land through squashed PRs, so git never sees them
as merged. So when a thread with a worktree is archived or deleted, the
patch's `teardown` runs first, in the worktree's root, whether or not the
branch landed. It's the worktree's own copy that's read and trusted, as
for `setup`. Its output goes in the transcript as an `action.run` event
(`trigger: "worktree.teardown"`); a deleted thread gets a notice only if
it fails. It runs on each archive, so it should be safe to run twice.

When a thread with a worktree is archived or deleted: if its branch is
merged into main (or has no commits of its own), the worktree is removed
(`git worktree remove`, `--force` only for untracked files) and the branch
deleted. Otherwise both stay, with a `system.note` saying so and how to
land or drop them. Unarchiving a thread whose worktree was removed starts
its next turn in the main checkout, with a note.

An archive's cleanup waits for the thread's turn-end hooks, which run in that
folder: on 7 Nov a typecheck lost its files partway through, "failed", and
woke the thread just archived. A restart waits for those hooks and a handover
keeps the thread while they run, for the same reason. A hook failure told to
the agent brings the thread back in its worktree, so nothing gets removed; one
that isn't told (`tell: "never"`, or past the limit) leaves it archived and the
cleanup goes ahead. If the runner stops or crashes first, the next one does the
cleanup (`cleanupOwed` in the worktree's record). Deleting a thread doesn't
wait: nothing is left to report a hook's result to.

The folder is only removed while it's still a checkout of the thread's own
branch: slot folders are reused, and a record can miss hearing its worktree
went (THREADBEAR-16, 6 Nov: a handover saved it while the cleanup ran; a
handover now keeps that thread until the cleanup is done, for up to ten
minutes, while the rest move across). A folder that's
another checkout's now is left alone and the thread treats it as a spare's.
At start, an archived thread whose branch is gone, and whose folder is gone
or another checkout's, is marked removed. A message sent while the cleanup
runs waits for it, so the turn never starts in a folder being removed.

## Reusing worktrees

Mike, 2026-10-29: removing a sound effect took a thread 18 minutes, 17 of
them `./check.sh --app` in a worktree that had never built. SwiftPM's
`.build` only works in the folder it was built in (copying main's, even
as an APFS clone, still rebuilt everything: 297 s), so every new worktree
paid 5 to 10 minutes for its first app build, more with other threads
compiling at once.

So a patch can list its build caches:

```ts
worktree: { setup: "...", check: "...", keep: ["app/.build", "dock/.build"] },
```

With `keep`:

- New worktrees are `<repo>-threads/slot-<n>`, not `<id8>`, as the folder
  outlives the thread.
- Cleaning up a landed one detaches it, deletes its branch and locks it
  (`git worktree lock --reason "threadbear spare"`): a spare. Past six
  spares (`MAX_SPARES`), it's removed as before.
- A new thread takes a spare if there is one: unlocking is the claim, so
  two threads (or a thread and `borrow-worktree.sh`) never get the same
  one. Then `git reset --hard`, `git clean -ffdx` with the kept folders
  left out (an old thread's `node_modules` goes, its build cache stays),
  and `git checkout -b thread/<id8> main`. Setup runs as for a new one.
- The record keeps `keep`, and `spare: true` once it's set aside. A
  spare's thread, unarchived, moves to the main checkout even though the
  folder is still there: it's another thread's now.

A thread started outside the repo borrows one by hand with
`./borrow-worktree.sh` and gives it back with `--give-back <dir>`.

A hand-merge from a borrowed worktree holds the same landing lock. From
that worktree, on a Mac:

```sh
/usr/bin/lockf -k "$(git rev-parse --git-common-dir)/threadbear-land.lock" /bin/sh /path/to/landing-script
```

On Linux use `flock -x` in place of `/usr/bin/lockf -k`. The script must
rebase on current main, run the required checks on that combined revision,
and fast-forward main, with the lock held for the whole sequence. Stop on
any failure. Don't call `tb land` inside the script: it takes the same lock
and would wait on itself. A plain git merge outside the lock can move main
under a landing whose check is already running. A check from before a
rebase cannot validate the new combination, even when files don't overlap.

## Not in v1

- Repos that aren't on `main`.
- Moving a thread with a worktree to another folder (allowed, the worktree
  just stays where it is until the thread is archived).
- Showing the branch in the app beyond the summary field.
- Projects (docs/projects-design.md): when patches load from project
  folders, `worktree` still only counts from the thread's own folder chain.
