# Automated Document Screening

<iframe width="560" height="315" src="https://www.youtube.com/embed/4IjsTNUv4ps" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Use an agent as an automated document screener that evaluates uploaded files against custom criteria and returns a pass/fail result with a detailed breakdown.

---

## Define Screening Criteria

1. Open the agent's **Prompts** tab.
2. In the system instructions, specify:
   - The criteria a document must meet (e.g., name present, years of experience, required skills)
   - Whether **all** criteria must pass or just some
   - The output format for a pass (e.g., a table with a check icon)
   - The output format for a fail (e.g., a bold red fail icon with a table of unmet criteria)

### Example Criteria

- Document contains the **name of an individual**
- Individual has at least **two years of experience** as a programmer
- Individual has experience with **Python and JavaScript**
- All three must be met for a pass

---

## Using the Canvas Side Panel

1. Enable **Canvas view** on the agent to display results in a side panel.
2. Upload a document (PDF, DOCX, etc.) in the chat.
3. The agent analyzes the document against your criteria and renders the result in the side panel.

---

## Example Results

**Fail** (uploading a receipt instead of a resume):
- Name of individual: No
- Two years experience: No
- Python & JavaScript: No
- Result: **FAIL**

**Pass** (uploading a qualifying resume):
- Name of individual: Yes
- Two years experience: Yes
- Python & JavaScript: Yes
- Result: **PASS**

---

## Customization Options

- Results can appear in the **side panel** (Canvas view) or the **regular chat frame**
- You can send a **custom prompt** alongside the document upload
- Adjust criteria, output formatting, and pass/fail thresholds entirely through the system prompt
