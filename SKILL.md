---
name: h5hub
description: Publish any local HTML file to a public temporary H5 link through the h5hub Cloudflare Worker. Use when the user asks to host, publish, preview publicly, share, generate a public link, or use h5hub for an HTML/H5 artifact.
---

# H5Hub

## When To Use

Use this skill when the user has a local HTML file or HTML content and wants a public shareable link.

Common user requests:

- `把 /path/to/page.html 托管成一个链接`
- `帮我发布这个 HTML`
- `这个 H5 调用 skill 托管试试`
- `给这个网页生成一个可以分享的公网链接`
- `用 h5hub skill 发布 /path/to/page.html`

Default public base URL:

```text
https://h5.h5hub.xyz
```

Successful page URLs look like:

```text
https://h5.h5hub.xyz/p/xxxxxx
```

## Required User Notice

Before publishing, briefly tell the user these constraints. Do not ask for confirmation unless the content appears risky or the user has not provided a concrete file.

- Do not publish illegal, infringing, phishing, gambling, pornographic, scam, malware, credential-collection, impersonation, or misleading pages.
- Do not publish company secrets, personal privacy data, customer data, access tokens, internal links, or pages that depend on sensitive cookies.
- Links are temporary. Default page TTL is 30 days, and page views extend expiry to at least 15 days from the access time.
- If the user asks for a fixed takedown time, publish with `--ttl`. Custom TTL pages do not extend on access.
- If the user asks for a visitor password, publish with `--password`. This is lightweight access gating, not a replacement for real auth or a safe place for secrets.
- Single HTML file limit is 10 MB.
- Anonymous publishing is rate-limited by source IP: 3 publishes/hour and 10 publishes/day.
- Access limits can expire a page: more than 30 unique visitors/day, more than 100 page views/day, or more than 200 total page views.
- Availability depends on network access to the hosting domain. Mainland China access may still vary by network.

## Standard Workflow

1. Verify the HTML file exists.
2. Briefly show the required user notice.
3. Ask the user to choose the publish policy unless the user already specified it in the request.
4. Publish with the bundled script. The script automatically reads saved login config from `~/.h5hub/config.json`.
5. Return only the successful public URL plus the chosen expiry/password status.
6. If publish fails with `Unauthorized` or `401`, ask the user for the invite code and run the login command below.
7. If publish fails for other reasons, diagnose using the failure handling section below.

## Publish Policy Prompt

When the user says they want to host/publish/share a page but has not specified access policy, ask one concise question before publishing. Do not expect non-technical users to know CLI flags.

Use this wording in Chinese when the user is Chinese:

```text
请确认这次托管策略：
1. 过期时间：默认 30 天自动续期 / 24 小时固定下架 / 7 天固定下架 / 自定义
2. 访问密码：不设置 / 设置一个访问密码
```

Rules:

- If the user chooses default expiry, do not pass `--ttl`.
- If the user chooses 24 hours, pass `--ttl 24h`.
- If the user chooses 7 days, pass `--ttl 7d`.
- If the user chooses custom expiry, ask for a value such as `30m`, `12h`, `3d`, or `30d`, then pass it as `--ttl VALUE`.
- If the user chooses no password, do not pass `--password`.
- If the user chooses password protection but has not provided a password, ask them to provide the visitor password, then pass it as `--password PASSWORD`.
- If the user already gave a clear expiry and/or password in the original request, do not ask again for those fields.
- Never expose the final command with the password unless the user explicitly asks for the command.

## Invite Code Login

If the Worker requires a publish token, the user should only need to enter the invite code once inside the current Agent conversation. Store it locally with:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" login INVITE_CODE
```

If `SKILL_DIR` is not set:

```bash
node scripts/publish-html.mjs login INVITE_CODE
```

This writes:

```text
~/.h5hub/config.json
```

Future publishes automatically use that saved invite code. The precedence order is:

```text
--token / --url > H5HUB_TOKEN / H5HUB_WORKER_URL > ~/.h5hub/config.json > https://h5.h5hub.xyz
```

For a small trusted group, one shared invite code is acceptable. For wider public use, prefer per-user or per-team tokens so abuse can be traced and revoked without rotating everyone.

Preferred command:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" /absolute/path/to/page.html "Page title"
```

