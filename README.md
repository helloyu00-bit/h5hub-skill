# h5hub skill

Publish local HTML files to a temporary public H5 link.

## Install

Ask your Agent:

```text
请从 https://github.com/helloyu00-bit/h5hub-skill 安装 h5hub skill
```

If the Agent asks for a command, clone this repo into its supported skills directory.

## Invite code

Run once after install, or the first time publishing fails with `Unauthorized`:

```bash
node <skills-dir>/h5hub/scripts/publish-html.mjs login YOUR_INVITE_CODE
```

## Use

Ask your Agent:

```text
帮我把 /path/to/page.html 托管一下
```

## Notes

- Do not publish private data, secrets, customer data, internal links, illegal content, phishing, scams, malware, credential collection, or misleading pages.
- Links are temporary. Default TTL is 30 days, and visits extend expiry to at least 15 days from access time.
- Single HTML file limit is 10 MB.
- This is an invited internal trial service.
