# Administration

<iframe width="560" height="315" src="https://www.youtube.com/embed/oGJeqkvaS08" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Control who can integrate and use a mentor via **LTI** from your LMS (e.g., Canvas), and who can see the mentor inside the **mentorAI web app**.

---

## LTI Accessible Toggle (Per-Mentor)

1. Open the mentor’s **Settings**.
2. Find **LTI accessible** and toggle:

   - **On** → The mentor appears in your LMS’s External Tool / Deep Linking picker and can be added to a course.  
   - **Off** → The mentor does not appear in the LMS picker; any existing LTI link will show an error after refresh.

### Canvas example (when LTI accessible = On):

- In a course: **Add External Tool → choose your mentor integration → select the mentor (e.g., AI Socratic Mentor) → Add Item → launch and chat.**
- If you later toggle **Off** and **Save**, refreshing the Canvas item shows an **error**, and the mentor is **removed** from the add-list.

---

## “Administrators can view” (Web-app visibility)

- If the mentor is marked **admin-only** in the mentorAI web app **and** LTI accessible is **On**:  
  - Other users won’t see the mentor in the web app  
  - But **students can still access it in the LMS**
- Use this to keep a mentor **hidden in the web UI** while leaving **LMS access intact**.

---

## Who Sees Mentors in the LMS Picker

- **Canvas admins:**  
  - Do **not** need a mentorAI account  
  - See **all mentors** available to the LMS environment (system-admin level)

- **Instructors:**  
  - **Must** have a mentorAI account using the **same email** as in the LMS  
  - This filters the LMS picker so instructors only see **mentors they created**

---

## Typical Workflow

1. In mentorAI → open the mentor’s **Settings** → toggle **LTI accessible = On** → **Save**.
2. *(Optional)* Set **Administrators can view** if you want the mentor **hidden in the web app but still usable from the LMS**.
3. In Canvas:  
   - **Add External Tool → choose your mentor integration → select the mentor → Add Item → launch**
4. To **revoke LMS access**:  
   - Toggle **LTI accessible = Off** → **Save**  
   - Existing LMS links **error after refresh** and the mentor **disappears** from the add-list.

---

## Results & Expectations

- **On** → mentor appears in LMS picker; launchable in courses.  
- **Off** → mentor disappears from LMS picker; existing links fail on refresh.  
- **Admin-only (web) + On (LTI)** → mentor hidden in web app but available to students in LMS.

---

Use these controls to manage **LTI visibility and access** without exposing mentors broadly in the web app.


