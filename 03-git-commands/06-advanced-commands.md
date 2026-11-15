# Chapter 06: Advanced Power Tools (`stash`, `clean`, `archive`, `bundle` & `bisect`)

---

## 1. Concept: Ye Kya Hai?

Ab tak aapne daily workflow ke core commands master kiye hain. Lekin jab production pressure, emergency interruptions, ya complex regression bugs aate hain, tab ek Senior Lead ke toolkit mein 5 advanced power tools hote hain:

1. **`git stash`**: Apne uncommitted code ko temporarily ek safe storage stack mein hide karna taaki aap clean slate par doosra kaam kar sakein.
2. **`git clean`**: Working directory se saari untracked garbage files ko ek jhatke mein safely purge karna.
3. **`git archive`**: Pure project ka production-ready zip ya tarball export karna (bina kisi hidden `.git` folder ke).
4. **`git bundle`**: Bina kisi internet ya GitHub server ke, ek single `.bundle` file ke zariye poori repository history offline share karna.
5. **`git bisect`**: Binary search algorithm use karke lakho commits ke beech se kisi hidden bug ko automatically dhoondh nikalna.

---

## 2. Why: Iski Zarurat Kyu Padi?

- **The Interruption Problem:** Aap 4 files mein aadha-adhura (broken) code likh kar baithe the. Achanak client outage hua aur bola gaya: *"Main branch checkout karke 2 minute mein hotfix deploy karo!"* Aap broken code commit nahi kar sakte, aur discard bhi nahi kar sakte. Yahan **`git stash`** aapka savior banta hai.
- **The Binary Bug Hunt:** Kisi ne 6 mahine pehle ek aisa commit kiya tha jisne reporting calculation 2% se galat kar di. 400 commits ko ek-ek karke manually test karne mein 3 din lagenge. **`git bisect`** ise 8 steps (5 minute) mein solve kar deta hai.

---

## 3. Real-World Example: Fully Automated Bug Hunting with `git bisect run`

Aap ek High-Frequency Trading firm ke Staff Engineer hain.
Tests fail ho rahe hain: `AssertionError: Order processing delay > 5ms`.
Yeh bug pichle 300 commits ke dauran kisi point par introduce hua tha.

Aap manual testing nahi karte. Aap ek automated 3-line bash script likhte hain `test_perf.sh` jo exit code 0 (good) ya 1 (bad) return karti hai.

Aap terminal par chalate hain:
```bash
git bisect start HEAD v2.1.0
git bisect run ./test_perf.sh
```

**The Magic:**
Git automated tarike se binary search chalata hai:
- 150th commit checkout karta hai, script run karta hai -> Pass (`good`)!
- 225th commit checkout karta hai, script run karta hai -> Fail (`bad`)!
- 187th commit...
Sirf **9 automated iterations** ke baad Git terminal par culprit commit throw kar deta hai:
`7e9b2c3 is the first bad commit: refactor(network): replace buffer pooling`
Aapke 2 din ka manual debugging 90 seconds mein automate ho gaya!

---

## 4. Internal Working: Stash and Bundle Deep Dive

### 1. `git stash` Internals:
`git stash` koi temporary text file nahi hoti! Internally Git stash ko **Commit Objects** ke roop mein store karta hai!
Jab aap `git stash -u` chalate hain, Git 3 commits create karta hai:
1. **Commit 1:** Current Staging Area ka snapshot.
2. **Commit 2:** Working directory modified tracked files ka snapshot.
3. **Commit 3:** Untracked files ka snapshot.
In commits ko `.git/refs/stash` reference aur uske reflog stack (`stash@{0}`, `stash@{1}`) mein track kiya jata hai.

```text
git stash Commit Topology:
               [Parent Commit (HEAD)]
                     ▲        ▲
                    /          \
    [Stash Index Commit]    [Stash Working Tree Commit] ◄── refs/stash (stash@{0})
                               ▲
                              /
                   [Stash Untracked Commit]
```

### 2. `git bundle` (The Sneakernet Superpower):
Defense ya Banking enterprises mein production servers internet se completely disconnected (Air-Gapped) hote hain. Wahan GitHub access nahi hota.
`git bundle create app.bundle --all` ek single binary file generate karta hai jisme raw packfile aur references hote hain.
Aap us file ko USB drive se air-gapped server par le jaakar direct clone kar sakte hain:
`git clone app.bundle my-app`
Git ise bilkul ek remote GitHub server ki tarah treat karta hai!

