# Analytics Q&A Agent

## Overview

This project is a lightweight natural-language interface for product analytics. It allows a user to ask questions such as:

* What was DAU on June 30, 2026?
* What was DAU for US users?
* What was D7 retention for the May signup cohort?
* Compare average DAU this week versus last week.
* Who are our best users?

The system converts clear analytics questions into SQLite queries, validates and executes them, and returns the underlying results. If a question is ambiguous, it asks for clarification instead of guessing.

The prototype uses a reproducible synthetic dataset containing 5,000 users, approximately 40,000 sessions, and 8,000 transactions across three tables: `users`, `sessions`, and `transactions`.

## 1. Architecture and Design Choices

The core pipeline is:

**User Question → LLM → READY or CLARIFY → SQL Safety Check → SQLite Execution → Result**

If SQL execution fails:

**SQL Error → LLM Repair Attempt → Safety Check → Retry Once → Result or Failure**

I used SQLite because the assignment required a small working prototype, and SQLite provides a simple way to demonstrate realistic SQL generation and execution without additional infrastructure.

The main agent uses Gemini 2.5 Flash. I chose a lightweight general-purpose model because this task primarily requires schema understanding, intent interpretation, and SQL generation rather than complex autonomous reasoning.

I deliberately did not use an agent framework such as LangChain. For this scope, the workflow is small and deterministic enough to implement directly. Adding a framework would increase abstraction and debugging time without providing enough value within the 4–6 hour time constraint.

The model receives:

* the database schema and table relationships;
* metric definitions such as exact-day D7 retention;
* the dataset end date for relative-date interpretation;
* rules for timestamp handling;
* instructions to distinguish answerable and ambiguous questions.

## 2. Handling Wrong or Hallucinated SQL

The prototype uses multiple lightweight safeguards.

First, the prompt restricts the model to tables and columns present in the supplied schema.

Second, generated SQL passes through a read-only validation layer. The prototype accepts read-only `SELECT` queries and CTE-based queries beginning with `WITH`, while blocking common destructive operations such as `DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`, and `CREATE`.

Third, if a query fails during SQLite execution, the system sends the original question, failed SQL, and database error back to the model for one repair attempt. The retry is intentionally limited to one attempt to control latency and API cost.

The system also cleans SQL returned inside markdown code fences before validation and execution.

A key limitation is that successful execution does not guarantee semantic correctness. During evaluation, one D7 retention query executed successfully but returned `0.1864` against a manually verified expected value of `0.194`. In percentage terms, this is approximately 18.64% versus 19.4%.

I kept this failure visible instead of tuning the prompt specifically to pass the evaluation case. It highlights that syntax validation and execution checks alone are not sufficient for trustworthy analytics.

In production, I would add stronger semantic verification using metric definitions, expected result-shape validation, query-plan checks, and comparison against trusted reference queries for important business metrics.

The current SQL validator is intentionally lightweight. A production implementation should use a proper SQL parser and enforce read-only access at the database permission level.

## 3. Evaluation

I created a 10-question evaluation set covering:

* simple aggregates;
* filtered aggregates;
* cohort retention;
* period comparisons;
* grouping questions;
* ambiguous questions requiring clarification.

The evaluation measures three things:

1. **Status accuracy** — whether the system correctly answers or asks for clarification.
2. **SQL execution success** — whether generated SQL executes successfully for answerable questions.
3. **Numeric accuracy** — whether single-value outputs match manually verified ground truth where available.

### Results

* **Status accuracy: 100%**
* **SQL execution success: 100%**
* **Numeric accuracy: 80%** across five questions with numeric ground truth

The single numeric failure was the D7 retention query, which returned `0.1864` against the verified expected value of `0.194`.

For comparison and grouping questions that return multi-column results, the current evaluator checks successful execution but does not automatically verify complete semantic correctness. This is a limitation of the current evaluation harness.

The main agent was developed using Gemini 2.5 Flash. The final evaluation run used Gemini 3.1 Flash Lite because the free-tier quota for the primary model was exhausted. Therefore, the evaluation is useful for testing the pipeline but is not a perfectly controlled same-model evaluation.

With more time, I would create a larger curated evaluation set containing manually reviewed SQL, expected result tables, paraphrased question variants, adversarial questions, and separate metrics for SQL correctness and final-answer correctness.

## 4. Scaling to a Warehouse with 100+ Tables

Passing the full schema of a large warehouse to the model would increase token cost, latency, and the probability of choosing incorrect tables or joins.

For a larger warehouse, I would use a staged architecture:

1. Retrieve relevant tables and columns using schema metadata, descriptions, embeddings, and query context.
2. Provide only the selected schema subset to the SQL-generation model.
3. Maintain a semantic layer containing approved definitions for metrics such as DAU, revenue, conversion, and retention.
4. Store trusted join paths and business rules rather than asking the model to infer them every time.
5. Execute queries through a read-only service account with query timeout, row limits, and cost controls.
6. Log questions, generated SQL, execution errors, latency, user corrections, and feedback for monitoring and continuous evaluation.

For frequently repeated questions, I would add caching based on normalized question intent, generated SQL, and data freshness requirements.

## Scope and Trade-offs

The project was intentionally scoped to the 4–6 hour constraint. I prioritized:

* a complete natural-language-to-SQL loop;
* all five required question behaviours;
* ambiguity handling;
* SQL safety checks;
* one-step SQL repair;
* a small evaluation harness.

I intentionally did not build a polished chat UI, caching layer, or chart generation. Given more time, my next priorities would be stronger semantic result verification, a proper SQL parser, multi-turn clarification handling, caching, and simple automatic visualizations for time-series and comparison queries.

## Running the Notebook

1. Open `analytics_qa_agent.ipynb` in Google Colab.
2. Add `GEMINI_API_KEY` to Colab Secrets and enable notebook access.
3. Run the setup and synthetic data generation cells.
4. Run the SQLite database and agent setup cells.
5. Use the `ask()` function to submit analytics questions.

Example:

`ask("What was DAU for US users on June 30, 2026?")`

The notebook generates the synthetic dataset and SQLite database, so no external dataset setup is required.

## Tools Used

* Python
* pandas
* SQLite
* Google Gemini API
* Google Colab

AI assistance was used during development for code guidance, debugging, and structuring the prototype. The final notebook was tested against manually calculated ground-truth queries and a small evaluation set.
