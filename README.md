# Jenkins MCP Server Setup

A step‑by‑step guide to install the **Jenkins MCP Server plugin**, harden Jenkins so MCP can connect (without breaking CSRF), wire it to **Claude Desktop**, and run your first commands.

---

## Table of Contents

1. [Overview & Requirements](#overview--requirements)
2. [Install the Jenkins MCP Server plugin](#install-the-jenkins-mcp-server-plugin)
3. [Step 1 — Create & verify a Jenkins API token](#step-1--create--verify-a-jenkins-api-token)
4. [Step 2 — Allow MCP endpoints through CSRF](#step-2--allow-mcp-endpoints-through-csrf)
5. [Step 3 — Configure Claude Desktop](#step-3--configure-claude-desktop)
6. [Step 4 — First things to try in Claude](#step-4--first-things-to-try-in-claude)
7. [How to build the Basic Authorization header](#how-to-build-the-basic-authorization-header)
8. [Common URLs](#common-urls)
9. [Troubleshooting](#troubleshooting)
10. [Security Notes](#security-notes)

---

## Overview & Requirements

* Jenkins 2.426+ (LTS recommended).
* Admin access to Jenkins.
* Ability to restart Jenkins (Homebrew, systemd, Docker, or running the WAR).
* Claude Desktop installed.

> **Terminology**: MCP = *Model Context Protocol*. The **Jenkins MCP Server** exposes tools over HTTP/SSE at `/mcp-server/*` so clients (Claude Desktop, etc.) can list/trigger jobs, fetch logs, etc.

---

## Install the Jenkins MCP Server plugin

1. **Manage Jenkins → Plugins**.
2. Open the **Available** tab and search for **“MCP Server”**.
3. Check **MCP Server** and click **Install without restart** (or **Download now and install after restart**).
4. After install, verify the plugin added endpoints:

   * `http://<jenkins>/mcp-server/mcp` (HTTP transport)
   * `http://<jenkins>/mcp-server/sse` (Server‑Sent Events transport)

> If your Jenkins runs under a context path like `/jenkins`, the endpoints become `/jenkins/mcp-server/mcp` and `/jenkins/mcp-server/sse`.

---

## Step 1 — Create & verify a Jenkins API token

1. In Jenkins, click your avatar (top‑right) → **Security** → **API Token** → **Generate new token**.

   * Name it `claude-mcp`.
   * **Copy the token once** and store it securely.
2. Verify the token works (replace `<NEW_TOKEN>`):

```bash
curl -s -u '<jenkins_id>:<NEW_TOKEN>' http://localhost:8080/whoAmI/api/json
```

Expected JSON includes:

```json
{"name":"<jenkins_id>","authenticated":true}
```

If you get `401 Unauthorized`, regenerate the token or confirm you are using the correct **Jenkins User ID** shown at `/user/<id>/`.

---

## Step 2 — Allow MCP endpoints through CSRF

The MCP plugin needs to register a client; Jenkins’ CSRF protection blocks that unless we **exclude** the MCP paths.

### macOS (Homebrew service)

```bash
brew services stop jenkins-lts
JENKINS_JAVA_OPTIONS='-Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_PATTERNS=/mcp-server/.*' \
  brew services start jenkins-lts
```

### Linux (systemd service)

```bash
sudo systemctl edit jenkins
```

Add the lines:

```
[Service]
Environment="JAVA_OPTS=-Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_PATTERNS=/mcp-server/.*"
```

Then apply and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

### Verify CSRF exclusion is effective

Build a Basic header (see the section below), then:

```bash
curl -I -H "Authorization: Basic <BASE64_USERNAME_COLON_TOKEN>" \
  http://localhost:8080/mcp-server/mcp
```

You should **not** see the HTML error `403 No valid crumb was included in the request`.

> If Jenkins is behind a proxy/ALB, also consider enabling **proxy compatibility** under **Manage Jenkins → Configure Global Security → CSRF Protection**.

---

## Step 3 — Configure Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` and add the Jenkins MCP server. Prefer SSE transport.

```json
{
  "mcpServers": {
    "jenkins": {
      "command": "/Users/shantanu/.nvm/versions/node/v22.16.0/bin/npx",
      "args": [
        "-y","mcp-remote",
        "http://localhost:8080/mcp-server/sse",
        "--transport","sse-only",
        "--header","Authorization: Basic <BASE64_USERNAME_COLON_TOKEN>"
      ],
      "env": {},
      "autoApprove": []
    }
  }
}
```

Replace `\<BASE64_USERNAME_COLON_TOKEN>` with the value from the next section. Restart Claude Desktop after saving.

---

## Step 4 — First things to try in Claude

Use natural language in Claude:

* “Use the Jenkins MCP server to call **whoAmI**.”
* “List jobs at the root and show each job’s **name** and **fullName**.”
* “List jobs inside **<folder>**.”
* “Get details for **<fullJobName>** to see if it’s parameterized.”
* “Trigger a build of **<fullJobName>** with parameters `{ ... }`.”

---

## How to build the Basic Authorization header

The header is `Authorization: Basic <base64(username:token)>`.

```bash
# 1) Create base64 of "username:token" (no newline)
printf %s 'shantanu:<NEW_TOKEN>' | base64
# 2) Use the one-line output in Claude config as:
#    --header "Authorization: Basic <that_output>"
```

You can also dry‑run with curl:

```bash
curl -I -H "Authorization: Basic <BASE64_USERNAME_COLON_TOKEN>" \
  http://localhost:8080/mcp-server/mcp
```

---

## Common URLs

* Jenkins root jobs (JSON): `http://localhost:8080/api/json?tree=jobs[name,fullName]`
* Inside a folder: `http://localhost:8080/job/<Folder>/api/json?tree=jobs[name,fullName]`
* Who am I: `http://localhost:8080/whoAmI/api/json`
* MCP endpoints:

  * HTTP: `http://localhost:8080/mcp-server/mcp`
  * SSE:  `http://localhost:8080/mcp-server/sse`

If Jenkins is hosted under a context path (e.g., `/jenkins`), prepend it to each path.

---

## Troubleshooting

**401 Unauthorized** (on `/whoAmI` or MCP):

* The **user ID** must match the one shown at `/user/<id>/` (case‑sensitive).
* Token is invalid/expired or typed incorrectly. Generate a new API token and retry.

**403 No valid crumb was included** (HTML page from Jetty):

* CSRF exclusion not set. Ensure Jenkins started with:
  `-Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_PATTERNS=/mcp-server/.*`
* If behind proxy/ALB, enable **proxy compatibility**.

**ENOENT npx** (Claude log shows `spawn .../npx ENOENT`):

* Update the `command` path to your actual Node `npx` (e.g., from NVM):
  `/Users/<you>/.nvm/versions/node/<version>/bin/npx`.

**Context path issues**:

* If Jenkins is at `http://host:8080/jenkins`, use `http://host:8080/jenkins/mcp-server/sse` in Claude args.

**Still failing? Quick checklist**

1. `curl -s -u '<jenkins_id>:<TOKEN>' http://localhost:8080/whoAmI/api/json` → authenticated true.
2. `curl -I -H 'Authorization: Basic <base64>' http://localhost:8080/mcp-server/mcp` → no CSRF HTML.
3. Claude config uses the same base64 and the correct URL & `npx` path.

---

## Security Notes

* Treat the **API token** like a password. Store it in a password manager.
* Prefer a dedicated **service user** (e.g., `mcp-bot`) with the minimal permissions needed (Overall/Read, View/Read, Job/Discover, Job/Build; add others as required).
* Rotate tokens regularly and immediately on suspected exposure.
* Keep CSRF protection **enabled** globally; rely on the **exclusion** for `/mcp-server/*` only.

---

**That’s it!** You can now control Jenkins from Claude via the MCP Server plugin—list jobs, inspect parameters, trigger builds, and fetch logs directly from chat.