If the user chooses 24 hours fixed takedown:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" --ttl 24h /absolute/path/to/page.html "Page title"
```

If the user chooses visitor password:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" --password VIEW_PASSWORD /absolute/path/to/page.html "Page title"
```

If the user chooses fixed takedown time plus visitor password:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" --ttl 24h --password VIEW_PASSWORD /absolute/path/to/page.html "Page title"
```

If `SKILL_DIR` is not set by the runtime, resolve it to this skill folder and run:

```bash
node scripts/publish-html.mjs /absolute/path/to/page.html "Page title"
```

For stdin:

```bash
cat ./page.html | node "$SKILL_DIR/scripts/publish-html.mjs" - "Page title"
```

Machine-readable output:

```bash
node "$SKILL_DIR/scripts/publish-html.mjs" --json /absolute/path/to/page.html "Page title"
```

## Runtime Options

- `--url WORKER_URL`: Override the Worker URL for one publish.
- `--token TOKEN`: Send a bearer token if the hosting service requires authentication.
- `--ttl VALUE`: Set a fixed takedown time for this page. Values support seconds, `m`, `h`, and `d`, for example `30m`, `12h`, `7d`, `30d`.
- `--password PASSWORD`: Require visitors to enter this password before viewing the page.
- `--json`: Print JSON.
- `login INVITE_CODE`: Save the invite code locally for future publishes.
- `H5HUB_WORKER_URL`: Default Worker URL override.
- `H5HUB_TOKEN`: Bearer token environment override.

Example override:

```bash
H5HUB_WORKER_URL="https://h5.h5hub.xyz" node "$SKILL_DIR/scripts/publish-html.mjs" ./dist/index.html "Campaign H5"
```

## Success Criteria

A successful publish prints:

```text
Publish succeeded
URL: https://h5.h5hub.xyz/p/xxxxxx
Code: xxxxxx
```

Return the `URL:` line to the user.

If verifying with curl, use `GET`, not `HEAD`. The Worker only handles `GET` for pages and `/api/health`; `curl -I` sends `HEAD` and may return `404`.

Correct verification:

```bash
curl -sS https://h5.h5hub.xyz/api/health
curl -sS https://h5.h5hub.xyz/p/xxxxxx | head -c 160
```

Expected health response:

```json
{"ok":true}
```

## Failure Handling

### File Problems

If the file does not exist, ask the user for the correct path.

If the file is larger than 10 MB, do not publish. Ask the user whether to compress, split, or use another hosting method.

### Network Problems

If publishing fails with connection, TLS, DNS, or timeout errors, check the public health endpoint:

```bash
curl -sS --connect-timeout 10 --max-time 15 https://h5.h5hub.xyz/api/health
```

If it does not return `{"ok":true}`, tell the user the hosting service is temporarily unreachable and retry later.

## Packaging Notes

When packaging this skill for another Agent, include:

```text
SKILL.md
scripts/publish-html.mjs
```

The script is self-contained Node.js and uses built-in `fetch`, so it expects Node.js 18+.

For invited users, the receiving Agent can run `node scripts/publish-html.mjs login INVITE_CODE` once. After that, the script reads `~/.h5hub/config.json` automatically.

If you host this skill on GitHub for others to install, publish only the skill folder or package contents:

```text
skills/h5hub/SKILL.md
skills/h5hub/scripts/publish-html.mjs
```

Do not include `.DS_Store`, local config, tokens, or historical handoff files.

## Operational Notes

- Default page TTL: 30 days; page views extend expiry to at least 15 days from the access time.
- Default Worker URL: `https://h5.h5hub.xyz`.
- Public pages are served from `/p/{code}`.
- Publish API endpoint: `POST /api/publish`.
- Health endpoint: `GET /api/health`.
