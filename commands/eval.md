---
name: eval
description: Run self-scoring evals for a specific skill.
---

# Eval Command

Use this command to execute self-scoring evaluation against a prompt-only skill co-located inside the `skills/` directory.

## 🎯 Command Usage
`/eval <platform>/<skill-name>` or `/eval <skill-name>`

If only `<skill-name>` is provided, the command will scan all platforms for a matching skill name.

---

## 🧭 Execution Protocol

As an LLM test runner, you must execute the evaluation using the following steps:

### Phase 1: Locate the Skill and Assets
1. Traverse the `skills/` directory to locate the folder matching the requested skill name.
2. Read the `SKILL.md` file (which contains the system instructions for the skill) and the co-located `eval.md` file.
3. Parse the frontmatter of the skill's `eval.md`:
   - If `fixtures:` is specified, load the fixtures from the central path (e.g. `evals/fixtures/some-skill.json` or `evals/fixtures/some-skill/`).
   - If `rubric:` is specified, load the rubric from the central path (e.g. `evals/rubrics/some-skill.md`).
   - If frontmatter is absent, fall back to reading the inline fixtures and rubric from the body of `eval.md`.

### Phase 2: Execute Fixtures (Simulated Sandbox)
For each loaded fixture:
1. **Initialise Sandbox:** Isolate your agent environment. Load the full contents of the `SKILL.md` as your system prompt / instruction set.
2. **Inject Input:** Supply the fixture's input to the sandbox using the correct injection mode:
   - **Single-turn (`input` is a string, or points to a code file):** Pass the string (or file contents) as the user's message.
   - **Multi-turn (`input` is an object containing a `turns` array):** Replay each entry of `turns` **sequentially as a separate user message within the same sandbox session**, preserving the accumulated state (prior tool calls, emitted JSON, and reasoning) between turns. Each turn object provides a `user` message; an optional `tool_output` field simulates the JSON a tool would return on `stdout` for that turn, which you must treat as observed reality when generating the next turn. Do not collapse the turns into a single blob.
3. **Execute:** Generate the raw response exactly as the skill would respond in a production environment. Do not add any evaluator commentary or meta-text to the generated response.
4. **Collect:** Store the generated output (the full ordered transcript for multi-turn fixtures) for grading.

### Phase 3: Rubric Evaluation
For each generated output:
1. Load the corresponding **Rubric** and **Calibration Guide** (from the central file or `eval.md`).
2. Inspect the output objectively.
3. For each criterion in the rubric (e.g. 1.1, 1.2, etc.):
   - Assess whether the output passes, fails, or if the criterion is not applicable (N/A).
   - Assign a score of **PASS (1)**, **FAIL (0)**, or **N/A**.
   - Write a short, precise justification referencing the specific parts of the output that dictated the score.

### Phase 4: Compile & Report
Generate a final evaluation report containing:
1. **Overall Summary Table:**
   - Fixture ID & Description
   - Criteria Scored
   - Total Score = `sum(PASS) / sum(applicable) * 100` (%)
   - Status Rating (90-100% = Excellent, 75-89% = Good, 60-74% = Needs work, <60% = Poor)
2. **Detailed Breakdown:**
   - A section for each fixture showing the input, expected outcome, actual generated response, and individual criteria scores with justifications.
   - For multi-turn fixtures, show the ordered turn-by-turn transcript.
3. **Actionable Recommendations:**
   - If the score is less than 100%, suggest concrete edits to the `SKILL.md` prompt to prevent those failure modes.
