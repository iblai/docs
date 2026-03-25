# Administration

<iframe width="560" height="315" src="https://www.youtube.com/embed/p0uIV35F7PI?si=XNgwGDlSsv8FL305" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Core Concepts

![](/images/rbac.png)

- **Groups** – collections of users (e.g., Tenant Admin group)  
- **Roles** – bundles of permissions  
- **Actions** = object-level rights (view, edit, delete)  
- **Data actions** = field-level rights (read, write)  
- **Policies** – bind a role to specific resources within a platform and link users or groups to that bundle  
- **Hierarchy** – a policy applied to a platform cascades to every resource beneath it

---

## Granting Full Tenant-Admin Access

- Locate the **Tenant Admin group**  
- Add yourself (or another user) to that group and **save**  
- **Refresh the app**: every tab (Tools, Prompts, Safety, Data Sets, History, API, etc.) is now visible  
- To revoke full access, **remove the user** from the group and **save**

---

## Creating a Limited-Access Role

- Create a **new role** (start broad, then remove what you don’t need)  
- Limit it to the required model—for example, only **Settings**  
- Make a **policy** that applies this role to the desired resources  
- Add yourself to the policy’s user list and **refresh**

### Result:

- **Settings** stays accessible  
- Tabs like **LLMs, History, Data Sets, and API** disappear  
- Only items classified under **Settings** (certain prompts, Safety) remain

---

## Field-Level Control (Data Actions)

- In the role, add a data-action rule such as:  
  - `description : read`

- **Refresh**: you can view but not edit that single field  
- Change the rule to:  
  - `description : read,write` *(or use `*`)* to allow editing

---

## Adding Access to Specific Resources

Follow the same pattern by updating the role:

- **Tools** – add `tools` with the actions you need; the **Tools** tab appears  
- **API Tokens** – add `api_tokens` with `read` and `list` (then `delete` if required); tokens become viewable and, with delete, removable  
- **Documents/Data Sets** – add `documents` with `read`; add `write` to untrain; add `delete` to remove

- For each added permission, **refresh the browser** to see the change

---

## Key Takeaways

- **Groups, Roles, and Policies** combine for precise, layered control  
- Adding a user to a group instantly grants that group’s policies  
- Roles can be fine-tuned from **section-wide access** down to **individual fields**  
- **Permission changes appear immediately after refresh**, letting you verify results in real time