---

## 5. Commands: Advanced Power Tools Suite

| Command | Safety | Description |
|---|:---:|---|
| `git stash push -u -m "msg"` | `🟡 [CAUTION]` | Tracked aur Untracked (`-u`) dono changes ko named stash mein store karta hai. |
| `git stash list` | `🟢 [SAFE]` | Saved stashes ki stack list print karta hai (`stash@{0}`, `stash@{1}`). |
| `git stash pop` | `🟡 [CAUTION]` | Top stash apply karta hai aur stash stack se delete kar deta hai. |
| `git stash apply` | `🟡 [CAUTION]` | Stash apply karta hai lekin stash stack mein preserve rakhta hai. |
| `git stash branch <name>` | `🟡 [CAUTION]` | Nayi branch create karke stash ko us par clean pop karta hai (Conflict prevention!). |
| `git clean -n` (Dry Run) | `🟢 [SAFE]` | **Must Run First:** Dikhata hai kaunsi untracked files delete hongi (Bina delete kiye). |
| `git clean -fd` | `🔴 [DANGEROUS]` | Saari untracked files aur directories ko disk se permanently wipe out karta hai! |
| `git archive -o release.zip HEAD` | `🟢 [SAFE]` | Clean production release zip create karta hai (bina `.git` metadata ke). |
| `git bundle create repo.bundle --all`| `🟢 [SAFE]` | Poori repository ko single offline binary packfile mein export karta hai. |
| `git bisect start / bad / good` | `🟡 [CAUTION]` | Binary search regression debugging session shuru karta hai. |
| `git bisect reset` | `🟡 [CAUTION]` | Bisect session end karke original branch state par wapas return karta hai. |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Stashing with untracked files:
$ git stash push -u -m "WIP: payment redesign"
Saved working directory and index state On main: WIP: payment redesign

# Stash listing:
$ git stash list
stash@{0}: On main: WIP: payment redesign
stash@{1}: On feature/auth: test configs

# Git clean dry run vs execute:
$ git clean -n -d
Would remove build/
Would remove temp-debug.log

$ git clean -f -d
Removing build/
Removing temp-debug.log
```

---

## 7. Visual Explanation: `git bisect` Binary Search Traversal

```text
Commit History (100 Commits):
Commit 1 (Known GOOD: v1.0.0) ───► ... ───► Commit 100 (Known BAD: HEAD)

Step 1: Git checks out Commit 50 (Midpoint). You test -> GOOD!
        Bug must be between 51 and 100!

Step 2: Git checks out Commit 75. You test -> BAD!
        Bug must be between 51 and 74!

Step 3: Git checks out Commit 62. You test -> BAD!
        Bug must be between 51 and 61!
...
Within 7 steps, Git pinpoints the EXACT 1 culprit commit!
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Running `git clean -fd` Without `-n` First:**
   - Mistake: Terminal saaf karne ke liye blind `git clean -fd` chala dena.
   - Disaster: Agar aapne koi nayi file likhi thi jo abhi tak `git add` nahi hui thi, wo **disk se permanently delete ho jayegi!** Iska koi undo ya reflog nahi hota. Always dry-run with `-n`!
2. **Stashing Without the `-u` (Untracked) Flag:**
   - Mistake: Sirf `git stash` chalana.
   - Result: Nayi create ki gayi files working tree mein hi reh jaati hain aur branch switch karte waqt conflicts create karti hain. Always use `git stash -u`!
3. **Forgetting to End Bisect with `git bisect reset`:**
   - Leaving the repository stuck in a detached bisect state.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar `git stash pop` conflict throw kare:

```text
1. OBSERVE:    Command: `git stash pop`
               Error: "CONFLICT (content): Merge conflict in src/app.js"
                      "The stash entry is kept in case you need it again."

2. INSPECT:    State check karein:
               `git status`
               Notice: Conflict markers in app.js, and stash is NOT dropped!

3. UNDERSTAND: Jis branch par aap stash pop kar rahe hain, uska code stash karte
               waqt wale code se differ kar chuka hai.

4. PLAN:       Option A: File khol kar conflict markers solve karein, stage karein,
                         aur stash ko manually drop karein (`git stash drop`).
               Option B: Safe Recovery: Stash ko ek dedicated isolated branch
                         par unpack karein jahan koi conflict na ho!

5. EXECUTE:    git merge --abort (ya restore file)
               git stash branch safe-stash-branch stash@{0}

6. VERIFY:     Git ek nayi branch banayega usi commit par jahan stash create hua tha,
               stash pop clean succeed ho jayega, aur stash auto-drop ho jayega!

7. PREVENT:    Stash apply karne se pehle hamesha ensure karein ki working tree
               clean ho aur aap correct branch context mein hon.
```

