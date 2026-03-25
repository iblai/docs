# License Creation

<iframe width="560" height="315" src="https://www.youtube.com/embed/6XnUAQ9ISAo" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Set Up a Content-Provider Platform

- In **Platforms**, click **Create Platform**.  
- Fill out:

  - **Display Name** – e.g., Licensing Demo  
  - **Platform Key** – e.g., licensing-demo  
  - Add initial **tenant-admin credentials** (username, email, password)

- Click **Save**.  
- A new content-provider tenant and its admin account are now created.

---

## Log In as the Tenant Admin

- Sign out of the super-admin account (if needed).  
- Log in with the **tenant-admin credentials** you just set up.  
- You’ll land on **Skills**. From there, you also have access to **AI Analytics**.

---

## Author Course Content for the Provider

- Go to **Studio → Authoring**.  
- Open **Courses** and click **Add New Course**.  
- In the creation form:

  - **Course Name**: Licensing Demo (or similar)  
  - **Organization**: auto-filled with the provider’s tenant

- Click **Create Course**.  
- Add at least one unit (e.g., a multiple-choice problem).  
- **Publish** the course and set course dates in the past so it’s immediately accessible.

---

## Assemble Courses into a Program

- Still in **Authoring**, choose **Programs → Add New Program**.  
- Enter a **program name**, e.g., Licensing Program.  
- Paste the **Course ID** of the course you just created.  
- Add more Course IDs as needed.  
- Click **Save**.

---

## Create a Program License (Super Admin)

- Log back in as a **Super Admin**.  
- Open **Data Manager → Program Licenses**.  
- Click **Add Program License** and fill out:

  - **Display Name**: e.g., Licensing Demo  
  - **Count**: total seats purchased (e.g., 10)  
  - **Start Date / Expire Date**: set the access window (optional but recommended)

- In the pop-up:

  - Select the **Platform** you created earlier  
  - Select the **Program** you assembled

- Click **Save**.  
- The program license is now active and enabled by default.

---

## Verify in Tenant Analytics

- Log back in as the **Tenant Admin**.  
- Go to **Analytics → Licenses**.  
- The new license pool (e.g., Licensing Demo) appears in the list with seat count and dates.

---

## Assign Licenses to Groups or Individuals

- In **Analytics → Licenses**, select the license pool.  
- Choose **Group** or **Individual** assignment.

  - For a **Group**, pick an existing group (e.g., IBL V1) and click **Assign**.  
  - For an **Individual**, search for and select specific users.

- The assigned seats now show under the **Groups** or **Individuals** tab.

---

## Key Takeaways

- **Platform** = tenant for the content provider  
- **Courses → Program → Program License** is the content chain  
- **Seat Count** controls how many users can access the program  
- **Start/Expire dates** define the license window  
- **Tenant admins** assign seats via **Analytics → Licenses** to groups or users  

You’ve now created a content-provider tenant, authored courses, bundled them into a program, issued a license pool, and assigned seats—all within mentorAI.
