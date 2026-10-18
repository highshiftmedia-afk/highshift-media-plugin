# Publishing the Highshift Media plugin

Sources: https://cursor.com/docs/plugins/building (Plugins reference) and
https://cursor.com/docs/context/mcp (checked 2026-10-18).

## Order of work

1. **Server first.** Implement OAuth on `https://highshiftmedia.com/mcp` following
   [OAUTH_IMPLEMENTATION.md](OAUTH_IMPLEMENTATION.md), and pass its test checklist (§14), including
   Cursor desktop and Cursor web sign-in. Cursor's reviewers will try to connect; a server that only
   accepts static tokens gives them a broken install.
2. **Logo.** `assets/logo.png` is the official 512×512 Highshift Media logo.
3. **Test the plugin locally** (checklist item "Plugin has been tested locally"):
   - Easiest: put the same server entry in `~/.cursor/mcp.json`
     (`{"mcpServers":{"highshift-media":{"url":"https://highshiftmedia.com/mcp"}}}`), restart Cursor,
     sign in when prompted, and run each tool once.
   - Then load this folder as a local plugin (see the plugin template repository linked from the
     Cursor docs for the current local-install method) and confirm the logo, name and MCP server appear.
4. **Public GitHub repo.** Create e.g. `github.com/<your-org>/highshift-media-plugin` (public), push
   this folder to it, then add `"repository": "https://github.com/<your-org>/highshift-media-plugin"`
   to `.cursor-plugin/plugin.json` and push again.
5. **Submit** at https://cursor.com/marketplace/publish with the repository link. The Cursor team
   reviews plugins manually before they are listed.

## Submission checklist (from the Cursor docs)

- [x] Valid `.cursor-plugin/plugin.json` manifest
- [x] `name` is lowercase kebab-case: `highshift-media` (uniqueness is checked by Cursor at submission)
- [x] `description` explains the purpose
- [x] All included components have valid files (only `mcp.json`; no rules/skills/agents/hooks)
- [x] Logo committed and referenced by relative path (`assets/logo.png`)
- [x] `README.md` documents usage and configuration
- [x] No `${VAR}` placeholders in `mcp.json`, so no `variables` schema needed
- [x] All manifest paths relative, no `..`, no absolute paths
- [ ] Plugin tested locally — **owner must do this after OAuth is live**
- [n/a] Multi-plugin `marketplace.json` — single-plugin repo

## Release updates

Bump `"version"` in `.cursor-plugin/plugin.json` for each change and push. Server-side changes
(new tools, scopes) don't require a plugin release unless the URL changes.
