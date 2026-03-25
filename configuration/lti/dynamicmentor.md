# Dynamic Mentor Integration (LTI 1.3)

<iframe width="560" height="315" src="https://www.youtube.com/embed/-R6nQbwyICc" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Overview

A dynamic mentor integration using LTI 1.3 that allows different mentors to appear in different courses within the same LMS environment. This example uses Canvas but applies to other LMS platforms.

---

## Admin Setup: Developer Keys

1. In Canvas, go to **Admin → Developer Keys**.
2. Click **Developer Keys → LTI Key** to create a new integration.
3. Configure the parameters (provided by the mentorAI team):
   - **Redirect URIs**
   - **Target Link URI**
   - **OpenID Connect Initiation URL**
   - **JWK Method**: Public JWK URL
4. Under **LTI Advantage Settings**, enable user data sharing (email, name) for reporting visibility.
5. Add the **custom field** that enables dynamic mentor selection per course.
6. Set **Privacy Level** to Public.
7. Set **Placements**: Account Navigation and Link Selection (defaults).

---

## Course Navigation Placement

Add **Course Navigation** as a placement:

- **Default enabled**: the mentor appears in every course's side navigation automatically
- **Default disabled** (via Paste JSON): the mentor does not appear in side navigation unless an instructor enables it

To disable by default, use Paste JSON and set `"default": "disabled"` under the course navigation placement.

---

## Instructor Configuration

1. Open the course where you want the mentor.
2. An instructor/admin panel shows an **Enable** toggle and a **Mentor ID** field.
3. Get the mentor's unique ID from the mentor platform URL (the segment after the last `/`).
4. Paste the Mentor ID and click **Save**.

---

## Student View

- Students see the mentor in the **side panel** and can chat directly.
- If both the side panel and course navigation are enabled, the mentor appears in both locations.
- Students can close the side panel and use the course navigation page exclusively.

---

## Hiding the Course Navigation Link

As an instructor:
1. Go to **Course Settings → Navigation**.
2. Drag the mentorAI item to the **hidden** section.
3. Click **Save** — the mentor no longer appears in the course navigation for students.

As an admin:
- Set `"default": "disabled"` in the Developer Key JSON to prevent the course navigation link from appearing globally.
- Instructors can still enable it per-course if the admin leaves the option available.

---

## Key Takeaways

- **Dynamic integration** allows different mentors per course using a single LTI tool
- **Mentor ID** links each course to a specific mentor from the platform
- **Course navigation** and **side panel** are independent — you can enable either or both
- **Instructors** control per-course visibility; **admins** control global defaults
