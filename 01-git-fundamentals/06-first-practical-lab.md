# Chapter 06: Capstone Practical Lab (Module 01 Integration)

---

## 1. Concept: Ye Kya Hai?

Ye Module 01 ka **Hands-on Capstone Lab** hai. Pichle 5 chapters mein aapne jo bhi theoretical aur internal concepts padhe hain—Distributed Architecture, Content-Addressable Hashing, Three Trees, Staging Area, aur `.git` Directory anatomy—unhe hum is lab mein ek single, realistic enterprise project ke roop mein end-to-end execute karenge.

Is lab mein koi shortcuts nahi honge. Hum scratch se terminal mein ek real microservice structure banayenge, intentional mistakes karenge, aur unhe internal inspection se diagnose karenge.

---

## 2. Why: Iski Zarurat Kyu Padi?

Theory padh lene ke baad aksar developers terminal par aate hi confuse ho jaate hain. Jab tak aap apne haath se:
- Git objects create nahi karenge,
- Staging area ke diffs ko compare nahi karenge,
- Aur `.git` ke files ko live update hote nahi dekhenge,
tab tak aapka mental model solid nahi hoga.

---

## 3. Real-World Scenario: Bootstrapping `auth-service`

Aap ek Senior Engineer hain jise company ke naye **Authentication Microservice** (`auth-service`) ki foundation rakhni hai.
Aapki responsibilities hain:
1. Repository initialize karna modern conventions (`main` branch) ke sath.
2. Standard `.gitignore` configure karna taaki sensitive files aur temporary logs leak na hon.
3. Clean, atomic commits banana.
4. Git ke internal database (`.git/objects`) ko verify karna ki koi redundant data create nahi hua.

---

## 4. Internal Working: Lab Architecture & State Map

```text
Phase 1: Initialize Repo ────────► Creates .git/ folder & sets HEAD -> refs/heads/main
Phase 2: Add Config & Secrets ──► Configures .gitignore, blocks accidental staging
Phase 3: Craft Atomic Commits ──► Generates Blobs, Trees, and Commit DAG
Phase 4: Inspect Plumbing ──────► Uses git cat-file to verify cryptographic hashes
Phase 5: Induce & Fix Errors ──► Tests forensic recovery of accidentally staged files
```

---

## 5. Commands Reference for this Lab

| Command | Safety | Purpose in Lab |
|---|:---:|---|
| `mkdir`, `cd`, `echo` | `🟢 [SAFE]` | Bash environment setup |
| `git init -b main` | `🟡 [CAUTION]` | Initialize repository with modern default branch |
| `git status -s` | `🟢 [SAFE]` | Rapid status verification |
| `git add <file>` | `🟡 [CAUTION]` | Stage specific atomic files |
| `git commit -m "..."` | `🟡 [CAUTION]` | Create immutable snapshot |
| `git diff --staged` | `🟢 [SAFE]` | Pre-commit sanity check |
| `git log --oneline --graph`| `🟢 [SAFE]` | Commit graph inspection |
| `git cat-file -p <hash>` | `🟢 [SAFE]` | Inspect Git raw object storage |

---

## 6. Step-by-Step Practical Lab Execution 💻

Har step ko apne Linux ya macOS terminal par sequence mein run karein:

### Phase 1: Environment Setup & Clean Initialization
```bash
# Step 1.1: Create isolated lab directory
mkdir -p ~/git-labs/auth-service && cd ~/git-labs/auth-service

# Step 1.2: Initialize git with main branch
git init -b main

# Step 1.3: Confirm .git internals were generated
ls -la .git
cat .git/HEAD
# Expected output: ref: refs/heads/main
```

---

### Phase 2: Production `.gitignore` Configuration
Real projects mein unwanted files commit hone se rokna pehla dharma hai:

