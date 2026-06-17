---
name: visual-mockup-delivery
description: "Create and deliver visual mockups during product brainstorming. Covers local-server and VPS/remote fallback patterns, HTML wireframe conventions, and Discord MEDIA attachment delivery."
---

# Visual Mockup Delivery

Deliver visual mockups and wireframes to users during brainstorming — whether they're on the same machine or a remote VPS.

## When to Use

- You've offered the visual companion and the user accepted.
- You need to show wireframes, layout options, or architecture diagrams during Step 2 of the brainstorming process.
- The user wants to see a design before you proceed to architect/UX agent alignment.

## Local Server Path (user is on the same machine)

1. Start the server:
   ```bash
   bash /path/to/skills/brainstorming/scripts/start-server.sh --background
   ```
2. Write your mockup HTML to the `screen_dir` returned by the server (typically `/tmp/brainstorm-<pid>-<ts>/content/index.html`).
3. Share the `url` from the server output (e.g. `http://localhost:63784`).
4. The user opens that URL in their browser.
5. Stop the server when done:
   ```bash
   bash /path/to/skills/brainstorming/scripts/stop-server.sh
   ```

## Remote/VPS Fallback Path (user cannot access localhost)

If the user is on a VPS, SSH session, or cloud shell, they cannot reach `localhost:<port>`.

**Workflow:**
1. If unsure, ask: **"Are you on this machine or a remote server?"**
2. Build your mockup as a **self-contained HTML file** — inline all CSS and JS, no external dependencies.
3. Save it to a temp path (e.g. `/tmp/travel2adrar-mockup.html`).
4. Send it as a Discord file attachment using the MEDIA syntax:
   ```
   Here's the mockup — download and open in any browser:
   MEDIA:/tmp/travel2adrar-mockup.html
   ```
5. The user downloads and opens the HTML locally. No server needed.

## HTML Mockup Conventions

- **Tabbed multi-page view** — one HTML file switchable via JS tabs for each page/section.
- **Interactive by default** — users expect working interactions, not static placeholders. Build functional JavaScript for:
  - Navigation tabs that switch content and highlight the active tab
  - Content filter buttons (category tags, price ranges) that show/hide matching cards
  - Gallery category tabs that filter image grid items
  - Card hover states, clickable "Inquire"/"Book" buttons
  - Do NOT build static wireframes where elements are hardcoded visible/invisible — users will try to click and interact.
- **Colour palette annotation** — use CSS variables for brand colours so the user can request changes.
- **Green status bar** — at the bottom of each page section, show `✅ MVP — <page name>` with any deferral notes.
- **Placeholder images** — use `background: linear-gradient(...)` with emoji or text labels rather than external image URLs.
- **Disabled form elements** — for non-functional mockup forms, set `disabled` attribute so the user can't submit.
- **No external dependencies** — everything inline to work offline after download.

## GitHub Repo Push Path (user wants the mockup in a repo)

When the user wants the mockup pushed to a GitHub repository (e.g. for sharing with the team or as the starting point for the real site):

1. **Check repo exists and is accessible.** Use `curl -s -o /dev/null -w "%{http_code}" https://github.com/<owner>/<repo>` — 404 is expected for private repos.
2. **Never use a compromised credential.** If the user shares a PAT that was previously posted in a public channel, treat it as compromised. Do not use it.
3. **Preferred secure method: SSH deploy key.**
   ```bash
   # Generate a deploy key
   ssh-keygen -t ed25519 -f /root/.ssh/<repo>_deploy -N "" -C "<purpose-description>"

   # Show the public key for the user to add
   cat /root/.ssh/<repo>_deploy.pub
   ```
   Ask the user to add it at `https://github.com/<owner>/<repo>/settings/keys` with **Allow write access** ticked.
4. **Clone and push via the deploy key:**
   ```bash
   GIT_SSH_COMMAND="ssh -i /root/.ssh/<repo>_deploy -o StrictHostKeyChecking=accept-new" \
     git clone git@github.com:<owner>/<repo>.git /tmp/<repo>-work
   cp /tmp/<mockup-file> /tmp/<repo>-work/index.html
   cd /tmp/<repo>-work
   git add -A && git commit -m "Initial mockup: <description>"
   GIT_SSH_COMMAND="ssh -i /root/.ssh/<repo>_deploy -o StrictHostKeyChecking=accept-new" \
     git push origin main
   ```
5. **Include a README.md and .gitignore** in the initial commit so the repo has a baseline.
6. **Tell the user the old PAT can now be safely revoked** at https://github.com/settings/tokens.

**Alternative: `gh auth login`** — If the user prefers no SSH keys, they can run `gh auth login --git-protocol https --web` themselves, then create the repo with `gh repo create <owner>/<repo> --private`. After that, you push via the authenticated `gh` CLI.

## Cloudflare Tunnel Path (user is on a VPS but wants live browser preview)

When the user is on a remote machine and wants to **interact with the mockup in their browser** (rather than downloading a file), use a Cloudflare quick tunnel:

