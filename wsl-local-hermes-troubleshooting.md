# Running Hermes Agent Locally on WSL2 (Windows) — Real Problems I Hit and How I Fixed Them

Posting this because I saw someone in the community about to give up on a local setup after getting stuck on permissions. I went through the exact same kind of pain building Hermes locally instead of on a VPS — here's everything that tripped me up, and the actual fix for each one. Hopefully it saves someone a few hours.

## My setup, in one paragraph

Windows 11 laptop, running **WSL2 with Ubuntu 24.04** instead of a paid VPS. Inside Ubuntu, I created a **dedicated, non-root user called `hermes`** (no sudo access, on purpose — least privilege) to actually run the agent, separate from my normal Windows/Ubuntu user (I'll call that one the "admin user" below — it's whatever account you use day-to-day, the one with sudo). The agent's files live in `~/workspace` inside the `hermes` account. If your setup differs slightly, the underlying causes below should still apply — it's mostly WSL2 and Linux-permissions behavior, not anything Hermes-specific.

---

## Installation & base system

### PATH doesn't find the `hermes` command after install

**Symptom:** `hermes --version` returns `command not found` right after running the install script.

**Cause:** the installer puts the binary at `~/.hermes/hermes-agent/venv/bin/`, but that path isn't always added to your shell's `PATH` automatically.

