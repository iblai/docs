# Document Retrieval

<iframe width="560" height="315" src="https://www.youtube.com/embed/shdYfSObDS8" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Description

**Document Retrieval** makes every Agentic OS answer **transparent and verifiable**.  
When a learner asks a question, the agent:

1. **Cites** the exact source inside the reply (e.g., *“Lecture 11 — Slides 35–36”*)  
2. **Shows** a dynamic **Source Panel** with the documents it used, ranked by relevance  
3. **Lets users open** any listed file with one click to read the full context  

Administrators control which materials can be shown by toggling each file’s **Visible** switch in the **Datasets** table—**no retraining required**.

---

## Target Audience

**Student · Instructor · Administrator**

---

## Features

#### Inline Citations in Answers
Replies reference the exact lecture, slide, or page used (e.g., *Lecture 11 — Slides 35–36*).

#### Source Panel (Ranked by Relevance)
A live panel lists retrieved documents for that specific answer and updates as the conversation continues.

#### One-Click Source Opening
Learners can open any listed file to read supporting context immediately.

#### Admin Visibility Controls
Per-file **Visible** toggles determine which sources can be shown/cited in the panel—without removing them from training.

#### Works at Scale
Handles large training sets; sources are still ranked and cited for each response.

#### Guided-Prompt Friendly
Use guided prompts to kick off a conversation when learners aren’t sure where to start.

---

## How to Use (step by step)

#### Ask a Question in Chat
- Example:  
  > “What are key epidemiological study designs?”  
- The agent reads your query, searches trained resources, and composes an answer with inline citations  
- Example citation:  
  > “These study designs are discussed in Lecture 2.10 of Prof. Quinlan’s course.”

#### Review the Source Panel
- The panel displays the documents used, ranked by relevance (often with a confidence/percentage indicator)  
- Click any source to open the original and read more

#### Ask Follow-Ups (Panel Updates Automatically)
- Example:  
  > “Can you explain case-control studies in detail?”  
- The Source Panel refreshes to show the most relevant documents for the new question and cites them in the reply (e.g., *“See Lecture 11, Slides 35–36”*)

#### Open & Read Sources
- Select a listed document (lecture, slide deck, PDF) to view full context and deepen understanding

#### (Admin) Control Visibility of Sources
- Go to **Settings → Datasets** for the agent  
- Use the **eye icon** in the Visible column to show or hide individual files:  
  - **Visible On** → learners can see/click the source in the panel  
  - **Visible Off** → the agent can still use the file for answers, but it won’t appear in the panel  
- Changes apply instantly; **no retraining is required**

---

## Pedagogical Use Cases

#### Transparent, Citable Answers
Teach students to verify claims and cite original materials—great for research and academic integrity.

#### Guided Reading & Deep Dives
Learners jump straight from an answer to the exact slide/page for fuller context.

#### Instructor QA & Content Gaps
Instructors can confirm the agent cites the right sources and spot where additional materials are needed.

#### Assessment Support
Link explanations to specific readings so students revisit core texts before quizzes or exams.

#### Scaffolded Disclosure
Keep some documents hidden (**Visible Off**) for assessments or proprietary content, while still letting the agent draw on them to answer.

---