```bash
# Step 2.1: Create application files
mkdir -p src config logs
echo 'console.log("Auth Service Starting...");' > src/index.js
echo 'PORT=8080' > config/app.env
echo 'SUPER_SECRET_JWT_KEY=shhh12345' > config/secrets.env
echo 'Debug log trace: 2026-11-14' > logs/debug.log

# Step 2.2: Check status before .gitignore
git status -s
# Notice: All files are untracked (??)

# Step 2.3: Create comprehensive .gitignore
cat << 'EOF' > .gitignore
# Secrets and Credentials
*.env
config/secrets.env

# Logs and Debugging
logs/
*.log

# OS & Editor Files
.DS_Store
.idea/
.vscode/
EOF

# Step 2.4: Verify .gitignore behavior
git status -s
# CRITICAL CHECK: Notice that config/secrets.env and logs/debug.log are GONE from status!
# Only src/index.js, config/app.env, and .gitignore are visible!
```

---

### Phase 3: Crafting Atomic Commits
Hum sab kuch ek sath commit nahi karenge. 2 alag atomic commits banayenge:

```bash
# Step 3.1: Stage and commit the .gitignore first
git add .gitignore
git diff --staged
git commit -m "chore: configure repository gitignore rules"

# Step 3.2: Stage only the application core
git add src/index.js
git commit -m "feat(core): initialize auth service entry point"

# Step 3.3: Verify commit history
git log --oneline --graph --decorate
```

Expected terminal log:
```text
* 7a2b3c1 (HEAD -> main) feat(core): initialize auth service entry point
* 1f8e4d2 chore: configure repository gitignore rules
```

---

### Phase 4: Git Internals Live Forensic Inspection
Ab dekhte hain Git ne parde ke peeche kya kiya:

```bash
# Step 4.1: Find the latest commit hash
LATEST_COMMIT=$(git rev-parse HEAD)
echo "Latest Commit Hash: $LATEST_COMMIT"

# Step 4.2: Inspect the commit object itself
git cat-file -t $LATEST_COMMIT
# Output: commit
git cat-file -p $LATEST_COMMIT
# Notice: It shows tree <hash>, parent <hash>, author, committer, and commit message!

# Step 4.3: Extract the Tree hash from the commit and inspect it
TREE_HASH=$(git cat-file -p $LATEST_COMMIT | grep tree | awk '{print $2}')
echo "Tree Hash: $TREE_HASH"
git cat-file -p $TREE_HASH
# Notice: It lists the directory contents (e.g. 100644 blob <hash> .gitignore, 040000 tree <hash> src)

# Step 4.4: Inspect the blob of src/index.js
BLOB_HASH=$(git rev-parse HEAD:src/index.js)
echo "Blob Hash: $BLOB_HASH"
git cat-file -p $BLOB_HASH
# Output: console.log("Auth Service Starting...");
```

---

### Phase 5: Error Induction & Forensic Recovery Drill
Ab hum intentionally ek galti karenge jo real teams mein aksar hoti hai:

```bash
# Step 5.1: A junior developer creates a temporary test script and accidentally stages it
echo 'const testToken = "fake-token";' > test-debug.js
git add test-debug.js

git status -s
# Output shows: A  test-debug.js (Staged!)

# Step 5.2: OBSERVE & INSPECT
# Developer realizes: "Wait! Ye test script mujhe commit nahi karna tha!"
git diff --staged --name-only
# Shows test-debug.js

# Step 5.3: EXECUTE UNSTAGE (Safe Recovery)
git restore --staged test-debug.js

# Step 5.4: VERIFY
git status -s
# Output shows: ?? test-debug.js (Safely moved back to Untracked without losing code!)

# Step 5.5: Safely discard the test file from disk
rm test-debug.js
```

---

## 7. Visual Summary of the Lab Result

```text
Commit 1: [Tree: .gitignore]
             ▲
             │ (Parent Pointer)
Commit 2: [Tree: .gitignore + src/index.js] ◄─── HEAD & main (Refs)
```

---

## 8. Common Mistakes in Lab

1. **Creating `.gitignore` After Staging Files:**
   Agar aapne file already `git add` kar di, aur uske baad use `.gitignore` mein daala, toh Git use ignore **nahi** karega! Git already tracked files ko ignore nahi karta. Use pehle `git rm --cached <file>` karna padta hai.
2. **Forgetting to Check `git diff --staged`:**
   Commit karne se pehle hamesha `git diff --staged` check karein taaki confirm ho ki sirf wahi code commit ho raha hai jo intended tha.

