# Git Command Production Cheatsheet 🛠️
### *Porcelain & Plumbing Commands with Strict Safety Classifications*

---

## 🚦 Safety Level Legend
- `🟢 [SAFE]`: Read-only. Repository history ya working files ko touch nahi karta. Safe to execute anytime.
- `🟡 [CAUTION]`: Working directory ya staging area ko modify karta hai. Uncommitted changes alter ho sakte hain.
- `🔴 [DANGEROUS]`: History rewriting ya permanent data deletion. Executing without understanding can cause data loss!

---

## 1. Setup & Configuration ⚙️

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git config --global user.name "John Doe"` | `🟡 [CAUTION]` | Global author name set karta hai (`~/.gitconfig` mein store hota hai). |
| `git config --global user.email "john@work.com"` | `🟡 [CAUTION]` | Global author email set karta hai (commits ke author metadata ke liye). |
| `git config --local user.email "john@personal.com"` | `🟡 [CAUTION]` | Sirf current repository ke liye email override karta hai (`.git/config`). |
| `git config --list --show-origin` | `🟢 [SAFE]` | Saare configurations aur unke source file paths display karta hai. |
| `git config --global init.defaultBranch main` | `🟡 [CAUTION]` | Default initial branch ka naam `main` set karta hai (`master` ke bajaye). |
| `git config --global core.editor "code --wait"` | `🟡 [CAUTION]` | Default commit message editor set karta hai (e.g. VS Code, nano, vim). |

---

## 2. Inspection & Forensic Logging 🔍

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git status` | `🟢 [SAFE]` | Working tree aur staging area ka current status check karta hai. |
| `git status -s` | `🟢 [SAFE]` | Compact format status (`M`, `A`, `??`, `UU`). Fast terminal overview. |
| `git log --oneline --graph --decorate --all` | `🟢 [SAFE]` | Saari branches ka ASCII graph visual commit tree format mein dikhata hai. |
| `git log -p -2` | `🟢 [SAFE]` | Last 2 commits ke actual code diffs (patch) inline print karta hai. |
| `git log --stat` | `🟢 [SAFE]` | Commits ke sath changed files aur line count additions/deletions dikhata hai. |
| `git diff` | `🟢 [SAFE]` | Working directory aur Staging area (Index) ke beech ka un-staged diff dikhata hai. |
| `git diff --staged` (ya `--cached`) | `🟢 [SAFE]` | Staged changes aur last commit (HEAD) ke beech ka diff dikhata hai. |
| `git diff branchA..branchB` | `🟢 [SAFE]` | Do alag branches ke tips ke beech difference compare karta hai. |
| `git blame <file>` | `🟢 [SAFE]` | File ki har ek line kis developer ne kis commit mein likhi thi wo author dikhata hai. |
| `git reflog` | `🟢 [SAFE]` | Local HEAD ke har move ka complete flight-recorder log (even deleted commits!). |

---

## 3. Staging & Committing 💾

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git add <file>` | `🟡 [CAUTION]` | Specific file ko Index (Staging area) mein stage karta hai. |
| `git add .` | `🟡 [CAUTION]` | Current directory ke saare modified aur new files ko stage karta hai. |
| `git add -p` (patch mode) | `🟡 [CAUTION]` | Interactive staging. File ke specific chunks/hunks ko alag-alag select karke stage karein. |
| `git commit -m "feat: message"` | `🟡 [CAUTION]` | Staged snapshot ko repository history mein naye commit object ke roop mein save karta hai. |
| `git commit --amend` | `🟡 [CAUTION]` | Last commit ka message update karta hai ya naye staged changes usme merge karta hai (**Local only!**). |
| `git commit --amend --no-edit` | `🟡 [CAUTION]` | Bina message change kiye last commit mein staged files add karta hai. |

---

## 4. Branching & Navigation 🌿

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git branch` | `🟢 [SAFE]` | Local branches ki list dikhata hai. Current branch `*` se marked hoti hai. |
| `git branch -a` | `🟢 [SAFE]` | Local aur Remote-tracking branches dono list karta hai. |
| `git branch -vv` | `🟢 [SAFE]` | Branches ke upstream tracking aur commit difference (`ahead`, `behind`) dikhata hai. |
| `git branch <name>` | `🟡 [CAUTION]` | Current HEAD par nayi branch pointer banata hai (checkout nahi karta). |
| `git switch <name>` (ya `git checkout <name>`) | `🟡 [CAUTION]` | Doosri branch par switch karta hai aur working tree update karta hai. |
| `git switch -c <name>` (ya `git checkout -b <name>`) | `🟡 [CAUTION]` | Nayi branch create karke turant us par switch karta hai. |
| `git branch -d <name>` | `🟡 [CAUTION]` | Fully merged branch ko safely delete karta hai (Git prevents if unmerged). |
| `git branch -D <name>` | `🔴 [DANGEROUS]` | Force delete branch! Chahe changes merge huye hon ya nahi. |

---