**Fix:**
```bash
echo 'export PATH="$HOME/.hermes/hermes-agent/venv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

### WSL suspends itself and every background service dies

**Symptom:** the dashboard and the Telegram bot randomly stop responding, even though the laptop is on.

**Cause — this is actually two separate problems stacked on top of each other:**
1. **Windows** puts the laptop to sleep after a period of inactivity, even when plugged in.
2. Independently of that, **WSL2 shuts down its own lightweight VM** about 15-20 seconds after you close the last open terminal window — a completely separate behavior from Windows sleep.

**Fix (you need both parts):**
```powershell
# In PowerShell, as admin — stops the laptop from sleeping while plugged in
powercfg /change standby-timeout-ac 0
```
Then create a **Windows Task Scheduler** entry, triggered "at log on", running in the background:
```
Program: wsl.exe
Arguments: -d Ubuntu --exec sleep infinity
```
This keeps the WSL VM alive without needing any visible terminal window open.

---

### This is the permissions one — files created from Windows Explorer aren't accessible to the agent user

**Symptom:** the `hermes` user can't read/write files inside its own workspace folder, when those files were created by dragging them in from Windows File Explorer.

**Cause:** a file created from Windows Explorer (or your normal Windows/Ubuntu login) ends up owned by *that* user, not by `hermes`. In Linux, ownership and permissions are separate things from "which folder it's physically in" — being inside `hermes`'s workspace doesn't automatically mean `hermes` owns it or can touch it.

**One-off fix** (grant access without breaking access for the other user):
```bash
# Run as whichever user actually owns the file/folder
chmod o+rx /home/hermes/workspace/some-folder
chmod o+rw /home/hermes/workspace/some-folder/*.md
```

**Permanent fix** (so new folders inherit the right permission automatically going forward):
```bash
# As the hermes user, on a folder hermes already owns
setfacl -d -m other:rwx ~/workspace/some-parent-folder
```

This was, by far, the thing that ate the most time for me — it's not obvious at all if you're new to Linux permissions, and the error messages don't point you toward "this is an ownership problem, not a code problem."

---

## Obsidian (if you're using it as a window into the agent's files)

### `EISDIR` error opening the vault from Windows

**Symptom:** Obsidian (installed on Windows) throws `Error EISDIR: illegal operation on a directory, watch \\wsl.localhost\...` when trying to open the WSL folder as a vault.

**Cause:** known, long-standing Obsidian bug — it can *list* a folder over a Windows network path (`\\wsl.localhost\...`) fine, but it can't reliably *watch* that folder for changes, which is required to open it as a vault.

**Real fix:** install the **Linux version of Obsidian** inside WSL itself (WSLg lets Linux GUI apps show up as normal Windows windows), so it accesses the folder natively instead of over the network:
```bash
# As your admin user (not the dedicated agent user), since apt install needs sudo
deb_link=$(curl -s https://api.github.com/repos/obsidianmd/obsidian-releases/releases | grep "browser_download_url.*amd64.deb" | head -1 | cut -d '"' -f 4)
curl -L -o ~/obsidian.deb "$deb_link"
sudo apt install -y ~/obsidian.deb
obsidian &
```
Then open the vault using the native Linux path (e.g. `/home/hermes/workspace`), not the Windows network path.

---

### Obsidian process runs but no window appears (WSLg)

**Symptom:** `ps aux` shows Obsidian running, but nothing shows up on the Windows desktop.

**Cause:** WSLg's graphics stack didn't start correctly for that session.

**Fix:**
```powershell
wsl --shutdown
```
Wait ~20 seconds, open a fresh terminal, and sanity-check WSLg with something lightweight first (`sudo apt install -y x11-apps && xeyes`) before retrying Obsidian.

---

## Models and cost control

### A specific model isn't available on your ChatGPT-OAuth connection

**Symptom:** `HTTP 400: model is not supported when using Codex with a ChatGPT account`.

**Cause:** some models in a given family aren't available through OAuth login, only through a direct paid API key.

**Fix:** switch to a sibling model that *is* available on that connection method (check via `/model` inside a chat).

---

### Free-tier monthly quota runs out

**Symptom:** the model stops responding; usage page shows 0% remaining, resetting in weeks, not hours.

**Cause:** some free plans cap usage **monthly**, unlike paid tiers that often use short rolling windows (much harder to fully exhaust).

**Fix:** configure a `fallback_providers` entry pointing at a genuinely different, cheap paid model — cents per day for light usage, and not subject to free-tier congestion.

---

### Your fallback model doesn't actually help

**Symptom:** logs show `Fallback skip: chain entry ... resolves to the same backend as the current one`.

**Cause:** primary and fallback were set to the exact same model.

**Fix:** fallback needs to be a genuinely different model/provider:
```yaml
model:
  provider: openrouter
  default: <primary-model>
fallback_providers:
  - provider: openrouter
    model: <a-different-model-from-a-different-company>
```

---

### A model leaks its internal reasoning into the chat

**Symptom:** you suddenly see raw, third-person "thinking out loud" text in the chat instead of a normal reply.

**Cause:** documented bug in some models (I hit it specifically with MiniMax's M2.7 and M3) where internal reasoning isn't reliably hidden from the final output.

**Fix:** switch away from the affected model family.

---

### A free model gets pulled without warning

**Symptom:** `HTTP 404: This model is unavailable for free. The paid version is available now`.

**Cause:** free-model catalogs (I'm on OpenRouter) change often — a free model today can be paid-only tomorrow, no notice.

**Fix:** always check the *current* free-models list before configuring one, rather than trusting info from even a day earlier.

---

### Everything gets slow/rate-limited at once, across unrelated free models

**Cause:** this happened to me on a day multiple major paid AI providers had a simultaneous outage — traffic surged toward free alternatives, saturating shared capacity.

**Fix:** in this specific case, just wait — it's an external load spike, not your config. A cheap *paid* model, even a few cents/month worth, isn't competing for that same saturated free capacity.

---

## Scheduling (cron) and cross-session memory

### A scheduled job silently gets skipped after you change your model config

**Symptom:** `RuntimeError: Skipped to prevent unintended spend: global inference config drifted since this job was created...`

**Cause:** intentional safety behavior — if the model changes after a job was created, the agent would rather stop than risk an unreviewed spend.

**Fix:**
```bash
hermes cron edit <job_id> --provider <current_provider> --model <current_model>
```

---

### Chat sessions don't "remember" what a scheduled run did

**Symptom:** in a normal chat, the agent denies having done something that a scheduled job actually did that same morning.

**Cause:** each scheduled run happens in its own session, separate from your regular chat sessions — they don't share conversation memory automatically (they do share the same files on disk, though).

**Fix:** add a standing rule to whatever "always-loaded" instructions file your setup uses (for me, `AGENTS.md`):
```markdown
Scheduled/cron runs happen in separate sessions from regular chat. If the user references something that already happened and you don't recognize it, don't deny it outright — check the relevant files on disk first.
```

---

### A scheduled job fabricates a result that doesn't exist

**Symptom:** an automated search presented a specific, plausible-looking result (name, numbers, a link) — but on closer investigation, the agent itself admitted "I can't find any evidence this exists... looks like an error on my part."

**Cause:** the model generated something that sounded plausible instead of admitting it found nothing, likely under the implicit pressure of "I should have results to show."

**Fix:** add an explicit rule to the relevant skill:
```markdown
- Never present a result (listing, data point, link) without having verified the source loads real content in that specific search. If you can't verify it, exclude it or flag it explicitly as "unverified" — never present it with full details as if it were reliable.
```

---

### Broken links get presented as valid anyway

**Symptom:** a link returning a 403/404 got included regardless, with a soft "worth double-checking" caveat.

**Cause:** no rule forced exclusion of an unverifiable source — a caveat isn't the same as a hard rule to drop it.

**Fix:** same rule as above — a failed fetch means automatic exclusion, not a footnote.

---

### The agent conflates two different listings from the same source

**Symptom:** two genuinely different postings from the same company got merged, with one's details (e.g. work modality) attributed to the other.

**Cause:** the comparison only ran against the historical "already seen" log, never *within* the current batch of new results itself.

**Fix:**
```markdown
- Before presenting a new batch of results, also compare them against EACH OTHER (not just the historical log), to catch duplicates or mixed-up details between similar entries.
```

---

### "I'm going to do X now..." messages persist even with `tool_progress: off`

**Symptom:** the agent kept narrating each step in chat, despite `display.tool_progress: off` being correctly set in `config.yaml`.

**Cause:** that setting only controls Hermes's own *system-generated* tool-activity messages — it doesn't stop the model itself from choosing to write step-narration text as part of its normal response.

**Fix:** an explicit instruction in `SOUL.md` (not `config.yaml`):
```markdown
## Communication During Multi-Step Tasks
When performing a multi-step task, don't send intermediate messages narrating what you're about to do. Work silently through all necessary steps and respond once with the complete final result, unless you genuinely need to ask a question to proceed.
```

---

### A skill doesn't reliably load during scheduled (cron) runs

**Symptom:** a scheduled job evaluated things without apparent knowledge of rules/data that were in an installed skill, even though the skill was mentioned by name in the job's prompt text.

**Cause:** each cron run happens in a fresh, isolated session — *mentioning* a skill in the prompt text doesn't guarantee it gets fully loaded, since that still depends on the model's own judgment.

**Real fix:** formally attach the skill to the job itself, not just reference it in text:
```bash
hermes cron edit <job_id> --skill <skill-name>
```
Verify with `hermes cron list` — it should show a `Skills:` field on the job, not just a mention inside the prompt text. This guarantees the full skill loads every run, independent of the model's judgment call.

---

### A scheduled job possibly fires twice

**Symptom:** the same scheduled job delivered two separate responses, with different content, the same day — without a manual re-trigger.

**Likely cause:** two gateway processes running at once (each with its own independent scheduler tick), which can happen after an abrupt restart or an unexpectedly closed terminal session.

**Diagnose it in the moment, if it happens again:**
```bash
ps aux | grep hermes-agent | grep -v grep
```
If more than one process with `gateway` in the command shows up, that's the cause:
```bash
hermes gateway restart
```
Honest caveat: by the time I investigated mine, no duplicate process was still around (likely cleaned up by a later WSL restart) — so I couldn't confirm the exact cause with certainty, just the most likely one per the official docs.

---

## Keeping the Windows auto-start task reliable long-term

### An "At log on" trigger stops firing after a few days

**Symptom:** WSL stopped auto-starting on its own after about a week, even with the laptop used daily — the task's "Last Run Result" in Task Scheduler showed a date over a week old.

**Cause:** an **"At log on"** trigger only fires on a genuine fresh Windows login (username + password from scratch) — if your daily routine is just unlocking the screen instead of logging out/in, that trigger never fires again. And if WSL's own lightweight VM dies mid-day for any other reason (see the suspend/sleep entry above), nothing brings it back up until the next real login.

**First attempt (has its own bug, don't use this one):** adding "Repeat task every: 5 minutes" with duration **"Indefinitely"** to the existing trigger. Sounds reasonable, but it's a **real, well-documented Windows Task Scheduler bug** (reported from Windows Server 2019 through Windows 11): pairing a recurring-style trigger with an "Indefinitely" duration makes the task silently stop firing after the first run, with no visible error.

**Fix that actually works, per multiple corroborating sources:** rebuild the trigger with these specific settings instead:
1. Trigger type: **"On a schedule"** → **"Daily"** (not "At log on")
2. Under **Advanced settings**: **"Repeat task every: 5 minutes"**
3. **"for a duration of:"** → **"1 day"** (NOT "Indefinitely" — this is the change that fixes the bug)
4. On the task's **Settings** tab: enable **"Run task as soon as possible after a scheduled start is missed"**

With "Daily" + a 1-day repeat duration, the cycle refreshes itself every 24 hours, sidestepping the "Indefinitely" bug entirely.

---

### The task opens a visible terminal window every time it repeats

**Symptom:** after applying the fix above, a visible Ubuntu terminal window pops up (or gets minimized and reappears) on the desktop every 5 minutes.

**Cause:** confirmed in official Microsoft documentation — the **"Run only when user is logged on"** option (the one that avoids a password prompt) makes any window the task opens **visible on the desktop, by design**. The alternative that runs invisibly ("Run whether user is logged on or not") normally requires a password.

**Fix: use both at once, via a checkbox that solves both needs simultaneously**
1. Select **"Run whether user is logged on or not"**
2. Check the box that appears beneath it: **"Do not store password. The task will only have access to local resources"**

With this combination, the task runs invisibly (as intended) and **doesn't ask for any password** — the "local resources only" restriction doesn't matter here, since `wsl.exe -d Ubuntu --exec sleep infinity` doesn't need any network resource access.

---

## External integrations (I used Composio for Gmail/Drive/Docs)

### A quick "connect" grants way more access than you need

**Symptom:** a simple one-click Gmail connection ended up granting 61 different tools, including permanent bulk-delete with no recovery.

**Cause:** the fast connection flow uses a broad OAuth scope by default, not a restricted read-only one.

**What I tried (partially blocked):** creating a custom, scope-restricted auth config — Google blocked it outright ("This app is blocked"), because that specific restricted scope requires the connecting app to go through its own security-verification process, which the shared/managed connector app hasn't completed for that combination. Solving this properly means creating your own Google Cloud project with your own OAuth credentials — a separate project in itself.

**What I did in the meantime:** explicitly listed the dangerous tool names as forbidden in the agent's core instructions file:
```markdown
## Forbidden Gmail Tools
Never use these under any circumstances: GMAIL_BATCH_DELETE_MESSAGES, GMAIL_DELETE_MESSAGE, GMAIL_DELETE_DRAFT, GMAIL_TRASH_MESSAGE, GMAIL_MOVE_TO_TRASH, GMAIL_CREATE_FILTER, GMAIL_MODIFY_LABELS, GMAIL_BATCH_MODIFY_MESSAGES.
```
Not as strong as an actual scope restriction, but a real, working guardrail while the "proper" fix sits on the to-do list.

---

## Terminal basics that tripped me up (if you're newer to Linux)

### The terminal hangs after `cat ... << EOF`, waiting forever

**Symptom:** pasting a long block of text to create a file, the terminal never returns to the normal prompt after typing `EOF`.

**Cause:** pasting long blocks sometimes introduces an invisible character or line break that stops `EOF` from being on its own clean line, which is required for it to be recognized.

**Fix:** `Ctrl + C` to break out, and use `nano` for long pasted blocks instead — more forgiving, as long as you double-check you're actually typing/pasting into the terminal itself and not into some other input box on screen.

---

If you're hitting a wall on the permissions piece specifically — that was my biggest time sink too. Happy to answer questions in the comments if any of this doesn't quite match what you're seeing.
