# User Migration

<iframe width="560" height="315" src="https://www.youtube.com/embed/vl2793vT9nE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Move an existing user from one tenant to another while controlling their permissions.

---

## Remove the User’s Access to the Current Tenant

- In **Data Manager**, expand **Core** and open **User Platform Links**.  
- Locate the target user (example: AshenBrown15, ID 126).  
- Edit that user’s **platform-link entry**.  
- Clear both **Admin** and **Active** check-boxes.  
  - This prevents the user from using admin tools and stops access to the tenant.  
- Click **Save**.

---

## Add the User to a New Tenant

- Still in **User Platform Links**, choose **Add User Platform Link**.  
- Search for and select the **same user** (ID 126).  
- Pick the **destination platform** (example: editech).  
- Set the desired permissions:

  - **Is Admin** – lets the user reach admin panels  
  - **Is Staff** – grants access to edX courses  
  - **Active** – allows log-in to the platform

- Click **Save**.

---

## Result

The user is now **removed from the original tenant** and **added to the new one**, with permissions exactly as configured.