## 5. Merging, Rebasing & Cherry-Picking 🔀

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git merge <branch>` | `🟡 [CAUTION]` | Target branch ko current branch mein integrate karta hai (Fast-Forward ya 3-Way). |
| `git merge --no-ff <branch>` | `🟡 [CAUTION]` | Always creates a true merge commit, even if fast-forward is possible. |
| `git merge --abort` | `🟡 [CAUTION]` | Conflict aane par merge cancel karke repo ko pre-merge state par le aata hai. |
| `git rebase <base-branch>` | `🔴 [DANGEROUS]` | Current branch ke commits ko naye base commit ke upar replay karta hai (linear history). |
| `git rebase -i HEAD~N` | `🔴 [DANGEROUS]` | Interactive rebase! Commits squash, reword, drop, ya edit karne ke liye. |
| `git rebase --continue` | `🟡 [CAUTION]` | Rebase conflict manually resolve karne ke baad agle commit par aage badhne ke liye. |
| `git rebase --abort` | `🟡 [CAUTION]` | Rebase cancel karke initial state restore karta hai. |
| `git cherry-pick <commit-sha>` | `🟡 [CAUTION]` | Kisi doosri branch ke single commit ko current branch par copy karta hai. |
| `git cherry-pick --abort` | `🟡 [CAUTION]` | Cherry-pick conflict cancel karta hai. |

---

## 6. Remote Synchronization 🌐

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git remote -v` | `🟢 [SAFE]` | Configured remote repositories ke URLs (fetch & push) dikhata hai. |
| `git remote add <name> <url>` | `🟡 [CAUTION]` | Naya remote link (e.g. `origin`, `upstream`) register karta hai. |
| `git fetch <remote>` | `🟢 [SAFE]` | Remote se naye objects aur branch refs download karta hai (working directory touch nahi hoti!). |
| `git pull <remote> <branch>` | `🟡 [CAUTION]` | `git fetch` + `git merge FETCH_HEAD` ek sath run karta hai. |
| `git pull --rebase` | `🟡 [CAUTION]` | Remote changes fetch karke local unpushed commits ko unke upar rebase karta hai. |
| `git push <remote> <branch>` | `🟡 [CAUTION]` | Local branch commits ko remote repository par bhejta hai. |
| `git push -u origin <branch>` | `🟡 [CAUTION]` | Push karta hai aur local branch ke liye upstream tracking set karta hai. |
| `git push --force-with-lease` | `🔴 [DANGEROUS]` | Saafer force push. Tabhi overwrite karega agar kisi aur ne push nahi kiya ho. |
| `git push --force` | `🔴 [DANGEROUS]` | **EXTREME DANGER!** Remote history ko blindly overwrite kar deta hai. Team ka work destroy ho sakta hai! |

---

## 7. Stash & Temporary Workspace 📦

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git stash` (ya `git stash push`) | `🟡 [CAUTION]` | Tracked uncommitted changes ko temporary clipboard stack mein save kar deta hai. |
| `git stash -u` (include untracked) | `🟡 [CAUTION]` | Tracked ke sath new untracked files ko bhi stash karta hai. |
| `git stash list` | `🟢 [SAFE]` | Saved stashes ki list display karta hai (`stash@{0}`, `stash@{1}`). |
| `git stash pop` | `🟡 [CAUTION]` | Top stash apply karta hai aur stash stack se delete karta hai. |
| `git stash apply` | `🟡 [CAUTION]` | Stash apply karta hai lekin stack mein safe rakhta hai. |
| `git stash drop stash@{0}` | `🔴 [DANGEROUS]` | Specific stash ko permanently delete karta hai. |
| `git stash clear` | `🔴 [DANGEROUS]` | Saare stashes permanently wipe out kar deta hai. |

---

## 8. Undoing Changes & Disaster Recovery 🚑

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git restore <file>` | `🟡 [CAUTION]` | Working directory file ke uncommitted changes ko discard karta hai (last commit state). |
| `git restore --staged <file>` | `🟡 [CAUTION]` | File ko unstage karta hai (Index se nikalta hai, working directory untouched). |
| `git revert <commit-sha>` | `🟡 [CAUTION]` | **Safe Undo!** Purane commit ke inverse changes ka naya commit banata hai. Public history safe rehti hai. |
| `git reset --soft HEAD~1` | `🟡 [CAUTION]` | Last commit ko undo karta hai, lekin changes ko Staging area mein safe rakhta hai. |
| `git reset --mixed HEAD~1` (default) | `🟡 [CAUTION]` | Last commit undo karta hai, changes ko Working directory mein un-staged chhodta hai. |
| `git reset --hard HEAD~1` | `🔴 [DANGEROUS]` | **Destructive!** Last commit aur uncommitted changes ko discard kar deta hai. |
| `git clean -n` (dry run) | `🟢 [SAFE]` | Dikhata hai ki `git clean` chalane par kaunsi untracked files delete hongi. |
| `git clean -fd` | `🔴 [DANGEROUS]` | Saari untracked files aur directories ko disk se permanently delete karta hai! |

---

## 9. Git Plumbing (Deep Internals) 🔬

| Command | Safety | Description (Hinglish) |
|---|:---:|---|
| `git hash-object -w <file>` | `🟡 [CAUTION]` | File ka SHA hash calculate karta hai aur `.git/objects/` mein Blob save karta hai. |
| `git cat-file -p <sha>` | `🟢 [SAFE]` | Kisi bhi Git object (Blob, Tree, Commit, Tag) ka raw content pretty-print karta hai. |
| `git cat-file -t <sha>` | `🟢 [SAFE]` | SHA ka type return karta hai (`blob`, `tree`, `commit`, `tag`). |
| `git ls-tree <tree-sha>` | `🟢 [SAFE]` | Tree object ke andar stored directory contents aur unke blob hashes dikhata hai. |
| `git rev-parse HEAD` | `🟢 [SAFE]` | Current HEAD ka full 40-character SHA-1 hash output karta hai. |
| `git fsck --full` | `🟢 [SAFE]` | Git database ki integrity check karta hai aur dangling/orphaned objects dhundhta hai. |