1. Start the local server:
   ```bash
   bash /path/to/skills/brainstorming/scripts/start-server.sh --background
   ```
2. Note the `screen_dir` and `port` from the server output.
3. Write your mockup HTML to `<screen_dir>/index.html`.
4. Start a Cloudflare tunnel:
   ```bash
   cloudflared tunnel --url http://localhost:<PORT> &>/tmp/cloudflared.log &
   ```
5. Wait ~10 seconds, then extract the URL:
   ```bash
   grep "trycloudflare" /tmp/cloudflared.log | tail -1
   ```
6. Verify with `curl -s -o /dev/null -w "%{http_code}" <url>`.
7. Share the full URL (e.g. `https://xxx.trycloudflare.com`).

**Cloudflare tunnel notes:**
- Quick tunnels are temporary and expire after inactivity — the URL changes on each restart.
- Background shell messages like `[1] 698134` are the wrapper shell completing; cloudflared keeps running.
- If the tunnel returns 502, check if the local server is still running and restart both if needed.
- The tunnel process and the server process are independent — diagnose them separately.

### Tunnel Expiry Renewal Procedure

When a user reports "expired" or "still see the old version" and curl returns 502:

1. **Diagnose separately:**
   - `curl -s -o /dev/null -w "%{http_code}" http://localhost:<PORT>` — check local server
   - `curl -s -o /dev/null -w "%{http_code}" <tunnel-url>` — check tunnel

2. **If local server is down:**
   - Restart server: `bash start-server.sh --background`
   - Capture the NEW `screen_dir` and `port`
   - Write mockup HTML **fresh** to the new screen_dir (do NOT copy from an old backup — see Content Directory pitfall)
   - Start a new tunnel pointing at the new port
   - Verify correct version before sharing

3. **If local server is up but tunnel is down:**
   - Kill old tunnel if needed: `kill $(lsof -ti:<PORT>) 2>/dev/null`
   - Start new tunnel: `cloudflared tunnel --url http://localhost:<PORT> &>/tmp/cloudflared<N>.log &`
   - Use an incremented log filename (`cloudflared2.log`, `cloudflared3.log`) each time to avoid confusion
   - Extract new URL and verify

4. **Verify the correct version is being served before sharing the new URL:**
   ```bash
   # Check for version-specific markers
   curl -s <new-url> | grep -c "terracotta"          # Should be >0 for terracotta palette
   curl -s <new-url> | grep -c "mockup-nav"           # Should be 0 (old v1 class)
   curl -s <new-url> | grep -c 'class="nav"'          # Should be >0 (v2 single nav)
   ```
   Check that the title tag says the correct version name (`v2` vs `v1`).

## Pitfalls

- Starting the server when the user is on a VPS wastes time and produces nothing usable. Ask first.
- The MEDIA attachment path must be an absolute path the Discord gateway can read. Use `/tmp/` for general access.
- Self-contained HTML files can be very large (10K+ lines) but that's fine — they're downloaded, not transmitted inline.
- **Verify which design version is current before pushing to a repo.** If the user approved a specific colour palette, layout, or font direction (e.g. terracotta/gold instead of navy/gold), confirm the mockup file reflects that approved version. Pushing the old wireframe after a design decision was made will frustrate the user. Check memory or session history for the latest approved design direction before pushing.
- **Never use a compromised credential to push.** If a PAT was posted in a public channel, it is compromised. Do not use it even if the user says "push first, then I'll delete it." Use an SSH deploy key or `gh auth login` instead.

### Content Directory Lifecycle (critical)

The server stores content at `/tmp/brainstorm-<PID>-<TIMESTAMP>/content/`. **This path changes on every server restart.**

- Always write content **after** starting the server — you won't know the path beforehand.
- If the server crashes or is restarted, the old content directory is destroyed and a new one is created.
- When the server restarts, always write the mockup **fresh** to the new screen_dir. 
- Do NOT copy from an old backup — the backup may be an older version, and you'll end up serving stale content.

**HARD RULE: Never copy from `/tmp/` backups after a server restart.** Always write fresh content from the latest approved version. If you have the mockup content in your active conversation context, write it fresh. If a project folder exists at `/root/ai-delivery-org-code/<project>/`, that is the source of truth — copy from there, not from `/tmp/`.

**Right flow after server restart (verified):**
1. Start server → get new screen_dir + port
2. Write mockup HTML directly to the new screen_dir/index.html (either from context or from `/root/ai-delivery-org-code/<project>/index.html`)
3. Start tunnel pointing at the new port
4. Verify version markers before sharing URL

**Wrong flow (caused the "old version" bug in the Travel2Adrar session):**
1. Server restarted → new content directory created
2. Copied `travel2adrar-mockup.html` (v1 backup from `/tmp/`) to the new content directory instead of writing the updated v2 HTML fresh
3. User saw stale v1 mockup and thought the updates hadn't been applied