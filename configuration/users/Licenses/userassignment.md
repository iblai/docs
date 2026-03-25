# User Assignment

<iframe width="560" height="315" src="https://www.youtube.com/embed/sU3uwf2fqEY" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Create a Group

- Open **Groups** and click **Add Group**.  
- Enter a name (e.g., IBL V1).  
- Search for users, select the ones you need, and click **Add**.  
- New tenants may list only a few users at first; the list grows as accounts are added.  
- Bulk-select and advanced search (e.g., by company) are planned for future releases.  
- The group now appears in the list; expand its dropdown to view current members.

---

## Create a User-License Pool

- Go to **Data Manager → User Licenses**.  
- Click **Add User License**.  
- Fill out:

  - **Name**: IBL Licensing Demo (or similar)  
  - **Count**: 20 seats (or any number you need)  
  - **Start Date / Expire Date** (optional access window)  
  - **Select the Platform** that the tenant admin belongs to

- Click **Save**.  
- The license pool is now available to that tenant admin inside **Analytics → Licenses**.

---

## Assign the License Pool to a Group

- Log in as the **Tenant Admin** and open **Analytics → Licenses**.  
- Choose the license pool you just created (Licensing Demo).  
- Select **Group assignment**.  
- Pick the group you created earlier (IBL V1) and click **Assign**.  
- All members of IBL V1 now hold active seats.

---

## Verify the Assignment

- In **Analytics → Licenses**, switch to the **Groups** tab to confirm the pool is linked to IBL V1.  
- The **Individuals** tab remains empty because you assigned seats at the group level.

---

## Quick Recap

- **Groups** let you bundle users for simpler license management.  
- **User License Pools** define seat counts and (optionally) start/expire dates.  
- **Tenant admins** assign pools to **Groups** or **Individuals** from **Analytics → Licenses**.  
- You can always add users to a group later; their licenses activate automatically.
