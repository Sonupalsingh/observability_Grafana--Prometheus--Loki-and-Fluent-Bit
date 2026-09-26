# GitHub Guide – Push, Update and Deploy (RHEL & Ubuntu)

Every step to put this project on GitHub **from a Linux server**, keep it updated, and deploy it to other servers. All commands work on **RHEL / Rocky / AlmaLinux / CentOS** and **Ubuntu / Debian**. Where they differ, both versions are shown.

- [Part A – Install and configure Git](#part-a--install-and-configure-git)
- [Part B – Create the GitHub repository](#part-b--create-the-github-repository)
- [Part C – Set up GitHub authentication](#part-c--set-up-github-authentication)
- [Part D – Prepare the project folder](#part-d--prepare-the-project-folder)
- [Part E – First push](#part-e--first-push)
- [Part F – Check the result on GitHub](#part-f--check-the-result-on-github)
- [Part G – Tag a release (optional)](#part-g--tag-a-release-optional)
- [Part H – Deploy from GitHub to a server](#part-h--deploy-from-github-to-a-server)
- [Part I – Everyday workflow: change, push, update](#part-i--everyday-workflow-change-push-update)
- [Part J – If the GitHub repo already has files](#part-j--if-the-github-repo-already-has-files)
- [Part K – Keeping secrets out of GitHub](#part-k--keeping-secrets-out-of-github)
- [Part L – Troubleshooting](#part-l--troubleshooting)
- [Quick reference](#quick-reference)

---

## Part A – Install and configure Git

### A1. Install Git

```bash
# RHEL / Rocky / AlmaLinux / CentOS
sudo dnf install -y git

# Ubuntu / Debian
sudo apt-get update && sudo apt-get install -y git
```

Check it:

```bash
git --version        # e.g. git version 2.43.5
```

### A2. Tell Git who you are

This name and email appear on every commit. Use the email address of your GitHub account.

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

---

## Part B – Create the GitHub repository

1. In a browser, go to <https://github.com/new>.
2. **Repository name:** `observability-stack`, or any name you like.
3. **Visibility:**
   - **Private:** only you and people you invite can see it (recommended for company use).
   - **Public:** anyone can see it.
4. ⚠️ **Leave these UNCHECKED:** "Add a README file", "Add .gitignore" and "Choose a license". The project already has these files, and adding them here causes a conflict on the first push.
5. Click **Create repository**.
6. Note the two URLs GitHub shows:
   - HTTPS: `https://github.com/<your-user>/observability-stack.git`
   - SSH: `git@github.com:<your-user>/observability-stack.git`

---

## Part C – Set up GitHub authentication

GitHub does **not** accept your account password for `git push`. Choose **one** of the two methods below. SSH is recommended for a server you push from often.

### Option 1: SSH key (recommended)

**1. Create a key on the server**

```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

**2. Add the key to GitHub**

Copy the whole line that `cat` printed (it starts with `ssh-ed25519`). On GitHub, click your profile picture → **Settings → SSH and GPG keys → New SSH key**, give it a title (for example the server name), paste the key and click **Add SSH key**.

**3. Test the connection**

```bash
ssh -T git@github.com
```

Type `yes` if asked about the host fingerprint. Expected output:

```
Hi <your-user>! You've successfully authenticated, but GitHub does not provide shell access.
```

From now on, use the **SSH URL** (`git@github.com:...`).

### Option 2: Personal Access Token (HTTPS)

**1. Create a token on GitHub**

Click your profile picture → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.

- **Repository access:** Only select repositories → `observability-stack`
- **Permissions → Contents:** Read and write
- Click **Generate token** and copy it. It is only shown once.

**2. Make Git remember the token (optional)**

```bash
git config --global credential.helper store
```

The first time you push, enter your GitHub username, and **paste the token as the password**. Git then saves it in `~/.git-credentials`.

⚠️ The token is saved in plain text, so only use this on servers you control. Leave out the `credential.helper` line if you prefer to paste the token each time.

From now on, use the **HTTPS URL** (`https://github.com/...`).

---

## Part D – Prepare the project folder

### D1. Unpack the project

Copy `github-repo-observability-v1.2.0.tgz` to the server (for example to `/root/`), then:

```bash
mkdir -p /root/observability-stack
cd /root/observability-stack
tar -xzf /root/github-repo-observability-v1.2.0.tgz
```

### D2. Add your private settings

```bash
cp my-alert-values.example.yaml my-alert-values.yaml
vi my-alert-values.yaml        # Grafana password, email/Slack
```

If you already have a filled-in `my-alert-values.yaml` (for example in the old `/root/observability` folder), copy it instead:

```bash
cp /root/observability/my-alert-values.yaml /root/observability-stack/
```

### D3. Check the layout

```bash
ls -la
```

You should see:

```
.gitattributes                  <- keeps consistent LF line endings for scripts/configs
.gitignore                      <- keeps secrets and archives out of git
README.md
docs/                           <- this guide
my-alert-values.example.yaml    <- template (goes to GitHub)
my-alert-values.yaml            <- YOUR settings (stays on the server, never pushed)
observability-stack/            <- the Helm chart
scripts/                        <- install.sh, prepare-node.sh, uninstall.sh, lib.sh
```

Check that the scripts are executable:

```bash
ls -l scripts/*.sh             # each line should start with -rwxr-xr-x
chmod +x scripts/*.sh          # fix it if not
```

---

## Part E – First push

Run these steps inside `/root/observability-stack`.

### E1. Start a Git repository

```bash
cd /root/observability-stack
git init
```

### E2. Stage all files

```bash
git add .
```

### E3. ⚠️ Safety check: make sure no secrets are included

```bash
git status
git check-ignore -v my-alert-values.yaml
```

- In the `git status` list you **must NOT** see `my-alert-values.yaml` or any `.tgz` file.
- `git check-ignore` must print `.gitignore:2:my-alert-values.yaml	my-alert-values.yaml`, which means the file is ignored.

If `my-alert-values.yaml` **does** appear in `git status`, stop and run:

```bash
git rm --cached my-alert-values.yaml
ls -la .gitignore              # make sure this file exists
git status                     # check again
```

### E4. Confirm the scripts keep their executable flag

```bash
git ls-files -s scripts        # every line should start with 100755
```

If a line shows `100644`, run `chmod +x scripts/*.sh`, then `git add scripts`.

### E5. Commit

```bash
git commit -m "Observability stack v1.2.0: Prometheus, Alertmanager, Loki, Fluent Bit, Grafana with local storage; RHEL + Ubuntu support"
```

### E6. Connect to GitHub and push

Use the URL that matches your authentication method from Part C.

```bash
git branch -M main

# SSH (Option 1)
git remote add origin git@github.com:<your-user>/observability-stack.git

# or HTTPS (Option 2)
# git remote add origin https://github.com/<your-user>/observability-stack.git

git push -u origin main
```

With HTTPS, enter your GitHub username and paste the **token** when asked for the password. A successful push ends with:

```
To github.com:<your-user>/observability-stack.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

---

## Part F – Check the result on GitHub

Open `https://github.com/<your-user>/observability-stack` and check:

- [ ] The **README** is displayed on the front page, including the architecture diagram.
- [ ] Folders `observability-stack/`, `scripts/` and `docs/` are there.
- [ ] `my-alert-values.example.yaml` is there.
- [ ] ❌ `my-alert-values.yaml` is **NOT** there.
- [ ] ❌ No `.tgz` files are there.

If `my-alert-values.yaml` is visible on GitHub, go to [Part K](#part-k--keeping-secrets-out-of-github) immediately.

---

## Part G – Tag a release (optional)

A tag marks a version so you can always get back to it.

```bash
git tag -a v1.2.0 -m "v1.2.0 - RHEL + Ubuntu support, alerting, pod dashboards"
git push origin v1.2.0
```

Then on GitHub, go to **Releases → Draft a new release**, choose tag `v1.2.0` and click **Publish release**.

To attach a packaged Helm chart to the release:

```bash
helm package ./observability-stack      # creates observability-stack-1.2.0.tgz
```

Upload that file on the release page.

To go back to a tagged version later:

```bash
git checkout v1.2.0
sudo ./scripts/install.sh
git checkout main                       # return to the latest version
```

---

## Part H – Deploy from GitHub to a server

Use this for a **new** server, or to rebuild an existing one. It works the same on RHEL and Ubuntu.

### H1. Install Git

See [A1](#a1-install-git).

### H2. Set up read access (private repository only)

- **SSH deploy key (read-only, one per server):**

  ```bash
  ssh-keygen -t ed25519 -C "deploy@$(hostname)" -f ~/.ssh/id_ed25519 -N ""
  cat ~/.ssh/id_ed25519.pub
  ```

  On GitHub, open the repository → **Settings → Deploy keys → Add deploy key**, paste the key and leave "Allow write access" **unticked**. Then test with `ssh -T git@github.com`.

- **Or a read-only token:** create one as in [Part C, Option 2](#option-2-personal-access-token-https), but with **Contents: Read-only**.

Public repositories need no access setup.

### H3. Clone

```bash
cd /root
git clone git@github.com:<your-user>/observability-stack.git              # SSH
# git clone https://github.com/<your-user>/observability-stack.git        # HTTPS
cd observability-stack
```

### H4. Create your private settings

`my-alert-values.yaml` is not on GitHub (on purpose), so create it on each server:

```bash
cp my-alert-values.example.yaml my-alert-values.yaml
vi my-alert-values.yaml
```

Or copy it from another server:

```bash
scp root@<other-server>:/root/observability-stack/my-alert-values.yaml .
```

### H5. Install

```bash
sudo ./scripts/install.sh --dry-run    # preview, changes nothing
sudo ./scripts/install.sh              # install / upgrade
```

`install.sh` detects RHEL or Ubuntu, SELinux, firewalld or ufw, and single or multi-node, and configures everything to match.

### Moving from the old `/root/observability` folder

Nothing breaks. The release is still called `obs` and the data is still in `/data/observability`, so `install.sh` simply upgrades the running stack. After everything works from `/root/observability-stack`, you can delete the old folder:

```bash
rm -rf /root/observability
```

Copy `my-alert-values.yaml` out of it first if it has your passwords.

---

## Part I – Everyday workflow: change, push, update

### I1. Make a change

For example, raise a threshold in `observability-stack/values.yaml`, add a dashboard JSON, or edit an alert rule:

```bash
cd /root/observability-stack
vi observability-stack/values.yaml
```

### I2. Apply and test it on this server

```bash
sudo ./scripts/install.sh
kubectl -n monitoring get pods
```

### I3. Commit and push

```bash
git status                      # see which files changed
git diff                        # see exactly what changed
git add .
git commit -m "Raise pod memory alert threshold to 4Gi"
git push
```

### I4. Update the other servers

```bash
cd /root/observability-stack
git pull
sudo ./scripts/install.sh
```

Only pods whose configuration changed are restarted.

### Good commit messages

| Good | Bad |
|---|---|
| `Add Slack receiver for alerts` | `update` |
| `Increase Loki retention to 14 days` | `changes` |
| `Fix Fluent Bit parser for Docker runtime` | `fix` |

### Adding a new dashboard

```bash
cp /path/to/my-dashboard.json observability-stack/dashboards/
git add observability-stack/dashboards/my-dashboard.json
git commit -m "Add my-app dashboard"
git push
sudo ./scripts/install.sh       # load it into Grafana
```

### Adding a new script

```bash
chmod +x scripts/new-script.sh
git add scripts/new-script.sh
git commit -m "Add new-script.sh"
git push
```

### Useful commands

```bash
git log --oneline -10           # last 10 commits
git show HEAD                   # what the last commit changed
git diff HEAD~1                 # compare with the previous commit
git checkout -- <file>          # discard your uncommitted edits to a file
git revert HEAD                 # undo the last commit (creates a new "undo" commit)
```

---

## Part J – If the GitHub repo already has files

This happens if you already pushed an earlier version, or ticked "Add a README" when creating the repo. `git push` is then rejected with:

```
! [rejected]        main -> main (fetch first)
```

**Option 1: Keep GitHub's history and add the new version on top (recommended)**

```bash
cd /root
git clone git@github.com:<your-user>/observability-stack.git observability-git
cd observability-git

git rm -r -q .                                            # remove old tracked files (history is kept)
tar -xzf /root/github-repo-observability-v1.2.0.tgz       # copy the new version in
cp /root/observability-stack/my-alert-values.yaml . 2>/dev/null || cp my-alert-values.example.yaml my-alert-values.yaml

git add .
git status                                                # my-alert-values.yaml must NOT be listed
git commit -m "Update to v1.2.0: RHEL + Ubuntu support"
git push
```

Then use `/root/observability-git` as your working folder. You can rename it with `mv /root/observability-git /root/observability-stack` after removing the old folder.

**Option 2: Replace everything on GitHub with your local version**

This deletes the history that is on GitHub. Only use it if nothing on GitHub needs to be kept.

```bash
git push -u origin main --force
```

---

## Part K – Keeping secrets out of GitHub

### What must never be pushed

| File / data | Why |
|---|---|
| `my-alert-values.yaml` | Grafana admin password, SMTP password, Slack webhook |
| Any file containing real passwords, tokens or webhook URLs | Anyone with read access could use them |
| `~/.kube/config`, `/etc/kubernetes/admin.conf`, `/etc/rancher/k3s/k3s.yaml` | Full admin access to the Kubernetes cluster |
| `~/.ssh/id_ed25519`, `~/.git-credentials` | Your GitHub access |

`.gitignore` already blocks `my-alert-values.yaml`, `*-secret*.yaml` and `*.secret.yaml`. Keep secrets in files with those names.

### Before every push

```bash
git status
git diff --cached --name-only
```

Check that neither list includes a secrets file.

### If a secret was pushed by mistake

Deleting the file in a new commit is **not enough**: the old commit still contains it, and anyone can read it in the history.

1. **Change the leaked passwords right away.** This is the most important step.
   - Gmail: delete the App Password at Google Account → Security → App passwords, then create a new one.
   - Slack: regenerate the webhook URL.
   - Grafana: change the password in `my-alert-values.yaml`, then run `sudo ./scripts/install.sh`.
2. Remove the file from Git going forward:

   ```bash
   git rm --cached my-alert-values.yaml
   git commit -m "Stop tracking private values file"
   git push
   ```

3. Optionally, remove it from the history as well:

   ```bash
   # RHEL:   sudo dnf install -y git-filter-repo    (EPEL)  or  pip3 install git-filter-repo
   # Ubuntu: sudo apt-get install -y git-filter-repo
   git filter-repo --invert-paths --path my-alert-values.yaml --force
   git remote add origin git@github.com:<your-user>/observability-stack.git
   git push -u origin main --force
   ```

   Anyone who already cloned the repo still has the old copy, which is why step 1 comes first.

---

## Part L – Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `git: command not found` | Git not installed | RHEL: `dnf install -y git`; Ubuntu: `apt-get install -y git` |
| `fatal: not a git repository` | Running the command in the wrong folder | `cd /root/observability-stack` |
| `Author identity unknown` | Name/email not set | Run step A2 |
| `Permission denied (publickey)` | SSH key not added to GitHub, or wrong key | `ssh -T git@github.com`; check the key under GitHub Settings → SSH keys; `cat ~/.ssh/id_ed25519.pub` |
| `Host key verification failed` | First connection to GitHub | Run `ssh -T git@github.com` once and type `yes` |
| `Support for password authentication was removed` | Used your account password over HTTPS | Paste a **token** instead (Part C, Option 2), or switch to SSH |
| `Authentication failed` after changing the token | Old token saved | `rm ~/.git-credentials`, then push again and paste the new token |
| `ERROR: The key you are authenticating with has been marked as read only` | Pushing with a read-only deploy key | Use a personal SSH key (Part C) on the machine you push from, or tick "Allow write access" on the deploy key |
| `remote origin already exists` | `git remote add` was run twice | `git remote set-url origin <url>` |
| `! [rejected] main -> main (fetch first)` | GitHub already has commits | `git pull --rebase`, then `git push`; or see [Part J](#part-j--if-the-github-repo-already-has-files) |
| `src refspec main does not match any` | No commit made yet | Run `git add .` and `git commit` (E2–E5) first |
| `./scripts/install.sh: Permission denied` | Executable flag lost | `chmod +x scripts/*.sh`, then `git add scripts && git commit -m "Make scripts executable" && git push` |
| `/bin/bash^M: bad interpreter` | A script was edited with CRLF line endings | `sed -i 's/\r$//' scripts/*.sh`, then commit. `.gitattributes` prevents this in the future. |
| `my-alert-values.yaml` shows in `git status` | `.gitignore` missing, or file was tracked before | Make sure `.gitignore` exists, then `git rm --cached my-alert-values.yaml` |
| `git pull`: `Your local changes would be overwritten` | Files edited on this server but not committed | Keep them: `git stash`, `git pull`, `git stash pop`. Discard them: `git checkout -- .` then `git pull` |
| `git pull`: merge conflict | The same line was changed on two servers | Edit the file and remove the `<<<<<<< ======= >>>>>>>` markers, then `git add <file>`, `git commit`, `git push` |
| Proxy / no internet from server (`Could not resolve host: github.com`) | Corporate proxy | `git config --global http.proxy http://proxy:port` (HTTPS), or allow outbound port 22 for SSH |

---

## Quick reference

**Push for the first time (from the server)**

```bash
sudo dnf install -y git        # RHEL      (Ubuntu: sudo apt-get install -y git)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" && cat ~/.ssh/id_ed25519.pub
# -> add the key on GitHub: Settings -> SSH and GPG keys; then: ssh -T git@github.com

mkdir -p /root/observability-stack && cd /root/observability-stack
tar -xzf /root/github-repo-observability-v1.2.0.tgz
cp my-alert-values.example.yaml my-alert-values.yaml && vi my-alert-values.yaml

git init
git add .
git status                     # my-alert-values.yaml must NOT be listed
git commit -m "Observability stack v1.2.0"
git branch -M main
git remote add origin git@github.com:<your-user>/observability-stack.git
git push -u origin main
```

**After a change**

```bash
cd /root/observability-stack
sudo ./scripts/install.sh      # test
git add .
git commit -m "Describe the change"
git push
```

**Deploy on another server**

```bash
git clone git@github.com:<your-user>/observability-stack.git
cd observability-stack
cp my-alert-values.example.yaml my-alert-values.yaml && vi my-alert-values.yaml
sudo ./scripts/install.sh
```

**Update a server**

```bash
cd /root/observability-stack
git pull
sudo ./scripts/install.sh
```
