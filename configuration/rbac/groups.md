# Groups

<iframe width="560" height="315" src="https://www.youtube.com/embed/tecYyRpQWjI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Use Groups to assign users a consistent set of permissions (policies) across the platform, and combine them with Teams to scope what data those users can see.

---

## Groups vs. Teams (Key Concepts)

### Groups

- Collections of users that are assigned policies.  
- Policies define what actions users can take (e.g., view analytics, enroll learners, create teams).  
- Best for role-based access (e.g., account executives, enrollment managers).

### Teams

- Collections of learners (e.g., Company A users).  
- Used to scope data visibility so managers only see their own learners’ data.  
- Best for data segmentation (Company A vs. Company B).

---

## Creating and Managing Groups

1. Go to **Tenant Settings → Management → Groups**.  
2. Create a group by providing:
   - Group name  
   - Description  
   - Group members (add or remove at any time as a tenant admin)  
3. Save the group.

Anyone added to the group automatically inherits the policies attached to it.

---

## Assigning Policies to Groups

1. Open **Policies** in Management.  
2. Each policy can have groups assigned to it.  
3. Add a group by searching for it and selecting it.

---

## Example

A group like **SBA Corporate Account Executive** may be assigned policies that include:

- Enrollment Manager  
- Analytics Viewer / Reader  
- List Users  
- List Teams  
- Create Teams  
- Access to all user reports  

This allows group members to:

- View users and teams.  
- Create teams.  
- View and download analytics.  
- Enroll learners by sending invitations.

---

## Using Groups Together with Teams (Scoped Access)

If you want someone to:

- Have a role like **Analytics Viewer**, but  
- Only see data for a subset of learners (e.g., Company A only),

Then you should:

1. Create a **Team**.  
2. Add the learners who belong to that company or cohort.  
3. Create or use a **Group**  
   - Example: *Company A Analytics Viewer*.  
4. Assign the **Team to a Policy**.

The policy links:
- The analytics role  
- The specific team  

5. Add users (or a group) to that policy.

---

## Result

Company A managers:

- Can see only Company A’s analytics.  
- Cannot see Company B’s data.  
- Cannot invite users or manage content unless explicitly allowed.

---

## What Groups Enable

Depending on assigned policies, group members can:

- View lists of users and teams.  
- Access analytics dashboards and reports.  
- Create and manage teams.  
- Enroll users into content via invitations.

What they cannot do is determined entirely by the policies attached to their group.

---

## Result

Groups provide clean, role-based permission management, while Teams ensure data stays properly segmented. Together, they give you fine-grained control over who can do what and which learners’ data they can see.
