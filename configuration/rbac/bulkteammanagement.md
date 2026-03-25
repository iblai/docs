# Bulk Team Management

<iframe width="560" height="315" src="https://www.youtube.com/embed/1tGZBe1kxsw" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Create teams and invite users at the same time using a platform invitation CSV. This lets tenant admins and enrollment managers bulk-enroll users while automatically assigning them to teams with a designated manager.

---

## Who Can Use This

- Tenant Admins  
- Enrollment Managers (users who can invite people to the platform/content)

---

## How It Works (High Level)

- You upload a platform invitation CSV.  
- The CSV specifies:
  - Which team (user group) the user belongs to.  
  - Who the team owner/manager is.  
- Invitations are sent.  
- Users appear as team members after they register.

---

## Step-by-Step

### 1) Prepare the CSV

- Download or use your existing platform invitation CSV.  
- Ensure the following fields are set correctly:
  - **User group:** the team the user should belong to (e.g., Team 2).  
  - **User group owner email:** the email of the person who will manage that team.  
- The user group owner email links the team to its manager.

### 2) Upload the CSV

- Go to **Management → Invite**.  
- Select **Upload CSV**.  
- Review the preview:
  - Validate user details.  
  - Confirm the user group and group owner are correct.  
- Click **Save**.  

This triggers the platform invitations.

### 3) User Registration

- Invited users must register before they appear as team members.

**Before registration:**
- The invite shows as pending.  
- The user does not appear in the team yet.

**After registration:**
- The invite shows as accepted.  
- The user automatically appears in the assigned team.

---

## What Admins and Managers See

### Tenant Admins

- Can view all teams.  
- Can see team members once registration is complete.  
- Have access to analytics for these users.

### Enrollment Managers

- Can create and manage teams via CSV.  
- If granted analytics access, can view the same analytics as tenant admins.  
- Do not access tenant settings or edit course content.

---

## Key Notes

- Users do not see which team they belong to from their own perspective.  
- Teams are populated only after registration is completed.  
- This method scales easily for onboarding large cohorts tied to specific managers.

---

## Result

With Bulk Team Management, you can onboard users, assign them to teams, and establish team managers—all in one CSV upload—streamlining large-scale enrollment and oversight.
