# Brightspace 

<iframe width="560" height="315" src="https://www.youtube.com/embed/xePQv8VC8Cc" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Prerequisites

![](/images/brightspacelti.png)

- A **System Admin** account in Brightspace.  
- All LTI 1.3 parameters from mentorAI:

  - **Domain**  
  - **Redirect URL(s)**  
  - **OpenID Connect Login URL**  
  - **Key-set (JWK) URL**  
  - **Target Link URI** (points to the specific mentor you want to surface)

## Register the Tool

- Log in as a system admin and click the **gear icon**.  
- Choose **Manage Extensibility**.  
- Open the **LTI Advantage** tab.  
- Click **Register Tool → Standard**.  
- Fill out:

  - **Name and Description** (anything you like)  
  - Paste the **Domain**, **Redirect URL(s)**, **OpenID Connect Login URL**, and **Key-set URL** provided by mentorAI  
  - **Target Link URI** → link to the exact mentor (varies per integration)

- Leave **Extensions**, **Roles**, **Substitution Parameters**, and **Custom Parameters** blank unless mentorAI instructs otherwise.  
- Save the registration.  
- Copy the **Registration Details** (they include the issuer, client ID, etc.) and send them back to mentorAI so they can complete their side of the setup.

## Create a Deployment

- In the same **LTI Advantage** area, click **View Deployments**.  
- Choose **New Deployment** (or open the one you just created).  
- Under **Security Settings**, check the user-related boxes so Brightspace passes learner identity to mentorAI.  
- If the mentor will push grades back, enable **Assignments and Grade Services**.  
- Under **Configuration Settings**, tick **Make Tool Available to the Org** so any course can use it.  
- Save the deployment.  
- Copy the **Deployment ID** and provide it to mentorAI (needed to finish the integration).

## Create a Link to the Mentor

- Still inside the deployment, click **View Links → New Link**.  
- Fill out:

  - **Name** (e.g., “mentorAI – Biology Tutor”)  
  - **URL** → the same **Target Link URI** you used in registration  
  - **Type of Launch** → Basic

- Save the link.

## Add the Mentor to a Course

- Return to the Brightspace homepage and open a course.  
- Navigate to the content area or module where you want the mentor.  
- Click **Add Existing → External Tool Activity**.  
- Select **mentorAI** from the tool list.  
- **Publish** the item so learners can see it.

## Result

- **Register Tool** with mentorAI-supplied URLs.  
- **Deploy it**, enabling user identity (and grade services if needed).  
- **Create a Link** pointing to the specific mentor.  
- **Insert the link** into any course via **External Tool Activity**.  

Your mentorAI assistant is now live in Brightspace, ready to help learners directly inside their course pages.

---

# Brightspace Deep Linking

<iframe width="560" height="315" src="https://www.youtube.com/embed/u7gROD8zEXs" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Integrate **Mentor AI content** into **Brightspace courses** using **LTI Deep Linking**.  
This allows instructors to insert specific mentors directly into course content modules.

## Step 1 — Configure the Tool in Brightspace

1. Go to the **Manage Extensibility** section as an admin  
2. Open the **LTI Advantage** tab  
3. Add a **new tool** or update an **existing one**  
   - No main registration settings need to change  
4. Under **Extension Settings**, make sure **Deep Linking** is enabled  
5. Go to **Deployments**  
6. Open an existing deployment  
7. Confirm that **Deep Linking** is marked  
8. *(Optional)* Adjust additional settings if needed (e.g., send user info)  
9. **Save changes**

## Step 2 — Add the Deep Linking Launch

1. In the **Deployments**, go to **Links**  
2. Add or edit a link that uses the **Deep Linking Launch URL**  
3. Adjust display settings:  
   - **Height and width**  
   - **Type** should be **Quicklink** so content can be selected  
4. **Save and close**

## Step 3 — Insert Into a Course

1. Open a **course** in Brightspace  
2. Navigate to **Course Content**  
3. Select a unit or create one  
4. Click **Add Existing**  
5. Choose the **Deep Linking option** (instead of the standard external tool)  
6. Select the **deep linking configuration** you set earlier  
7. A **content picker** opens, showing the mentors available for your tenant  
8. Mentors display with their names for easy selection  
9. Pick the desired mentor and **add it to the course**

## Result

The chosen mentor appears as **integrated course content**. Learners can **launch it directly**, and instructors can **repeat the process** for additional mentors or resources.

---

# Course to Deep-Link

<iframe width="560" height="315" src="https://www.youtube.com/embed/gfqIb8RyIxs" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Set up a **mentor for LTI Deep Linking** so you can add it to your **LMS** (e.g., Brightspace) yourself—assuming the base integration exists and deep linking is enabled.

## 1) Get the Mentor ID

1. Open the mentor you want to integrate.  
2. Copy the **unique identifier** from the mentor’s URL.

## 2) Create the Course in Studio

1. Go to **[ibl.ai Studio](https://studio.learn.iblai.app)**.  
2. Create a new course (e.g., “Socratic Mentor”) and choose the organization.

### Settings → Schedule & Details

- Set the **Course Start Date** to a time in the past.  
- Set **Enrollment Start** to at least a day before the start date.  
- Click **Save**.

### Settings → Advanced Settings

- In the **advanced modules list**, add: `"ibl mentor_xlog"`  
  *(This enables the mentor component.)*  
- **Save** changes.

## 3) Add the Mentor Component

1. Go to **Outline → add Section → Subsection → Unit**.  
2. Click **Advanced → Add New Component → Mentor**.  
3. Click **Edit** and paste the mentor’s **unique ID** you copied earlier.  
4. Set the **Display Name** (e.g., “Socratic Mentor”).

### Optional Settings (from the demo)

- **Context Awareness:** Enter your LMS domain; copy it into the **Mentor Domain** and **Domain** fields.  
- **Anonymous:** Toggle if you want users to chat without authentication.  
- **Advanced View:** Enable tabs like **Summarize, Translate, Expand**.  

Click **Save** and **Publish** the unit.

## Result

Your mentor is **published in Studio** and **selectable via deep linking** in your LMS—letting you add mentors to courses on your own.
