# Project: LLM Needle-in-a-Haystack Evaluation for Italian Legal Precedents

## 1. Task Overview

A "needle-in-a-haystack" (NiaH) eval to assess the long-context retrieval and reasoning capabilities of Large Language Models (LLMs). The test will use Italian Supreme Court (Cassazione) case holdings ("massime ufficiali") as both "needles" (target documents) and "haystack" (distractor documents).

The evaluation will assess the model's ability to:
1.  Retrieve a specific target document ("needle") from a large corpus of distractor documents ("haystack") across varying context lengths (log scale with 8 bins from 8k and up to 1M  tokens). Each document is composed by the principle of law stated in a massima, plus an obfuscated year and identifier (the holding and court case years and identifiers will be obfuscated to prevent memorization from the training data).
2.  Understand and apply legal concepts of "Conformi" (agreement/conformity) and "Difformi" (dissonance/contrast) in identifying the correct needle.
3.  Adhere to constraints, such as retrieving the "most recent" document when multiple conformi documents exist.
4.  Perform accurately despite anonymized identifiers and dates, forcing reliance on in-context information.

## 2. Core Evaluation Goals

*   Measure retrieval accuracy across a 8x10 matrix of context lengths (8 bins) and needle placement positions (10 positions, and 10% intervals of the context lenght).
*   Differentiate performance on "Conformi" vs. "Difformi" retrieval tasks (300 conformi queries and 60 difformi queries are available).
*   Assess the impact of easy matches (query and needly with high or near complete textual overlap) and hard semantic matches in "Conformi" pairs.
*   Evaluate the model's ability to handle the "most recent" constraint.
*   Test robustness against data contamination by using anonymized document metadata within the haystack.

## 3. Available Data & Materials

### 3.1. Needle/Query Pairs
*   **Total Pairs:** 360 pairs, derived from citations in Italian case holdings. Each pair consists of a "query document" and a "needle document."
    *   **Conformi Pairs:** 300 pairs, where the needle document expresses the same legal interpretation as the query document.
        *   **Sub-types of Conformi (to be potentially categorized):**
            *   C1: Identical/Near-Identical text between the query document and the needle document.
            *   C2: High textual overlap with minor variations/additions.
            *   C3: Conceptually Conformi but with lower textual overlap (semantically harder).
    *   **Difformi Pairs:** 60 pairs, where the needle document expresses a contrasting or opposite legal interpretation to the query document.
*   **Note on Definitions:** An official definition of Conformi/Difformi will be provided to the LLM. However, actual citation practices may include "grey areas" where Conformi criteria are not strictly met. The evaluation will use actual citation links as ground truth.

### 3.2. Distractor Documents
*   Up to 12,500 other Italian case holdings (massime) available to be used as distractor documents in the haystack.
*   Average document length: ~256 tokens.

### 3.3. Metadata
*   Each case holding (needles, queries, distractors) has associated metadata:
    *   `ruling_number`
    *   `ruling_year`
    *   `holding_id` (e.g., "Rv. 652412 - 01")
    *   Full text of the `holding_principle`.
*   This metadata will be crucial for the "most recent" task and will be subject to anonymization within the haystack (preserving the relative order of dates and progressive identifiers).

## 4. Methodological Design

### 4.1. Test Matrix & Runs
*   **Matrix:** 8 context length ranges x 10 needle placement positions = 80 unique cells.
    *   **Context Lengths:** Logarithmically spaced from 8k up to 1 million tokens.
    *   **Needle Placements:** Distributed across the context (e.g., 0%-10%, 10%-20%, ..., 40-50% (middle), ..., 90%-100% (bottom)).
*   **Total Test Runs:** (to be confirmed) 360 (one for each unique needle/query pair). This results in an average of 4.5 runs per cell in the 8x10 matrix.

### 4.2. Needle/Query Pair Allocation

#### 4.2.1. Difformi Pairs Strategy (60 pairs)
*   **Goal:** Maximize challenge and insight from these harder pairs.
*   **Distribution:**
    *   Assign all 60 Difformi pairs to the **central placement positions** (30%-80%) within the context.
    *   Exclude the first ~20% and the last ~20% of placement positions for Difformi tests.
    *   Each of the 8 context length ranges will receive `60 pairs / 8 ranges = 7-8` Difformi tests.
    *   These 7-8 tests per context range will be distributed among the chosen central placement positions.

#### 4.2.2. Conformi Pairs Strategy (300 pairs)
*   **Distribution:**
    *   The 300 Conformi pairs will fill the remaining test slots across all 80 cells.
    *   Cells at edge placements (not receiving Difformi tests) will receive only Conformi tests.
    *   Cells in central placements will receive a mix of Difformi and Conformi tests.
    *   **Consideration:** Attempt to distribute the C1/C2/C3 Conformi sub-types across cells, or at least track which type is used for nuanced analysis.

### 4.3. Haystack Construction

#### 4.3.1. Document Selection & Placement
*   For each of the 360 test runs:
    1.  Select one needle/query pair. The "needle document" is the target.
    2.  Determine the target context length based on the matrix cell.
    3.  Randomly sample the required number of distractor documents from the 12.5k pool to build the haystack.
        *   Ensure the chosen needle document and its corresponding query document are not accidentally included in the sampled distractors for that run.
    4.  Place the (anonymized) needle document at the designated position within the haystack.

#### 4.3.2. Anonymization
*   **Rationale:** To prevent the LLM from exploiting memorized knowledge of publicly available documents and their citation relationships, forcing true in-context reasoning.
*   **Dates (`ruling_year`, etc.):**
    *   Implement **global sequential anonymization**. Collect all unique dates, sort them, and assign sequential anonymous date IDs (e.g., `ANON_DATE_1`, `ANON_DATE_2`). These anonymous IDs will be used in the haystack.
*   **Identifiers (`holding_id`, `ruling_number`):**
    *   Implement **per-run anonymization**. For each test run, create a temporary, run-specific mapping from original document IDs to new anonymous IDs (e.g., `ANON_DOC_ID_A`, `ANON_DOC_ID_B`). These anonymous IDs will be used in the haystack.
*   **Metadata Presentation in Haystack:**
    ```
    --- DOCUMENT START ---
    ANON_DOC_ID: DOC_087
    ANON_DATE_ID: DATE_152
    HOLDING_PRINCIPLE: [Text of the holding principle...]
    --- DOCUMENT END ---
    ```

### 4.4. Prompt Design
*   **Structure:** Haystack first, then instructions and query.
    ```
    [START OF HAYSTACK]
    (Anonymized Document 1...)
    (Anonymized Document 2...)
    ...
    (Anonymized Needle Document somewhere within)
    ...
    (Anonymized Document M...)
    [END OF HAYSTACK]

    Instructions:
    (...)
    ```
