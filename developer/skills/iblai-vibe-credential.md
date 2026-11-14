# /iblai-vibe-credential

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`skills/organizations/iblai-vibe-credential/SKILL.md`](https://github.com/iblai/vibe/blob/main/skills/organizations/iblai-vibe-credential/SKILL.md). This page is generated — edit it in the repository, not here.

RBAC setup that lets an API token **list and unmask integration
credentials** on a platform. Integration secrets are served **masked by
default** — a token can read the sensitive fields only when it is bound to
an RBAC policy granting the credential actions on the platform's
`integration-credentials` resource.

## Grant a token unmask access

1. Open **Organization Settings** (Account Dialog) → **Integration → APIs**.
2. Add a new API key and note the token you get.
3. A super admin must change the **Mode → Token Policies**.
4. In **Organization Settings** → **Management → Roles**, create a new Role
   with a name like `Unmask Sensitive Integration Credentials`.
5. Under **Actions**, click search and add the following actions, then save:
   - `Ibl.Mentor/IntegrationCredentialsSensitive/action`
   - `Ibl.Mentor/IntegrationCredentialsUnmask/action`
   - `Ibl.Mentor/IntegrationCredentials/list`
6. In the **Policies** tab, create a new policy and select the Role you
   created in step 4.
7. Set the policy's **Resources** to
   `/platforms/<platform-pk>/integration-credentials/` (the platform **PK** —
   the integer, not the key), add your token under **Tokens**, and save.
8. A super admin needs to to to the Rbac Policies, find the policy you just created and then under Tokens, add your token and save it

The token can now hit the integration-credential endpoint with the unmask
query param:

```
GET {dmUrl}/api/ai-account/orgs/<org>/integration-credential/?unmask_sensitive_fields=true
```

To restrict the token to specific credentials rather than all of them, add
the credential's **PK** (int) to the policy's **Resources** instead of the
whole collection.

## Related

- [`/iblai-vibe-rbac`](/developer/skills/iblai-vibe-rbac) — default roles, the action-definitions endpoint, and the SDK Roles + Policies components.
- [`/iblai-vibe-account`](/developer/skills/iblai-vibe-account) — the Organization Settings surface where API keys, Roles, and Policies are managed.
