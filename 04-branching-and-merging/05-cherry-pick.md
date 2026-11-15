# Chapter 05: Surgical Commit Transplanting (`git cherry-pick` & Backporting)

---

## 1. Concept: Ye Kya Hai?

**`git cherry-pick`** Git ka ek surgical operation tool hai.

Sochiye aapke paas ek experimental branch hai jisme 20 commits hain aur code abhi production ke liye bilkul ready nahi hai. Lekin un 20 commits ke beech mein **Commit #7** ek critical security vulnerability ya crash ko fix karta hai.

Aap poori branch ko merge nahi kar sakte (kyunki baaki 19 commits broken hain).
Aapko sirf aur sirf us ek **single commit** ko nikaal kar `main` ya `production` branch par apply karna hai. 
Is surgical code copying process ko **Cherry-Picking** kehte hain.

---

## 2. Why: Iski Zarurat Kyu Padi? (The Backporting Dilemma)

Enterprise software engineering mein companies ek sath multiple active versions maintain karti hain:
- Version 3.0 (Latest production on `main`)
- Version 2.4 (Enterprise clients ke liye maintenance branch `release/v2.4`)
- Version 2.0 (Legacy banking clients ke liye `release/v2.0`)

Jab `main` branch par ek zero-day security bug fix hota hai, toh wo bugfix purane v2.4 aur v2.0 releases par bhi hona zaroori hai.
Aap `main` ko v2.4 mein merge nahi kar sakte kyunki hazaron naye features purane release ko break kar denge.
Aap us single bugfix commit ko pakadte hain aur use v2.4 aur v2.0 branches par **Cherry-Pick (Backport)** kar dete hain!

---

## 3. Real-World Example: Cherry-Picking a Merge Commit

Aap ek Senior Engineer hain. Ek junior developer ne ek critical feature branch merge ki thi jisme 5 commits the.
Lead aapse bolta hai: *"Is pure feature ko staging branch par cherry-pick karo!"*

Aapne merge commit hash liya `M` aur chalaya:
`git cherry-pick M`

Git ne turant error throw kiya:
`error: commit M is a merge but no -m option was given.`
`fatal: cherry-pick failed`

**Why did it fail?**
Merge commit ke **do parents** hote hain! Git ko pata nahi chalta ki wo Parent 1 ke respect mein diff apply kare ya Parent 2 ke respect mein.
Aapne chalaya:
`git cherry-pick -m 1 M`
Git ne Parent 1 ko baseline banakar pure feature ke diff ko ek single shot mein staging branch par apply kar diya!

---

## 4. Internal Working: What Happens During a Cherry-Pick?

Jab aap `git cherry-pick <commit-sha>` chalate hain, toh background mein Git kya karta hai?

1. **Delta Calculation:** Git us commit ke Tree aur uske direct parent ke Tree ke beech ka diff (patch) nikaalta hai:
   $$\Delta = \text{Commit}_{\text{Target}} - \text{Commit}_{\text{Parent}}$$
2. **Patch Application:** Git us delta ko aapke current active `HEAD` commit ke upar apply karne ki koshish karta hai.
3. **New Commit Object Creation:**
   - Git ek **Brand New Commit Object** generate karta hai!
   - Naye commit ka parent aapka current `HEAD` banta hai.
   - **Crucial Metadata Preservation:**
     - `Author` aur `AuthorDate`: Original developer ka hi rehta hai!
     - `Committer` aur `CommitDate`: **Aapka** naam aur current timestamp banta hai!
   - Isiliye naye commit ka **SHA hash completely different** hota hai!

```text
Feature Branch:
C1 ───► [C2 (Bugfix: SHA a1b2)] ───► C3 (Broken experiment)

Target Main Branch (Before Cherry-Pick):
M1 ───► M2 ◄── main (HEAD)

Command: git switch main && git cherry-pick a1b2

Target Main Branch (After Cherry-Pick):
M1 ───► M2 ───► [C2' (Bugfix: NEW SHA 99zz)] ◄── main (HEAD)
                (Exact same code diff, but new commit object!)
```

---

## 5. Commands: Cherry-Pick Control Suite

| Command | Safety | Description |
|---|:---:|---|
| `git cherry-pick <sha>` | `🟡 [CAUTION]` | Specific commit ko current branch par copy aur apply karta hai. |
| `git cherry-pick -x <sha>` | `🟡 [CAUTION]` | **Audit Standard:** Commit message ke footer mein automatically `(cherry picked from commit <sha>)` add karta hai. |
| `git cherry-pick -n <sha>` (no commit)| `🟡 [CAUTION]` | Code changes ko Staging Area mein load karta hai bina naya commit banaye (Allows custom edits!). |
| `git cherry-pick A..B` | `🟡 [CAUTION]` | Commit range (A ke baad se lekar B tak) ko sequentially cherry-pick karta hai. |
| `git cherry-pick -m 1 <merge-sha>` | `🟡 [CAUTION]` | Merge commit ko parent 1 ko mainline maankar cherry-pick karta hai. |
| `git cherry-pick --continue` | `🟡 [CAUTION]` | Conflict manually solve karne ke baad process complete karta hai. |
| `git cherry-pick --abort` | `🟡 [CAUTION]` | Cherry-pick conflict cancel karke repo ko exact pre-command state par restore karta hai. |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Executing cherry-pick with audit trace flag (-x):
$ git cherry-pick -x 4f8a192c
[main 7e9b2c3] fix(security): sanitize user input against sql injection
 Author: Alice <alice@work.com>
 Date: Fri Nov 13 18:20:10 2026 +0530
 1 file changed, 5 insertions(+)

