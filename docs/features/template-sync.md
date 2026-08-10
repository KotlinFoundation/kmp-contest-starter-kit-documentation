---
sidebar_position: 22
---

# Template Sync

Apps created from the starter kit run `refactor_package.sh`, which renames the package in
practically every file. After that, a plain `git merge` against the template drowns in rename
noise — hundreds of conflicts for even a small template update. The template-sync tooling restores
a real merge base, so a sync only surfaces genuine feature changes plus the places your app edited
the same code.

Everything is driven by one script:

```bash
./MobileApp/scripts/sync_template.sh
```

The agent-facing manual lives in `skills/sync-template/SKILL.md`; release notes for each template
change live in `CHANGELOG.md` at the repo root.

## How it works

The script renames the **incoming template tree** instead of fighting your app's rename (the
"vendor branch" pattern):

1. A local `template-base` branch holds *renders* — the template tree at a given commit with
   `refactor_package.sh --app-id <your app>` applied. Renders are chained commits, so git always
   has the previous render as the merge base.
2. Each sync runs on a fresh `template-sync/<sha>` work branch cut from your current branch —
   **your branch is never touched**. The script adds a new render on `template-base`, then merges
   it into the work branch. Both sides speak your package name, so rename noise cancels out.
3. You resolve any conflicts and validate on the work branch, then merge it back:

   ```bash
   git checkout <your-branch> && git merge template-sync/<sha> && git branch -d template-sync/<sha>
   ```

4. `.template-version` at the repo root records the template commit last synced; a re-run with
   nothing new is a no-op.

## Usage

One-time setup:

```bash
git remote add template https://github.com/KotlinFoundation/kmp-contest-starter-kit.git
```

Then, whenever you want template updates (clean working tree required):

```bash
./MobileApp/scripts/sync_template.sh
```

### First run and "Use this template" repos

Repos created with GitHub's **Use this template** (or from a squashed clone) share no git history
with the template. The first run then asks for `--bootstrap <sha>` — the template commit your app
was created from. Find it by your repo's creation date:

```bash
git log template/main --until="<when you created the app>" -1 --format='%H %s'
```

An approximate **older** commit is fine — a slightly-off base only means a few extra conflicts in
the first merge. The script renders that commit and grafts ancestry with an `ours` merge (your
tree is unchanged); afterwards every sync behaves like a normal fork.

## Resolving conflicts

Conflicts appear only where your app edited the same code the template changed. Rules of thumb:

- **Your product code wins, template infrastructure wins.** Your screens, models, and copy stay;
  template changes to scripts, gradle, CI, and skills come in.
- **`strings.xml`** — union-merge both sides; your rewording of a key beats the template's.
- **Shared wiring files** (`Routes.kt`, `AppNavigation.kt`, `Di.kt`, `AppDatabase.kt`) — keep your
  entries and add the template's new ones.
- **Demo screens you deleted or replaced** — keep the deletion; don't resurrect template demo code.

Validate before merging back — run the quality gates from `MobileApp/` (spotless, tests,
`assembleDebug`). Template changes can compile against demo code your app no longer has.

## The changelog

`CHANGELOG.md` at the repo root gives every template change a short, derived-app-facing entry:
scope tags (`[app]` / `[skills]` / `[docs]`) plus a **Manual:** line for anything you must do by
hand after the merge. Read the entries since your `.template-version` before resolving conflicts —
they state the intent behind each change.

## Heavily diverged apps

If your app rewrote whole subsystems (navigation, DI, storage), merging degrades into porting
work — still safe (abort with `git branch -D template-sync/<sha>` and nothing happened), but
noisy. Past that point, skip the merge and port instead: the render diff

```bash
git diff template-base~1 template-base
```

is exactly what the template changed, already expressed in your package names — apply it to your
architecture by hand, guided by the changelog entry.
