# RBAC Troubleshooting

<iframe width="560" height="315" src="https://www.youtube.com/embed/BAExpMFykEw" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Overview

A starting point for diagnosing issues when agent editors cannot see certain settings or tabs, or when RBAC policies do not appear to be working as expected.

---

## Common Issue: Missing Tabs for Agent Editors

**Symptom**: A user with editor access cannot see certain tabs (e.g., Data Sets).

### Steps to Diagnose

1. Go to the agent's **Access** tab.
2. Verify the user has the **Editor** role assigned.
3. If tabs are missing despite editor access:
   - Create a **test user**.
   - Add the test user to the same **Access** tab with the same role.
   - Check which settings are visible to the test user.
   - Compare with what the original user sees.

---

## Verifying Policy Assignments

### Check Roles and Policies

1. Go to **Tenant Settings → Management → Roles**.
   - Note: agent editor roles assigned via the **Access** tab won't appear here — they exist only on the agent's Access tab.
2. Go to the **Policies** tab.
3. Look for the policy associated with the user.

### Verify the Resource Mapping

1. Open the browser **Network** tab (Developer Tools).
2. Refresh the page.
3. Search for the `check` endpoint.
4. The first result shows the **tenant** information.
5. The second result shows which **agent** the policy resource number maps to.
6. Confirm the agent number in the policy matches the correct agent.

---

## Checking User and Group Membership

1. In the **Policies** tab, click on the relevant policy.
2. Check the **Users** section to confirm the user is listed.
3. Check the **Groups** tab to see if the user belongs to a group assigned to the policy.
4. Verify group membership by expanding the group to see all members.

---

## Verifying Platform and Agent Resources

- The **platform number** can be found by looking at default policies in the Policies tab.
- The **agent number** can be verified via the Network tab's `check` endpoint.
- Ensure both the platform and agent resource numbers in the policy match the intended targets.

---

## Key Takeaways

- **Access tab roles** (editor) are separate from **Management tab roles** — check both locations
- Use the **Network tab** to verify which agent a policy resource number maps to
- Always verify **user membership** in the policy's Users or Groups sections
- If the setup looks correct but issues persist, escalate for deeper investigation