---

## 10. Practice Lab: Mastering Stash, Bundle & Bisect 💻

Chaliye Linux terminal par full suite hands-on drill run karte hain:

```bash
# Step 1: Create sandbox
mkdir -p /tmp/git-advanced-lab && cd /tmp/git-advanced-lab
git init -b main

# Step 2: Create initial clean state
echo "function calculate() { return 10; }" > app.js
git add app.js && git commit -m "feat: stable calculation v1"
GOOD_COMMIT=$(git rev-parse HEAD)

# Step 3: Accumulate some commits, introducing a bug in commit 4
for i in {2..5}; do
  if [ $i -eq 4 ]; then
    echo "function calculate() { return -999; } // BUG!" > app.js
    git commit -am "feat: update calculation logic v$i (CULPRIT)"
  else
    echo "// Update step $i" >> app.js
    git commit -am "chore: regular update $i"
  fi
done
BAD_COMMIT=$(git rev-parse HEAD)

# Step 4: Run Automated Git Bisect!
git bisect start $BAD_COMMIT $GOOD_COMMIT

# We tell bisect how to test: if calculate returns -999, it is BAD (exit 1), else GOOD (exit 0):
git bisect run bash -c 'grep -q -- "-999" app.js && exit 1 || exit 0'

# Output: Git immediately prints:
# <hash> is the first bad commit: feat: update calculation logic v4 (CULPRIT)

# Reset bisect back to normal:
git bisect reset

# Step 5: Test Git Bundle (Offline Repo Sharing)
git bundle create /tmp/my-project.bundle --all
# Notice: A single binary file was created in /tmp/!

# Clone directly from this single file:
cd /tmp && git clone my-project.bundle cloned-from-bundle
cd cloned-from-bundle && git log --oneline
# Boom! Full repository history intact from a single offline file!

# Cleanup
cd ~ && rm -rf /tmp/git-advanced-lab /tmp/my-project.bundle /tmp/cloned-from-bundle
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Untracked Directory Purge Trap
> **Problem Statement:**
> Ek build script fail hone ke baad working tree mein 15 giant build folders ban gaye:
> `build/`, `dist/`, `coverage/`, `.cache/`, aur 50 `.tmp` files.
> 
> Developer ne chalaya:
> `git clean -f`
> Lekin files delete hone ke baad bhi `build/`, `dist/`, aur `coverage/` folders wahi ke wahi khade rahe!
> 
> **Questions:**
> 1. `git clean -f` ne directories ko delete kyu nahi kiya?
> 2. Saari directories ko recursively remove karne aur sath mein `.gitignore` mein ignored files ko bhi force purge karne ka command flag combination kya hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Flag Mechanics:

**1. Why directories remained:**
Default taur par `git clean` sirf **files** par act karta hai. Directories ko delete hone se rokna Git ka safe default behavior hai taaki accidentally pura folder na ud jaye.

#### 2. The Comprehensive Clean Command:
Directories ko delete karne ke liye **`-d`** flag aur ignored files ko bhi purge karne ke liye **`-x`** flag zaroori hota hai:

```bash
# Step 1: ALWAYS Dry-run first!
git clean -ndx

# Step 2: Execute actual purge:
git clean -fdx
```

#### Flag Breakdown:
- `-f` (force): Mandated by Git to prevent accidental execution.
- `-d`: Untracked directories ko bhi recursively remove karta hai.
- `-x`: Saari un-tracked files ke sath sath `.gitignore` mein ignored build artifacts (`node_modules/`, `dist/`) ko bhi wipe out karta hai!
Repository wapas 100% pristine clean clone state par aa jaati hai.
</details>

---

### 📌 Scenario Challenge 2: The Stash Selective File Restore
> **Problem Statement:**
> Aapne `git stash push -m "WIP"` mein 10 modified files stash ki thi:
> `src/auth.js`, `src/billing.js`, `src/dashboard.js`, etc.
> 
> Ab aapko pure 10 files wapas pop nahi karni!
> Aapko sirf aur sirf `src/billing.js` file ka stashed version restore karna hai, aur baaki 9 files ko stash stack mein hi safe rakhna hai!
> 
> **Question:** `git checkout` ya `git restore` use karke stash object ke andar se kisi specific single file ko kaise extract karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: Restoring Single File from Stash

Kyunki stash internally ek commit object hai, aap stash reference se kisi bhi specific file ko directly checkout ya restore kar sakte hain!

```bash
# Using modern git restore:
git restore --source=stash@{0} -- src/billing.js

