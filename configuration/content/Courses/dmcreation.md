# DM Creation

<iframe width="560" height="315" src="https://www.youtube.com/embed/rtGp3lyulFs" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Where to Start

- Sign in as a **Super Admin**.  
- Open **Data Manager → Catalog App**.  
- You’ll import both **SCORM** and **video courses** from here.

---

## Prepare the CSV

- Click **Import** to see a minimalist upload screen.  
- Download the **sample CSV** to review required fields:  
  - `platform_key`, `course_name`, `section`, `subsection`, `unit`, `tags`, etc.—the same fields used in the single-course creation wizard.  
- Populate the CSV with one row per course you want to add.

---

## Import SCORM Courses

- In **Catalog → Courses**, press **Import**.  
- Choose **File** and select your completed CSV.  
- Click **Submit**.  
- A **validation dialog** appears—review the parsed rows.  
- Click **Confirm Import** if everything looks correct.  
- If any row contains errors (e.g., duplicate course), the system flags it and cancels that row; fix and re-upload as needed.  
- A **success message** lists every course that was added.

---

## Import Video Courses

- In **Catalog → Course Videos**, click **Import**.  
- Download the **sample CSV** to confirm field names (they’re shown on-screen).  
- Fill in the CSV, choose it, and run **Submit → Confirm Import** just like SCORM.  
- Success or error feedback appears immediately.

---

## Locate Imported Courses

- After import, all new **courses and videos** appear in the **Catalog App**.  
- They behave exactly like any manually created course; no further setup is required.

---

## Learner View Check

- Switch to the **Skills** front-end.  
- Open **Discover**.  
- Search for a course title from your CSV (e.g., SC B Upload).  
- The imported course is **visible and enrollable**, confirming a successful bulk upload.

---

## Error Handling Tips

- Re-uploading an identical CSV row triggers a **“course already exists”** error—update the row or remove duplicates.  
- Always **validate before confirming**; the dialog catches missing or malformed fields.

---

## Key Takeaways

- **Bulk import** is only for **SCORM** and **video course** types.  
- Use the **sample CSV** as your template and keep field names intact.  
- **Validation** ensures you don’t create duplicates or malformed entries.  
- Imported items instantly populate the **Catalog** and **Discover** views—no extra publishing step.  

With these steps you can mass-create courses in minutes, freeing super admins from repetitive single-course setup.
