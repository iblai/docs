# Data Sources

<iframe width="560" height="315" src="https://www.youtube.com/embed/xL_HcXuyGeo" 
title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Description

Dataset Visibility in Agentic OS lets instructors control whether learners can see and open the exact resources (“datasets”) that the agent used to generate an answer. When visibility is enabled, a Retrieved Documents side panel appears in chat so learners can explore the source material themselves. When visibility is disabled, Agentic OS still uses the dataset behind the scenes, but the source files remain hidden—useful when you want the AI to draw on proprietary, assessment, or advance material without revealing it.



![](/images/datasources.png)

---

## Target Audience

**Student**

---

## Features

#### Learner Transparency Toggle

Decide, per dataset, whether students may open the exact resources Agentic OS retrieved.

#### Retrieved Documents Side Panel

When visibility is on, chat answers are accompanied by a clickable list of source files so learners can read, cite, and verify the material.

#### Non‑Destructive Control

Toggling visibility does not remove the dataset from Agentic OS’s training; it only controls whether learners can access the documents.

#### One‑Click Icon Interface

An eye icon (👁️ = visible, 👁️‍🗨️ = hidden) in the Datasets tab makes it effortless to turn visibility on or off.

#### Automatic Retraining (if needed)

When a dataset is made visible again after being hidden, Agentic OS seamlessly retrains on that content to ensure up‑to‑date retrieval.

---

## How to Use (step by step)

#### Open Settings

In your agent admin view, click **Settings**.

#### Select the Datasets Tab

You’ll see a table of every resource collection used to train this agent.

#### Locate the Desired Dataset

Scroll or search to find the dataset whose visibility you want to adjust.

#### Check the Eye Icon

- 👁️ (no slash) = learners currently see this dataset in the side panel.  
- 👁️‍🗨️ (with slash) = learners cannot open this dataset.

#### Toggle Visibility

Click the eye icon to switch states.

- Turning off (👁️ → 👁️‍🗨️) hides the resource from students; Agentic OS still uses it to answer questions.  
- Turning on (👁️‍🗨️ → 👁️) reveals the resource and retrains the agent if necessary.

> **Note:** Visibility only affects learner access. The dataset remains in the agent’s knowledge base unless you explicitly remove it.

---

## Pedagogical Use Cases

#### Source Transparency & Citation Practice

Enable visibility so learners can open primary sources, encouraging proper citation and critical evaluation of evidence.

#### Scaffolded Learning Paths

Start courses with visibility off to prevent information overload; toggle on later to let advanced students explore deeper materials.

#### Controlled Assessment Support

Keep answer keys or formative‑assessment rubrics hidden while still letting Agentic OS reference them to provide feedback.

#### Encouraging Independent Research

By showing retrieved documents, you prompt learners to read beyond the AI’s summary, fostering information‑literacy skills.

#### Selective Disclosure of Proprietary Content

Hide internal documents from external cohorts while maintaining the agent’s ability to leverage that expertise.

---

## Quick Reference

- **Visible resources:** learners see and can open them in the Retrieved Documents panel.  
- **Hidden resources:** Agentic OS still uses them, but learners cannot access the files.