# ya using git checkout:
git checkout stash@{0} -- src/billing.js
```

#### Result:
Sirf `src/billing.js` aapki working directory mein wapas load ho jayegi, aur pura stash `stash@{0}` stack mein 100% safe aur un-popped rahega!
</details>

---

### 📌 Scenario Challenge 3: Creating a Production Release Zip via `git archive`
> **Problem Statement:**
> DevOps team ko production deployment ke liye ek clean `.zip` archive chahiye jisme:
> - Sirf release tag `v2.4.0` ka code ho.
> - Hidden `.git/` folder aur `.gitignore` include **NA** ho.
> - Zip ke andar saari files ek root folder `my-service-v2.4.0/` ke andar nested hon (taaki extract karne par files messy na failein).
> 
> **Question:** `git archive` command ka syntax kya hoga is archive ko generate karne ke liye?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `git archive` with Prefix Flag

```bash
git archive --format=zip --prefix=my-service-v2.4.0/ v2.4.0 -o my-service-v2.4.0.zip
```

#### Why this is enterprise best practice:
- `--format=zip`: Generates clean zip archive.
- `--prefix=my-service-v2.4.0/`: Ensures all extracted files live inside a polite parent folder.
- Zero Git metadata: Zip 70% choti banti hai aur server par koi internal Git objects leak nahi hote.
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: How does `git stash` store modifications internally?
**Answer:** `git stash` creates two (or three, if untracked files are included via `-u`) special commit objects in `.git/objects/`. The first commit represents the Staging Index state, the second commit represents the Working Directory state (with the first commit as its second parent), and the optional third commit stores untracked files. The reference pointer is stored under `.git/refs/stash` and managed via a reflog stack.

### Q2: How does `git bisect` achieve $O(\log N)$ search complexity in finding a regression bug?
**Answer:** `git bisect` applies a binary search algorithm over the topological sort of the Directed Acyclic Graph (DAG) between a known good commit and a known bad commit. By testing the midpoint commit node, Git eliminates half of the remaining commit graph in each step, allowing a bug across 1,000 commits to be pinpointed in approximately $\log_2(1000) \approx 10$ steps.

### Q3: What is a Git Bundle and when is it used over `git push`?
**Answer:** A Git Bundle is a single binary container file generated via `git bundle create`. It encapsulates raw packfile objects and repository reference heads. It is utilized in **Air-Gapped environments** (defense networks, highly secure bank intranets) where machines have no internet or remote Git server connectivity, allowing complete repositories and branch updates to be transferred via physical media (USB/DVD) and cloned directly.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Automated Overnight CI Bisect Regression Testing:**
   Jab koi sporadic performance ya memory leak bug report hota hai jiska origin unknown ho, Tech Lead ek nightly Jenkins/GitHub Actions workflow schedule karta hai jo `git bisect run ./benchmark_suite.sh` execute karta hai. Subah aate hi Team Lead ko exact culprit commit aur author ki automated Slack notification mil jaati hai.
2. **Release Tarball Automation with `.gitattributes`:**
   Tech Lead repository root par `.gitattributes` configure karta hai with `export-ignore` rules:
   ```text
   tests/ export-ignore
   .github/ export-ignore
   docs/ export-ignore
   ```
   Jab `git archive` chalta hai, toh test files aur internal CI scripts production release zip se automatically exclude ho jaati hain.

---

### 🔗 Official Documentation References:
- [Git SCM - Git Stash Documentation](https://git-scm.com/docs/git-stash)
- [Git SCM - git-clean Documentation](https://git-scm.com/docs/git-clean)
- [Git SCM - git-bisect Documentation](https://git-scm.com/docs/git-bisect)
- [Git SCM - git-bundle Documentation](https://git-scm.com/docs/git-bundle)

