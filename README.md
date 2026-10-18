# Highshift Media (HSM) plugin for Cursor

Connect your **Highshift Media** account to your AI agent. Highshift Media provides hands-free AI
services for local businesses — AI receptionists, missed-call rescue, lead agents, CRM automation,
websites and apps. This plugin adds the official HSM MCP server so your agent can work with your
HSM workspace directly from chat.

- Website: https://highshiftmedia.com
- MCP endpoint: `https://highshiftmedia.com/mcp` (Streamable HTTP)
- Support: info@highshiftmedia.com

## What it does

| Tool | What it does | Access |
| --- | --- | --- |
| `whoami` | Shows the signed-in HSM user and company | read |
| `billing_status` | Subscription, plan and free-trial status | read |
| `services_list` | Lists HSM services available to your account | read |
| `crm_list_contacts` | Lists / searches CRM contacts | read |
| `crm_get_contact` | Gets one CRM contact | read |
| `crm_create_contact` | Creates a CRM contact | write |
| `crm_delete_contact` | Deletes a CRM contact | write (destructive) |
| `crm_list_conversations` | Lists CRM conversations | read |
| `email_list_accounts` | Lists email accounts connected to HSM | read |
| `email_list_messages` | Lists messages in a connected inbox | read |
| `email_read_message` | Reads one email message | read |
| `reel_publish` | Publishes a Reel to a connected Facebook Page | write (public) |
| `social_publish_status` | Checks the status of a social publish job | read |

Every call runs with **your** HSM permissions and is audit-logged in your HSM account.

## How to connect

1. Install **Highshift Media** from the Cursor Marketplace (or Customize → Plugins).
2. When the agent first uses an HSM tool, Cursor opens a browser window to
   **highshiftmedia.com**. Sign in with your HSM account.
3. Review the requested permissions on the HSM consent screen and click **Allow**.
4. You're connected. Cursor stores and refreshes the token for you — no API key to copy.

You don't need an HSM token or any configuration: the plugin uses standard MCP OAuth
(OAuth 2.1 + PKCE). To disconnect, remove the plugin in Cursor, or revoke the connection in HSM under
**Settings → Connect an AI agent**.

### Permissions (OAuth scopes)

| Scope | Allows |
| --- | --- |
| `account.read` | `whoami`, `services_list` |
| `billing.read` | `billing_status` |
| `crm.read` | `crm_list_contacts`, `crm_get_contact`, `crm_list_conversations` |
| `crm.write` | `crm_create_contact`, `crm_delete_contact` |
| `email.read` | `email_list_accounts`, `email_list_messages`, `email_read_message` |
| `social.publish` | `reel_publish`, `social_publish_status` |

### Advanced: manual setup with a personal token

Other MCP clients that don't support OAuth can still use a personal token created in HSM under
**Settings → Connect an AI agent**. Never commit the token anywhere:

```json
{
  "mcpServers": {
    "highshift-media": {
      "url": "https://highshiftmedia.com/mcp",
      "headers": { "Authorization": "Bearer ${env:HSM_MCP_TOKEN}" }
    }
  }
}
```

## Example prompts

- "Who am I signed in as on Highshift Media?"
- "List my newest 10 CRM contacts."
- "Add Jane Doe, jane@example.com, +49 151 000000 as a CRM contact."
- "Show the latest emails in my connected inbox."
- "Publish this video as a Reel on my Facebook page with the caption '…'."

## Requirements

- A Highshift Media account (https://highshiftmedia.com).
- For email and Reels tools: an email account / Facebook Page connected inside HSM.

## Privacy & security

- No secrets are stored in this repository.
- Sign-in happens on highshiftmedia.com; Cursor never sees your HSM password.
- Access tokens are short-lived and scoped to `https://highshiftmedia.com/mcp`.
- Destructive or public actions (`crm_delete_contact`, `reel_publish`) are confirmed by Cursor's
  tool-approval prompt before running.

## For maintainers

Server-side OAuth requirements are documented in [docs/OAUTH_IMPLEMENTATION.md](docs/OAUTH_IMPLEMENTATION.md).
Release steps are in [docs/PUBLISHING.md](docs/PUBLISHING.md).

## License

MIT — see [LICENSE](LICENSE).