# Inspecting the newly created commit:
$ git log -1
commit 7e9b2c3a1b2c3d4e5f67890123456789abcdef01
Author:     Alice <alice@work.com>
AuthorDate: Fri Nov 13 18:20:10 2026 +0530
Commit:     Bob TechLead <bob@work.com>
CommitDate: Sat Nov 14 14:45:00 2026 +0530

    fix(security): sanitize user input against sql injection

    (cherry picked from commit 4f8a192c7b8a9d0e1f2a3b4c5d6e7f8a9b0c1d2e)
```

---

## 7. Visual Explanation: The Cherry-Pick Operation

```text
ORIGINAL BRANCH (feature/wip):
Commit 10 ──► [Commit 11: SQL Fix] ──► Commit 12 (Broken AI engine)
                    │
                    │ git cherry-pick -x <Commit 11>
                    ▼
TARGET BRANCH (release/v2.0):
v2.0.4 ──────► [Commit 11': SQL Fix (New SHA)] ◄── release/v2.0
               (Author: Original Dev, Committer: Release Manager)
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Forgetting `-x` During Enterprise Backports:**
   - Mistake: Bina `-x` ke cherry-pick karna. Baad mein audit ke waqt kisi ko pata nahi chalta ki ye patch main branch ke kis commit se copy kiya gaya tha.
2. **Cherry-Picking Commits That Depend on Earlier Commits:**
   - Mistake: Commit 5 ko cherry-pick kiya, lekin Commit 5 ek variable use kar raha tha jo Commit 3 mein define hua tha!
   - Result: Code syntax error aur massive merge conflict throw karega. Always inspect dependencies!
3. **Overusing Cherry-Pick Instead of Merging:**
   - Mistake: Feature branch ke 20 commits ko ek-ek karke cherry-pick karna. Isse duplicate commits ban jaate hain aur future merges mein conflicts badh jaate hain.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar cherry-pick conflict throw kare:

```text
1. OBSERVE:    Command: `git cherry-pick 4f8a192`
               Error: "CONFLICT (content): Merge conflict in src/auth.js"
                      "error: could not apply 4f8a192... fix auth bug"

2. INSPECT:    State inspect karein:
               `git status` (Shows: "You are currently cherry-picking commit 4f8a192.")

3. UNDERSTAND: Jis code context par commit 4f8a192 original branch par likha gaya tha,
               target branch par wo context differ kar raha hai (lines aage piche hain).

4. PLAN:       Option A: File open karke conflict manually solve karein aur continue karein.
               Option B: Safe Exit: Operation cancel karein (`git cherry-pick --abort`).

5. EXECUTE:    Step 1: Open src/auth.js, resolve conflict markers (<<<<<<<).
               Step 2: git add src/auth.js
               Step 3: git cherry-pick --continue

6. VERIFY:     `git log -1` run karein; commit successfully applied display hoga!

7. PREVENT:    Hamesha ensure karein ki patch self-contained ho aur kisi unmerged
               refactoring par heavily dependent na ho.
```

---

## 10. Practice Lab: Hands-On Surgical Cherry-Pick & Backport Drill 💻

Chaliye Linux terminal par full cherry-pick lifecycle run karte hain:

```bash
# Step 1: Create sandbox
mkdir -p /tmp/git-cherry-pick-lab && cd /tmp/git-cherry-pick-lab
git init -b main

# Step 2: Create initial base
echo "Base System" > app.js
git add app.js && git commit -m "feat: initial system launch"

# Step 3: Branch out to an experimental unstable branch
git switch -c experimental-ai
echo "Unstable AI prototype" >> app.js
git commit -am "wip: incomplete neural network prototype"

# Step 4: Inside experimental branch, create a critical bugfix!
echo "function secureHash() { return 'sha256-verified'; }" >> app.js
git commit -am "fix(security): implement secure hash validation"
BUGFIX_SHA=$(git rev-parse HEAD)
echo "Bugfix SHA: $BUGFIX_SHA"

# Add another broken commit after the bugfix
echo "Broken GPU memory allocation" >> app.js
git commit -am "wip: broken gpu acceleration"

# Step 5: Switch back to main (We want ONLY the security fix!)
git switch main
cat app.js
# Notice: main has only "Base System".

# Step 6: Surgically Cherry-Pick the bugfix commit with -x audit trace!
git cherry-pick -x $BUGFIX_SHA

# Step 7: VERIFY THE SURGERY!
git log -1
cat app.js
# Notice: "secureHash" function is safely merged into main!
# The broken AI code was completely left behind!

# Cleanup
cd ~ && rm -rf /tmp/git-cherry-pick-lab
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Duplicate Commit PR Collision
> **Problem Statement:**
> Developer Alice ne feature branch par kaam karte waqt commit `C1` banaya.
> Developer Bob ko us commit ki turant zarurat thi, toh Bob ne apni branch par `git cherry-pick C1` chala diya.
> 
> Ab Alice aur Bob dono apni-apni PRs `main` branch mein merge karne ja rahe hain.
> Bob pareshan hai: *"Kyunki mere paas Alice ke commit ka duplicate copy naye SHA ke sath hai, kya Git merge karte waqt duplicate code lines add kar dega ya merge conflict fekega?"*
> 
> **Questions:**
> 1. Kya Git modern 3-way merge engine identical cherry-picked patches ko detect karke duplicate hone se bacha sakta hai?
> 2. `git cherry-pick` ke bajaye doosre developer se unmerged code lene ka clean standard workflow kya hona chahiye tha?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Patch-ID Intelligence:

**1. How Git Handles It:**
Git ke paas ek built-in algorithm hota hai jise **`git patch-id`** kehte hain.
Jab Alice aur Bob ki branches merge hongi, Git dekhega ki dono commits ke SHA hashes alag hain, LEKIN unka mathematical **Patch-ID (code diff)** 100% identical hai!
Modern Git ORT merge engine automatically duplicate patch ko recognize karke silently skip kar deta hai bina code duplicate kiye!

#### 2. The Clean Standard Workflow:
Alice ko apna common helper commit ek separate isolated branch par nikaal kar pehle merge karwana chahiye tha, ya Bob ko direct cherry-pick karne ke bajaye Alice ki feature branch se temporary base branch kaatni chahiye thi.
</details>

---

### 📌 Scenario Challenge 2: Modifying Code During Cherry-Pick (`-n` Flag)
> **Problem Statement:**
> Aapko `main` branch se ek commit cherry-pick karna hai `release-v1` par:
> Commit message: `feat(api): set timeout to 5000ms`.
> Lekin legacy `release-v1` branch par server slow hone ki wajah se timeout 5000ms nahi, balki **8000ms** hona chahiye.
> 
> Aap nahi chahte ki pehle commit bane aur fir aap naya commit karke 8000ms karein.
> 
> **Question:** `git cherry-pick` ke kaunse flag se aap code ko bina commit kiye staging area mein la sakte hain taaki aap commit se pehle value edit kar sakein?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `git cherry-pick -n` (or `--no-commit`)

```bash
# Step 1: Cherry-pick without creating a commit object:
git cherry-pick -n <commit-sha>

# Step 2: Edit the file to adjust the value:
sed -i 's/5000/8000/' config.js

# Step 3: Stage and make your custom commit:
git add config.js
git commit -m "feat(api): set timeout to 8000ms for legacy release"
```
Aapne commit ke patch ko adaptively modify karke release-specific commit bana diya!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: What is the difference between `git merge` and `git cherry-pick`?
**Answer:**
- `git merge` integrates the **entire historical lineage** of two branches starting from their common ancestor, producing a 2-parent merge commit (or advancing via fast-forward).
- `git cherry-pick` extracts the isolated **diff patch of a single specific commit** and reapplies it onto the current branch as a brand-new single-parent commit object, completely ignoring the rest of the source branch's history.

### Q2: Why does a cherry-picked commit have a different SHA-1 hash from the original commit?
**Answer:** A commit's SHA hash is computed over its root Tree hash, parent commit hash, author metadata, committer metadata, committer timestamp, and commit message. When a commit is cherry-picked, its parent becomes the current branch's HEAD, the committer becomes the person executing the cherry-pick, and the committer timestamp is set to the current time. Because the inputs to the cryptographic hash function change, the resulting commit SHA must be different.

### Q3: Why is the `-x` flag recommended for enterprise cherry-picks?
**Answer:** The `-x` flag automatically appends a standardized footnote line `(cherry picked from commit <original_sha>)` to the commit message. In enterprise multi-version maintenance (backporting security fixes across legacy release branches), this provides an immutable audit trail that enables compliance scanners and release engineers to track which upstream patch was backported and verify that all CVE vulnerabilities are accounted for.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Automated Backport Bots via GitHub Actions:**
   Tech Lead enterprise repositories mein automated backport bots (jaise **Mergify** ya **backport-action**) configure karta hai. Jab koi PR `main` par merge hoti hai aur developer us par label lagata hai `backport to v2.4`, bot automatically background mein ek PR open karta hai jo us commit ko `release/v2.4` branch par `cherry-pick -x` karke unit tests verify kar leta hai.
2. **Preventing Cherry-Pick Drift:**
   Tech Lead team code reviews mein strictly monitor karta hai ki developers feature branches ke beech random cherry-picking na karein, kyunki excessive cherry-picking commits ke dependency graph ko confuse kar deta hai.

---

### 🔗 Official Documentation References:
- [Git SCM - git-cherry-pick Documentation](https://git-scm.com/docs/git-cherry-pick)
- [Git SCM - Rebasing and Cherry-Picking Workflows](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)

