# Blackboard 

<iframe width="560" height="315" src="https://www.youtube.com/embed/VDBKUMKWW2Y" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/LpdYOKA8xHU" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Prerequisites

- Obtain the **Client ID** and the **Target Link URI** (tool launch link) from the **Agentic OS** team.  
- Obtain the **LTI launch URL** (and any other parameters) if you’ll add the tool directly at the course level.  
- Be an **instructor (or admin)** with permission to add **LTI tools** in your Blackboard course.

## Admin Setup (One-Time)

1. In the **Admin Panel**, open **Integrations → LTI Tool Providers**.  
2. Register a **new tool** (or edit an existing one).  
3. Paste the **Client ID** and **Submit**.  
4. The registration form auto-populates from the Client ID.  
5. Optionally, choose user info to send (e.g., name, email).  
   - Grade services, membership services, and user acknowledgment are optional in this demo.  
6. **Create Placements** for how the tool appears in courses:
   - **Course Content Tool** – adds the agent inside courseware.  
   - **Deep Linking** – opens a content picker to choose specific agents/resources.  
   - **Course Tool** – adds an always-available course-level entry point.  

   For each placement:
   - Give it a **Name** (and optional Description).  
   - Set a unique **Handle**.  
   - Mark **Available** and set **Placement Type** (Course Content, Deep Linking, or Course Tool).  
   - For **Course Content Tool**, enable **Allow Grading** if you want grade passback.  
   - Ensure **Allow Student Access** is on when ready for learners.  
   - Set the **Target Link URI** (same as the tool launch link for standard launches; deep linking uses its specific launch).  
   - **Save** your changes.

## Blackboard Ultra – Workflow A (Teaching Tools with LTI Connection)

1. Open your course in **Blackboard Ultra**.  
2. Navigate to the folder/area where you want the agent.  
3. Click **Create → Teaching Tools with LTI Connection**.  
4. Fill the form:
   - **URL:** paste the Agentic OS launch URL.  
   - **Name:** e.g., “EN Comp AI”.  
   - **Description:** short friendly description.  
   - **Open in a New Window:** enable if you prefer a separate tab.  
5. **Save** and toggle **Visible to Students**.  
6. Test by clicking the item (it opens embedded or in a new window per your choice).

## Blackboard Ultra – Workflow B (Content Market using a Placement)

1. Open the course.  
2. Click **Add Content → Content Market**.  
3. Select your **Course Content Tool placement** (created in Admin Setup).  
4. The item is added; launch it to start chatting with the agent.

## Blackboard Original (Legacy Interface)

1. Open the classic-layout course.  
2. Go to **Build Content → Web Link**.  
3. Provide:
   - **Name** of your AI agent.  
   - **URL** (Agentic OS launch link).  
   - Check **This link is a tool provider** (marks it as LTI).  
   - Optional **Description**.  
4. **Submit** and launch from the new link.

## Result

- **Admin Setup:** Register the tool with **Client ID**, create **Placements**, set the **Target Link URI**, and enable student access.  
- **Ultra:** Use **Teaching Tools with LTI Connection (Workflow A)** or **Content Market with your placement (Workflow B)**.  
- **Original:** Build Content → Web Link → Tool Provider.  

Always use the **URLs/IDs provided by Agentic OS** and make the item **visible** so users can access the assistant.

---

# Course to Deep-Link

<iframe width="560" height="315" src="https://www.youtube.com/embed/gfqIb8RyIxs" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Set up a **agent for LTI Deep Linking** so you can add it to your **LMS** (e.g., Blackboard) yourself—assuming the base integration exists and deep linking is enabled.

## 1) Get the Agent ID

1. Open the agent you want to integrate.  
2. Copy the **unique identifier** from the agent’s URL.

## 2) Create the Course in Studio

1. Go to **[ibl.ai Studio](https://studio.learn.iblai.app)**.  
2. Create a new course (e.g., “Socratic Agent”) and choose the organization.

### Settings → Schedule & Details

- Set the **Course Start Date** to a time in the past.  
- Set **Enrollment Start** to at least a day before the start date.  
- Click **Save**.

### Settings → Advanced Settings

- In the **advanced modules list**, add: `"ibl mentor_xlog"`  
  *(This enables the agent component.)*  
- **Save** changes.

## 3) Add the Agent Component

1. Go to **Outline → add Section → Subsection → Unit**.  
2. Click **Advanced → Add New Component → Agent**.  
3. Click **Edit** and paste the agent’s **unique ID** you copied earlier.  
4. Set the **Display Name** (e.g., “Socratic Agent”).

### Optional Settings (from the demo)

- **Context Awareness:** Enter your LMS domain; copy it into the **Agent Domain** and **Domain** fields.  
- **Anonymous:** Toggle if you want users to chat without authentication.  
- **Advanced View:** Enable tabs like **Summarize, Translate, Expand**.  

Click **Save** and **Publish** the unit.

## Result

Your agent is **published in Studio** and **selectable via deep linking** in your LMS—letting you add agents to courses on your own.
