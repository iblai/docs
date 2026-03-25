# Blackboard

<iframe width="560" height="315" src="https://www.youtube.com/embed/k91vOq4CANg" frameborder="0" allowfullscreen></iframe>

## Purpose

Connect a **Blackboard course** as a data source so a mentor can read course content and attached files, cite them, and answer questions from that material.

---

## Part 1 — Install the REST API Integration (Blackboard Admin)

1. In the **Blackboard Developer Panel**, create (or use) an application and copy its **Application ID** (this ID will be shared for installations).  
2. In your Blackboard instance, open the **Admin Panel → search REST API Integrations**.  
3. Click **Create Integration** and paste the **Application ID**.  
4. Choose a user to link to the application (pick one who has access to the course).  

   **Permissions noted in the demo:**
   - Authorized to act as user: **Not needed**  
   - Any user access: **Yes**

5. Submit. The test application shows as **integrated**.

---

## Part 2 — Add the Blackboard Course as a Data Source (Mentor Platform)

1. In Blackboard, open the **target course** and copy the **course URL** from your browser.  
2. In the mentor’s **Datasets**, choose **Blackboard** as the data source.  
3. Paste the **course URL** and click **Submit**.  
4. The document queues for **training**; once trained, it appears in the list.  
5. *(Optional)* Mark it **Visible** so you can see cited content.

---

## How It Works in Chat

- The mentor can answer questions from the **course’s text content** and **attached files** (e.g., PDFs).  
- Source snippets appear, showing exactly **where the answer was drawn from**.  

**Example (from the demo):**  
A question about **Las Casas’s writing** returns an answer about Spanish colonists’ treatment of Indigenous people, with snippets pointing to the relevant attached documents/sections.

---

## Auto-Retraining

- By default, the Blackboard content is set to **auto-retrain every 7 days**.  
- You can reschedule this to any number of days.

---

## Result

Your mentor now **ingests the specified Blackboard course and its attachments**, cites where answers come from, and stays **up to date via scheduled retraining**.

