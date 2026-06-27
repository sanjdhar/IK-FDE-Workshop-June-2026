# Hands-On Workshop: Build a Fully Automated Personal Agent with OpenClaw running inside a Docker container

In this lab you'll stand up OpenClaw (a self-hosted personal AI agent) in Docker, connect it to Telegram so you can talk to it from your phone, and then hand it a single prompt that builds and publishes your personal website on GitHub Pages — with no code written by you.

This guide covers both **macOS** and **Windows** (via Git Bash). Wherever a step differs between the two, you'll see separate **Mac/Linux** and **Windows (Git Bash)** instructions.

---

## What You'll Build

1. OpenClaw running locally in Docker, connected to an LLM provider of your choice.
2. A Telegram bot paired to your OpenClaw agent, so you control everything by chatting.
3. A GitHub Personal Access Token (PAT) that lets the agent act on your behalf.
4. A live personal bio page at `https://<your-github-username>.github.io/` — created, committed, and published entirely by the agent.

---

## Prerequisites

Before you start, make sure you have:

- **Docker Desktop** installed and running. Depending on how you setup Docker Desktop it may not be running even if installed. Make sure you start up Docker Desktop and confirm its running. From a command window you can run ```docker ps``` and it should not return any errors.

Docker packages OpenClaw and its dependencies into an isolated container so it doesn't touch your host machine directly — this is the safest way to run it.
- **An API key from an LLM provider.** Any one of the following works: OpenAI, Anthropic, Google Gemini, OpenRouter, or another OpenAI-compatible provider. You'll paste this key in during setup, so have it copied somewhere handy.

