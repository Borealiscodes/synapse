# Termux ARM64 Edge Audit & Testing Guide (`RFC-006` / `RFC-007`)

> **Author & Pacing Disclaimer:** As a neurodivergent human rights defender facing duress, transnational repression, academic exclusion, and denial of grants to explore this technology, **I owe myself the kindness of returning to this project strictly on my own terms.** Even with the roadmap and dependencies mapped out, this guide is a calm technical anchor—not a deadline or obligation. Pause, step away, and resume only when energy, safety, and capacity allow.

---

## Architecture & Environment Constraints
This guide adapts the `RFC-006` (*Institutional Enclosure Geometry*) and `RFC-007` (*Decoupling Attribution from Inference*) verification workflow for an **ARM64 Android Edge Workstation (`Termux` + `OpenSSH` + `Ed25519` sovereignty key)** operating under tight `/data/data` storage limits.

* **Zero Local Model Weights (`0 MB`):** Because `RFC-007` decouples probabilistic LLM inference from deterministic CPU verification, all manifest hashing, quote checking, and commit-seam rejection tests run purely on CPU without `Ollama` or multi-gigabyte model files.
* **Pure-Python Footprint (`< 80 MB`):** Avoids compiled C-wheels (`pymupdf`, `torch`) and heavy database daemons (`SurrealDB`, `Docker`) that fail or exhaust flash storage in Android's sandbox.
* **Ephemeral Lifecycle (`Clone -> Seal -> Push -> Purge`):** Every session clones minimally, verifies locally, commits/signs via SSH, and immediately wipes working trees.

---

## Step 1: Pre-Flight Storage Check & Cleanup
Check available headroom in `$PREFIX` (`/data/data/com.termux/files/usr`) and clear cached package archives before starting:

```bash
df -h \$PREFIX
pkg clean
```

> **Emergency Freeze Recovery Rule:** If storage ever reaches `99%` and Termux becomes unresponsive, **never uninstall Termux** (which destroys your SSH identity). Instead:
> 1. Open Android **Settings -> Apps -> Termux -> Force Stop**.
> 2. Tap **Storage -> Clear Cache** (do **not** tap *Clear Data*).
> 3. Reopen Termux and delete working folders (`rm -rf ~/synapse-work ~/.cache`) while preserving `~/.ssh/id_ed25519_sovereignty`.

---

## Step 2: Verify Minimal Native Packages
Ensure only core CLI utilities are installed:

```bash
pkg update && pkg install -y python git openssh
```

---

## Step 3: Bounded Shallow Clone
Clone `Borealiscodes/synapse` using `--depth 1` and a `1 MB` blob filter so historical assets never touch flash storage:

```bash
git clone --depth 1 --filter=blob:limit=1m git@github.com:Borealiscodes/synapse.git ~/synapse-work
cd ~/synapse-work
```

---

## Step 4: Create a Zero-Cache Virtual Environment (`< 80 MB`)
Create an isolated Python environment and install only pure-Python audit dependencies with `--no-cache-dir` so wheel archives are never duplicated on disk:

```bash
python -m venv .venv
source .venv/bin/activate
pip install --no-cache-dir pydantic pypdf pytest
```

---

## Step 5: Run Offline Deterministic Gate Tests
Execute `RFC-007` verification tests (raw binary `SHA-256` hashing, deterministic chunk manifesting, verbatim quote verification, and commit-seam rejection) using a temporary cache directory:

```bash
pytest -o cache_dir=/tmp/pytest_cache tests/
```

---

## Step 6: Sign, Push, and Purge (`Clone -> Seal -> Push -> Purge`)
Once verified, cryptographically sign your commit with your `Ed25519` sovereignty key, push to GitHub, and purge the local working tree to restore tablet storage:

```bash
git add .
git commit -S -m "audit(rfc007): update deterministic verification gate"
git push origin main

# Deactivate environment, return home, and reclaim all working space
deactivate
cd ~
rm -rf ~/synapse-work /tmp/pytest_cache ~/.cache/pip
pkg clean
```