---

## 9. Troubleshooting Scenario: Untracked Files Clean

Agar lab ke dauran unnecessary temporary files ban jayein:
```bash
# Safe dry-run check (kuch delete nahi karega):
git clean -n

# Remove untracked files safely:
git clean -f
```

---

## 10. Lab Completion Challenge 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Parent Verification & Database Growth Drill
> **Mastery Task:**
> 1. `auth-service` ke andar `src/auth.js` file banayein jisme dummy login function ho.
> 2. Use stage karein aur commit karein message ke sath: `"feat(auth): add login handler"`
> 3. `git cat-file -p HEAD` se verify karein ki is naye commit ka parent commit pichla commit (`feat(core): initialize auth service entry point`) hai ya nahi!
> 4. `git count-objects -v` run karke dekhein ki ab total kitne loose objects `.git/objects/` mein ban chuke hain.

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Step-by-Step Solution:

```bash
# Step 1: Create auth.js
echo 'function login(user, pass) { return true; }' > src/auth.js

# Step 2: Stage and commit
git add src/auth.js
git commit -m "feat(auth): add login handler"

# Step 3: Inspect new commit object
git cat-file -p HEAD

# Look at the output:
# tree <new_tree_sha>
# parent 7a2b3c1...   <--- NOTICE THIS! Parent pointer points directly to previous commit!
# author ...
# committer ...
# feat(auth): add login handler

# Step 4: Verify count of objects
git count-objects -v
# Notice that count has increased by exactly:
# 1 new Blob (src/auth.js) + 1 new Tree (src/) + 1 new Root Tree + 1 new Commit = +4 objects!
```
</details>

---

### 📌 Scenario Challenge 2: The Tracked Secrets Emergency
> **Problem Statement:**
> Ek developer ne `config/secrets.env` file ko galti se tab add kar diya tha jab `.gitignore` set nahi tha.
> Baad mein usne `.gitignore` mein `config/secrets.env` likh diya.
> Lekin jab bhi wo `config/secrets.env` mein password change karta hai, `git status` abhi bhi chillata hai:
> `modified: config/secrets.env`
> 
> Developer pareshan hai: *".gitignore mein likhne ke baad bhi Git ise ignore kyu nahi kar raha?"*
> 
> **Question:** Git ko is file ko track karna band karne ke liye bolna hai, LEKIN file ko local disk se physically delete NAHI karna hai! Kaunsi exact command use karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `git rm --cached`

Jab koi file **already Git repository ke index mein tracked** hoti hai, toh `.gitignore` us par apply nahi hota! `.gitignore` sirf untracked files ko ignore karta hai.

#### The Command:
```bash
git rm --cached config/secrets.env
git commit -m "chore: stop tracking secrets.env without deleting from disk"
```

#### Why `--cached` is Critical?
- Agar aap sirf `git rm config/secrets.env` chalayenge, toh Git aapke laptop ki disk se bhi file delete kar dega!
- `--cached` flag Git ko bolta hai: *"Sirf Git ke index/repository se tracking hatao, local hard drive par file jaisi hai waisi hi chhod do."*
Ab aage se `.gitignore` rule successfully apply ho jayega!
</details>

---

## 11. Module 01 Self-Assessment Checklist

Bina solution dekhe in questions ke answers terminal par demonstrate karein:
- [ ] Kya aap kisi bhi commit hash ka root tree inspect kar sakte hain?
- [ ] Kya aap Staging Area mein bina code delete kiye changes unstage kar sakte hain?
- [ ] Kya aap `.git/HEAD` aur `.git/refs/heads/main` ke difference ko explain kar sakte hain?
- [ ] Kya aap verify kar sakte hain ki `.gitignore` rules kaam kar rahe hain?

Agar haan, toh **Congratulations!** Aapne **Module 01: Git Fundamentals** ko fully master kar liya hai. 
Ab aap [PROGRESS-TRACKER.md](../PROGRESS-TRACKER.md) mein Module 01 ke saare chapters ko `[*]` (Mastered) mark kar sakte hain aur **Module 02: Git Internals** par aage badh sakte hain!