> **Expected cost:** Running through this workshop typically costs around **$2–$5** in API usage, depending on which provider and model you choose. **Advanced users** who want to avoid API costs entirely can skip the hosted provider and instead point OpenClaw at a **local model** via **Ollama** or **Docker Model Runner** — both run the model on your own machine at no per-token cost.
- **A GitHub account.** You'll create a Personal Access Token (PAT) later — a GitHub account is all you need for now.
- **Telegram app**, installed on your iOS or Andriod mobile device. You will also be accesing your Telegram account via your browser at [web.telegram.org](https://web.telegram.org).
- **Windows only:** Git Bash installed. Git Bash isn't a standalone download — it's bundled with **Git for Windows**, so download and install Git for Windows from [git-scm.com/install/windows](https://git-scm.com/install/windows), then launch the **Git Bash** app it installs. All commands in this guide assume you're running them inside a Git Bash terminal, not PowerShell or Command Prompt — Git Bash gives Windows users the same `bash`/Unix-style commands that Mac and Linux use natively.

> **Why Git Bash on Windows?** OpenClaw's setup scripts are shell (`.sh`) scripts. Git Bash provides a Unix-style shell on top of Windows so the same commands work without translation. PowerShell and cmd.exe will not run these scripts correctly.

---

## Step 1: Clone or download zip file for the OpenClaw Repository

**Windows (Git Bash) note:** Before cloning, it's worth telling Git not to convert line endings on checkout, since OpenClaw's setup script is a Unix shell script and CRLF line endings can break it:

```bash
git config --global core.autocrlf input
```

**Mac users without Git installed:** macOS doesn't always ship with Git by default (it may prompt you to install Xcode Command Line Tools the first time you run a `git` command). If you'd rather not install Git at all, you can download a zip of the repo instead:

1. Download [openclaw/archive/refs/heads/main.zip](https://github.com/openclaw/openclaw/archive/refs/heads/main.zip).
2. Unzip it — this creates a folder named `openclaw-main`.
3. `cd` into that folder in Terminal in place of `cd openclaw` above.

Open your terminal (Terminal on Mac, Git Bash on Windows) and run:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

---

## Step 2: Configure OpenClaw in Docker

Running OpenClaw inside Docker is the recommended path for this workshop — it's the most secure and isolated option, since the agent and its tools stay sandboxed inside the container rather than running with full access to your host machine.

From inside the cloned `openclaw` folder, run:

```bash
./scripts/docker/setup.sh
```

**Windows (Git Bash) note:** if you get a "permission denied" error, the script's execute bit may not have carried over. Run it explicitly with `bash` instead:

```bash
bash scripts/docker/setup.sh
```
This builds the gateway image locally. If you run into any issues or erroors, you can use a pre-built image instead as a fall-back option: 

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./scripts/docker/setup.sh
```

The setup kicks off an interactive setup wizard inside the Docker container. It will:

1. Build the OpenClaw Docker image.
> This can take 5-10 minutes and will be a good time for a coffee/bio break

2. Walk you through a security disclaimer (type/select **Yes** to continue — this is a personal agent with one trusted operator, which is exactly our setup).
3. Ask for a **Setup mode** — choose **QuickStart (recommended)**. This locks in sensible defaults: gateway port `18789`, loopback bind, token-based auth, Tailscale off.

> **If you have access to a cloud server:** advanced users can skip the Docker container and install OpenClaw directly on a Linux VM instead, for a setup that's reachable from anywhere. This guide focuses on the local Docker path, which is what we recommend for the workshop.

### Step 2a: Choose Your Model/Auth Provider

When prompted for **Model/auth provider**, select whichever LLM provider you have an API key for (OpenAI, Anthropic, Google, OpenRouter, etc.) and paste in your key when asked. Pick a concrete model from that provider's list rather than a generic/placeholder model ID — see the **Troubleshooting** section below for why this matters.

> ⚠️ **Billing Warning** As mentioned ealrier the steps in this workshop should cost no more than $5 in API charges. Make sure you follow the clean up steps at the end of this guide. If your API key is active, the agent can call the model repeatedly. Make sure to **delete or revoke your LLM API key once the workshop is over** to avoid any unexpected or excessive charges.

### Step 2b: Skip Optional Extras

The wizard will also ask about web search, skill dependencies, hooks, and other integrations like Slack, etc. For this workshop, it's fine to select **Skip for now** on all of these — they aren't required to complete the website automation exercise.

---

## Step 3: Locate Your Gateway Token

OpenClaw's setup writes your gateway authentication token to disk. You'll want this token to run health checks, log into the local dashboard, and complete the remaining steps.

### Primary method: the `.env` file

The Docker setup script caches the token in a `.env` file at the root of your cloned `openclaw` repo. This is the easiest place to look — same location on both Mac and Windows (Git Bash), since it's just relative to the repo you cloned:

```
openclaw/.env
```

Open it (in a text editor, or with `cat .env` from inside the `openclaw` folder) and look for a line like:

```dotenv
OPENCLAW_GATEWAY_TOKEN=<your-token-here>
```

Copy the value after the `OPENCLAW_GATEWAY_TOKEN=` — that's your gateway token. Keep this copied in your clipboard or in Notepad/TextEdit app. We will feed this token to your agent via the web UI setup later. Treat this token like a password.

### Backup method: `openclaw.json`

If the `.env` file is missing or doesn't have the token, fall back to the main config file:

**Mac/Linux:** the file lives at:

```
~/.openclaw/openclaw.json
```

**Windows (Git Bash):** the equivalent file lives under your Windows user profile. In Git Bash, `~` maps to your Windows profile folder, so the same path works:

```
~/.openclaw/openclaw.json
```

If you need the Windows-native path (e.g., to open it in Notepad or VS Code outside of Git Bash), it's:

```
C:\Users\<your-username>\.openclaw\openclaw.json
```

Open the file and look for the gateway token field (commonly nested under a `gateway` section, e.g. `gateway.token`).

Either way, treat this token like a password — don't share it or commit it to a public repo.

---

## Step 4: Open the Web UI and Activate the Gateway

Open a browser and go to:

```
http://127.0.0.1:18789/
```

The first time you load this, you may see a message like:

```
Gateway: not detected (connect ECONNREFUSED 127.0.0.1:18789)
```

This is expected — the gateway is up but waiting to be authenticated from the browser side. Take the token you copied in **Step 3** and paste it into the token field on this web UI page. Once submitted, the page should refresh and the gateway should report as detected/running.

This applies identically on Mac and Windows — it's just a browser page, so there are no OS-specific differences here.

---

## Step 5: Verify the Gateway Is Running

From inside your cloned `openclaw` repo folder, you can tail the gateway logs or run a health check. Both Mac and Windows (Git Bash) use the identical command — Docker Compose handles the path resolution for you as long as you run it from the repo root:

```bash
docker compose -f ./docker-compose.yml logs -f openclaw-gateway
```

```bash
docker compose -f ./docker-compose.yml exec openclaw-gateway sh -lc 'node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"'
```

> **Path tip:** Always `cd` into the cloned `openclaw` folder first and reference `./docker-compose.yml` with a relative path, on both Mac and Windows. Avoid pasting a full absolute path copied from someone else's machine (e.g. `/Users/sanjay/...`) — your folder location will differ, and on Windows, absolute paths copied from a Mac can mix forward slashes with your machine's actual layout. Relative paths sidestep this entirely.

A healthy gateway should respond without an `ECONNREFUSED` error. If it's not running yet, start it with:

```bash
docker compose -f ./docker-compose.yml up -d
```

---

## Step 6: Set Up Telegram as Your Communication Channel

Telegram is how you'll actually "talk" to your OpenClaw agent. There are two ways to wire this up: the **dashboard flow** (recommended — this is what the [walkthrough video](https://www.youtube.com/watch?v=R0aNDeKrhHE) demonstrates, all done through the browser) and a **CLI flow** using Docker Compose commands (useful if the dashboard isn't cooperating, or you prefer the terminal). Both end in the same place.

### Option A: Dashboard Flow (recommended)

**1. Initiate bot creation.** In the OpenClaw chat (the web dashboard at `http://127.0.0.1:18789/` from Step 4), type: `Let's set up Telegram`. Follow the link OpenClaw gives you, or search for **@BotFather** directly in Telegram. Click **Start**, then send `/newbot`.

**2. Configure your bot.**

- Choose a **name** for your bot (just the display name).
- Choose a **username** that ends in `bot` — for example: `ik_demo_<your-initials/avatar-name>_bot`. 
> NOTE: The username needs to be unique across the Telegram system
- BotFather replies with a **Bot API token** (looks like `123456:ABC-DEF...`). Copy it and treat it like a password.

**3. Link to OpenClaw.** Paste the Bot API token into the OpenClaw Gateway dashboard (the same web UI from Step 4). Wait for confirmation that Telegram is connected and that the DM policy has been set to **pairing mode**.

**4. Pair your account.**

- Click the link in Telegram to message your new bot (or search for it by username and hit **Start**).
- Your bot replies with a message containing your **Telegram user ID** and a **pairing code**.
- Copy that entire message and paste it back into the OpenClaw dashboard, then send it.
- Wait for the confirmation message stating your ID has been successfully paired.

**Security note:** Pairing mode is what makes this safe. Once your user ID is added to the allow list, the bot will ignore messages from anyone else — protecting you from unauthorized access even if someone else finds your bot's username.

### Option B: CLI Flow (Docker Compose)

If you'd rather do this from the terminal, the same steps map to Docker Compose commands run from inside your cloned `openclaw` repo folder (identical on Mac and Windows/Git Bash):

Register the bot token:

```bash
docker compose -f ./docker-compose.yml run --rm openclaw-cli channels add --channel telegram --token <your-bot-token>
```

Switch the DM policy to allowlist and restrict it to your own Telegram user ID — note these also run **through Docker Compose**, not as a bare `openclaw` command, since the CLI lives inside the container:

```bash
docker compose -f ./docker-compose.yml run --rm openclaw-cli config set channels.telegram.dmPolicy "allowlist"
docker compose -f ./docker-compose.yml run --rm openclaw-cli config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'
```

You'll still need to message your bot in Telegram to receive your pairing code, and approve it with:

```bash
docker compose -f ./docker-compose.yml run --rm openclaw-cli pairing approve telegram <code>
```

**Security note:** Same as above — pairing mode plus an allow list means the bot will only respond to you. Don't skip this if you're running the bot on a network where others could discover it.

---

## Step 7: Set Up Your GitHub Personal Access Token (PAT)

The agent needs a way to authenticate to GitHub on your behalf. Rather than giving it your password, you'll create a scoped, short-lived token.

Use a **classic** PAT for this, not a fine-grained one. Fine-grained tokens have to be scoped to a specific, already-existing repository — but in this workshop the agent itself creates the repo, so there's nothing to scope a fine-grained token to yet. A classic token avoids that chicken-and-egg problem.

1. Log into GitHub and go to **Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token → Generate new token (classic)**.
3. Give it a note (e.g. "OpenClaw workshop") so you remember what it's for.
4. Set the **expiration to 7 days** — short-lived tokens limit the blast radius if one ever leaks, which is good practice for a workshop credential.
5. Under **Select scopes**, check only the top-level **`repo`** box ("Full control of private repositories"). This single scope covers everything the agent needs — creating the repository, pushing commits, and enabling GitHub Pages — and automatically includes its sub-scopes (`repo:status`, `repo_deployment`, `public_repo`, `repo:invite`, `security_events`). You don't need to check those individually or grant anything outside of `repo`.
6. Click **Generate token** and copy it immediately — GitHub only shows it once.

Keep this token somewhere you can paste it from when the agent asks for it. Do not commit it to any file in your repo.

---

## Step 8: Give the Agent Its Mission

Now for the fun part. Message your OpenClaw bot in Telegram with a prompt like this:

> I want to build a personal blog page using the free GitHub web hosting that's available by pointing any web browser to `https://<your-github-username>.github.io/`. Make sure you enable GitHub Pages when you connect to my account. My GitHub URL is `https://github.com/<your-github-username>`. Tell me what you need to be able to create a sample personal bio page for me. I want this to be fully automated so you are doing everything. I don't want to do anything other than perhaps setting up a GitHub Personal Access Token (PAT) so you can authenticate to my GitHub account. Tell me what you need to make this happen.

The agent should respond by asking for your PAT (from Step 7) and any preferences for the bio page content. From there, it should handle repository creation, GitHub Pages configuration, content generation, Git version control, and any future updates — all without you writing or editing code yourself.

---

## Step 9: Have Your Agent Do Other Things

Once your site is live, keep going — this is where it gets fun. Try messaging your OpenClaw bot in Telegram with requests like:

- **Send a photo and have it added to the site.** Take a selfie in Telegram and send it to your bot, then ask the agent to add it to your bio page (e.g., as a profile photo or in a new "About Me" section).
- **Add a navigation menu.** Ask the agent to add a menu bar to your site with links like Home, Contact, and Blog — even if those pages don't exist yet, this is a good test of how far it will go to fill in the gaps.
- **Have it write and publish a blog post.** Pick a topic you're interested in and ask the agent to write a short blog post about it and publish it to your site, linked from the menu bar above.

Each of these should flow the same way as Step 8: you describe what you want in plain language, and the agent handles the repository changes, commits, and any GitHub Pages updates on its own.

---

## Cleanup

When you're done with the workshop, shut things down and revoke anything that could otherwise keep costing you money or stay exposed:

1. **Stop the OpenClaw containers.** From inside your cloned `openclaw` repo folder:

   ```bash
   docker compose -f ./docker-compose.yml down
   ```

2. **Delete your LLM provider API key.** Go back to whichever provider's dashboard you used in Step 2 (OpenAI, Anthropic, Google, OpenRouter, etc.) and delete/revoke the key you created for this workshop.
3. **Delete your GitHub PAT.** Go to **Settings → Developer settings → Personal access tokens → Tokens (classic)** and revoke the token from Step 7, rather than waiting for it to expire on its own.
4. **Optional:** revoke/delete your Telegram bot via @BotFather (`/deletebot`) if you don't plan to keep using it.

---

## Troubleshooting / Common Pitfalls

These are real issues encountered while testing this workshop — worth reading before you hit them yourself.

**Absolute paths in `docker compose -f` commands break across machines.** Commands copied from someone else's terminal output (including this guide's source material) often contain hardcoded absolute paths like `/Users/sanjay/Development/.../docker-compose.yml`. These won't exist on your machine. Always `cd` into your own cloned `openclaw` folder and use `./docker-compose.yml`.

**Windows path separators.** Windows natively uses backslashes (`C:\Users\you\...`) while Git Bash, Docker, and OpenClaw all expect forward slashes (`/c/Users/you/...` or `~/...`). Stick to Git Bash's `~` shorthand and relative paths wherever possible, and avoid manually typing backslash paths into commands.

**Script won't execute on Windows.** If `./scripts/docker/setup.sh` errors with a permissions issue in Git Bash, run `bash scripts/docker/setup.sh` instead.

**Gateway shows `ECONNREFUSED`.** This just means the gateway container isn't up yet. Run `docker compose -f ./docker-compose.yml up -d` from the repo root and re-check.

**Telegram bot ignoring your messages after pairing.** Double-check that `channels.telegram.allowFrom` contains your correct numeric Telegram user ID (not your username) and that `dmPolicy` is set to `allowlist`.

---

## Quick Reference: Key Commands

Run all of these from inside your cloned `openclaw` repo folder, on both Mac and Windows (Git Bash):

```bash
# Tail gateway logs
docker compose -f ./docker-compose.yml logs -f openclaw-gateway

# Health check
docker compose -f ./docker-compose.yml exec openclaw-gateway sh -lc 'node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"'

# Start the gateway
docker compose -f ./docker-compose.yml up -d

# Add a Telegram channel
docker compose -f ./docker-compose.yml run --rm openclaw-cli channels add --channel telegram --token <your-bot-token>

# Lock Telegram down to just you
docker compose -f ./docker-compose.yml run --rm openclaw-cli config set channels.telegram.dmPolicy "allowlist"
docker compose -f ./docker-compose.yml run --rm openclaw-cli config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'

# Approve a pairing code
docker compose -f ./docker-compose.yml run --rm openclaw-cli pairing approve telegram <code>
```

---

## Further Resources

- [Install OpenClaw on Windows via Docker: The 2026 Setup That Actually Works](https://www.youtube.com/watch?v=k4fQiEBRWYY)
- [How to Connect OpenClaw to Telegram](https://www.youtube.com/watch?v=R0aNDeKrhHE)
- [OpenClaw Security Docs](https://docs.openclaw.ai/gateway/security)
- [OpenClaw Pairing/Channels Docs](https://docs.openclaw.ai/channels)

Run `openclaw security audit --deep` periodically once your agent is up and running — it's good hygiene for any tool-enabled agent, especially one with GitHub write access.
