# Administration

<iframe width="560" height="315" src="https://www.youtube.com/embed/oGJeqkvaS08" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Control who can integrate and use an agent via **LTI** from your LMS (e.g., Canvas), and who can see the agent inside the **Agentic OS web app**.

---

## LTI Accessible Toggle (Per-Agent)

1. Open the agent’s **Settings**.
2. Find **LTI accessible** and toggle:

   - **On** → The agent appears in your LMS’s External Tool / Deep Linking picker and can be added to a course.  
   - **Off** → The agent does not appear in the LMS picker; any existing LTI link will show an error after refresh.

### Canvas example (when LTI accessible = On):

- In a course: **Add External Tool → choose your agent integration → select the agent (e.g., AI Socratic Agent) → Add Item → launch and chat.**
- If you later toggle **Off** and **Save**, refreshing the Canvas item shows an **error**, and the agent is **removed** from the add-list.

---

## “Superadministrators can view” (Web-app visibility)

- If the agent is marked **admin-only** in the Agentic OS web app **and** LTI accessible is **On**:  
  - Other users won’t see the agent in the web app  
  - But **users can still access it in the LMS**
- Use this to keep an agent **hidden in the web UI** while leaving **LMS access intact**.

---

## Who Sees Agents in the LMS Picker

- **Canvas admins:**  
  - Do **not** need an Agentic OS account  
  - See **all agents** available to the LMS environment (system-admin level)

- **Administrators:**  
  - **Must** have an Agentic OS account using the **same email** as in the LMS  
  - This filters the LMS picker so administrators only see **agents they created**

---

## Typical Workflow

1. In Agentic OS → open the agent’s **Settings** → toggle **LTI accessible = On** → **Save**.
2. *(Optional)* Set **Superadministrators can view** if you want the agent **hidden in the web app but still usable from the LMS**.
3. In Canvas:  
   - **Add External Tool → choose your agent integration → select the agent → Add Item → launch**
4. To **revoke LMS access**:  
   - Toggle **LTI accessible = Off** → **Save**  
   - Existing LMS links **error after refresh** and the agent **disappears** from the add-list.

---

## Results & Expectations

- **On** → agent appears in LMS picker; launchable in courses.  
- **Off** → agent disappears from LMS picker; existing links fail on refresh.  
- **Admin-only (web) + On (LTI)** → agent hidden in web app but available to users in LMS.

---

Use these controls to manage **LTI visibility and access** without exposing agents broadly in the web app.


