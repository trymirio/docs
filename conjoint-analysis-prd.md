# Conjoint Analysis PRD

Status: Draft

Owner: Product / Research / Engineering

Last updated: 2026-04-25

## Summary

Add a new structured survey question type to Versive for conjoint analysis.

Recommended MVP:

- Add a new question type: `conjoint`
- Support `choice-based conjoint (CBC)` only in v1
- Let researchers define attributes and levels in the study builder
- Auto-generate balanced choice tasks from that definition
- Let participants complete all tasks inside a single Versive question
- Store structured task-level responses in the existing message pipeline
- Add a new structured insight type that computes aggregate utilities and attribute importance
- Add a dedicated conjoint analysis card on the insights page

This is the right first step because it delivers real conjoint value without forcing us to solve adaptive conjoint, respondent-level hierarchical Bayes, or market simulation on day one.

## Problem

Versive already supports structured questions such as `multipleChoice`, `ranking`, `allocation`, `matrixRatingScale`, `cardSort`, and `treeTest`, but it does not support conjoint-style tradeoff analysis.

Researchers who want to measure preference tradeoffs currently have to leave Versive and use a specialized survey platform. That breaks the workflow for:

- study authoring
- respondent collection
- results export
- analysis
- reporting

We want a first-party conjoint workflow that fits the existing Versive study builder, interview runtime, insights pipeline, and report experience.

## Goals

- Let researchers run simple conjoint studies fully inside Versive
- Produce usable preference outputs, not just raw task responses
- Fit the existing structured-question architecture rather than creating a parallel system
- Keep the first version simple enough to ship safely

## Non-Goals For V1

- Adaptive choice-based conjoint (`ACBC`)
- MaxDiff
- Menu-based or configurator-style conjoint
- Respondent-level utilities via hierarchical Bayes
- Advanced market simulation
- Prohibitions / impossible combinations
- Partial-profile designs
- Alternative-specific constants beyond basic modeling support
- AI-generated conjoint study design
- Automatic translation or localization logic specific to conjoint levels
- AI simulation support for conjoint questions

## Recommendation: What We Should Support To Start

We should start with a narrow, credible CBC implementation.

### V1 product scope

- Question type ID: `conjoint`
- Method: `CBC` only
- Concepts per task: `2` or `3`
- Tasks per respondent: default `8`, configurable `6-12`
- Attributes per study: recommended `3-6`
- Levels per attribute: recommended `2-5`
- Response mode: choose exactly one concept per task
- Optional `None of these` choice
- Task order randomization
- Concept order randomization within task
- Auto-generated task design from attributes and levels
- Aggregate utility estimation across all completed responses
- Attribute importance calculation
- Choice share summaries
- Task-level raw response export

### Why this is the right MVP

- It matches the current structured-question pattern in Versive
- It is analytically meaningful without being overly ambitious
- It avoids the biggest implementation risks in adaptive designs
- It gives researchers something they can interpret immediately

## User Stories

### Researcher

- As a researcher, I can add a conjoint question in the study builder
- As a researcher, I can define attributes and levels
- As a researcher, I can choose how many concepts appear per task and how many tasks each respondent sees
- As a researcher, I can preview the generated tasks before launch
- As a researcher, I can see aggregate conjoint results on the insights page
- As a researcher, I can export raw task-level choices for external analysis if needed

### Participant

- As a participant, I can view one conjoint task at a time
- As a participant, I can select the option I prefer
- As a participant, I can complete the full conjoint question without confusion

### Analyst

- As an analyst, I can see utilities and importance scores
- As an analyst, I can inspect observed choice shares
- As an analyst, I can understand when the sample is too small or the model fit is weak

## Proposed UX

### Study Builder

The `conjoint` question should appear in the structured-question picker, likely under `Choices & Sorting` or a new `Tradeoffs` category.

Recommended builder sections:

1. Question prompt
- Example: `Which option would you choose?`

2. Design settings
- `conceptsPerTask`: `2` or `3`
- `tasksPerRespondent`: integer, default `8`
- `includeNoneOption`: boolean
- `randomizeTaskOrder`: boolean
- `randomizeConceptOrder`: boolean

3. Attributes and levels
- Researchers define a list of attributes
- Each attribute contains a label and a list of levels

4. Generated task preview
- Show the generated tasks
- Allow `regenerate design`
- Do not allow freeform manual editing of tasks in v1

Recommendation: for v1, the researcher edits the attribute/level schema and the system regenerates tasks from that schema. We should avoid hand-editing the generated task matrix at first because that increases QA complexity and can break design quality.

### Participant Experience

The participant should see one task at a time inside a single Versive question.

Each task should show:

- task progress, for example `Task 3 of 8`
- 2 or 3 concept cards
- attribute rows within each concept
- optional `None of these` option

Behavior:

- if the question is required, all tasks must be answered before submission
- if the question is optional, the participant can skip the full conjoint question
- the final submission should store all task answers as one structured response

This is important because the current Versive runtime is optimized around one answer payload per question. A multi-task conjoint block fits that architecture better than turning each task into a separate question.

### Results And Insights

Add a dedicated conjoint insight card on the study insights page.

Recommended tabs or sections:

1. Overview
- total completed responses
- total completed tasks
- concepts per task
- tasks per respondent
- whether `None of these` was enabled

2. Attribute importance
- horizontal bar chart of normalized importance percentages

3. Level utilities
- grouped chart or table of part-worth utilities by attribute level
- utilities should be normalized within attribute for readability

4. Observed choice shares
- overall choice share per level
- optional `None` share if enabled

5. Diagnostics
- warning for low sample size
- warning when estimation fails or design is rank-deficient
- show that results are aggregate-level estimates in v1

## Reporting And Export

V1 should support:

- inclusion in study insights UI
- CSV export of raw task-level responses
- report export with charts and summary tables

V1 does not need a full conjoint simulator in the report editor.

## Analysis Method

V1 should compute aggregate-level conjoint outputs on the backend.

Recommended method:

- use one-hot or effects coding for attribute levels
- fit an aggregate multinomial logit / conditional logit style model from task-level choices
- use `numpy` and `scipy`, which already exist in the backend environment

Derived outputs:

- part-worth utility per level
- normalized attribute importance:
  - importance of attribute `A` = range of utilities for `A` / sum of utility ranges across attributes
- observed choice shares by level

Fallback behavior:

- if estimation fails, still show descriptive task summaries
- if sample size is too small, show a warning and still compute descriptive shares

Recommended minimum guidance:

- show a caution below `50` complete conjoint responses
- prefer `100+` complete respondents for more stable utilities

## Data Model Proposal

### Question type

New frontend/backend type:

```ts
type Type =
  | ...
  | 'conjoint';
```

### Question props

Proposed `Question<'conjoint'>['props']`:

```ts
type ConjointAttribute = {
  id: string;
  label: string;
  levels: {
    id: string;
    label: string;
  }[];
};

type ConjointConcept = {
  id: string;
  levelsByAttributeId: Record<string, string>;
};

type ConjointTask = {
  id: string;
  concepts: ConjointConcept[];
};

type ConjointProps = {
  required: boolean;
  method: 'cbc';
  attributes: ConjointAttribute[];
  tasks: ConjointTask[];
  conceptsPerTask: 2 | 3;
  tasksPerRespondent: number;
  includeNoneOption: boolean;
  randomizeTaskOrder: boolean;
  randomizeConceptOrder: boolean;
  designSeed?: string | null;
};
```

Notes:

- Persist the generated tasks in `props`
- Do not regenerate the task matrix at runtime
- The persisted task matrix is the source of truth for rendering and analysis

### Answer payload

Store one user answer message with structured content:

```json
{
  "content": [
    "Task 1: Concept B",
    "Task 2: None",
    "Task 3: Concept A"
  ],
  "structuredContent": {
    "version": 1,
    "responses": [
      {
        "taskId": "task_1",
        "chosenConceptId": "concept_b",
        "noneSelected": false
      },
      {
        "taskId": "task_2",
        "chosenConceptId": null,
        "noneSelected": true
      }
    ]
  }
}
```

Optional later:

- response time per task
- displayed concept order per task

## Insight Contract Proposal

Add a new frontend insight type:

```ts
type ConjointInsights = {
  questionId: string;
  type: 'conjoint';
  data: {
    totalResponses: number;
    totalTasksCompleted: number;
    includeNoneOption: boolean;
    attributeImportance: {
      attributeId: string;
      label: string;
      importance: number;
    }[];
    utilities: {
      attributeId: string;
      attributeLabel: string;
      levels: {
        levelId: string;
        label: string;
        utility: number;
      }[];
    }[];
    observedShares: {
      attributeId: string;
      attributeLabel: string;
      levels: {
        levelId: string;
        label: string;
        chosenCount: number;
        chosenShare: number;
      }[];
    }[];
    diagnostics: {
      estimationSucceeded: boolean;
      warning?: string | null;
    };
  };
};
```

## Recommended Repo Touchpoints

### Frontend: question definition and builder

- `interviewer/types/questions.d.ts`
- `interviewer/services/question.ts`
- `interviewer/components/pages/study/contexts/questions.context.tsx`
- `interviewer/components/pages/study/create/question-editor.tsx`
- `interviewer/components/StudyCreate/AddQuestionDialog.tsx`
- new builder card component in `interviewer/components/StudyCreate/question-card/`

### Frontend: respondent runtime

- `interviewer/components/questions-types/`
- `interviewer/components/Questioner/components/Question.tsx`
- `interviewer/components/Questioner/components/Footer.tsx`

### Frontend: insights and exports

- `interviewer/services/insights.ts`
- `interviewer/components/StudyInsights/QuestionInsight.tsx`
- new insight component in `interviewer/components/StudyInsights/question-types/`
- `interviewer/utils/export-insights-pdf.ts`
- results CSV utilities as needed

### Backend: question config and study generation

- `versive_api/versive_api/app/schemas/questions.py`
- `versive_api/versive_api/app/services/questions.py`
- `versive_api/versive_api/app/services/studies/ai.py`
- `versive_api/versive_api/app/agent/`

Recommendation for v1:

- add the `conjoint` type to the core question model and manual UI
- do not let the AI study creator generate it yet
- if the agent touches a study containing a conjoint question, it should preserve it rather than trying to rewrite it

### Backend: insights

- `versive_api/versive_api/app/services/insights/structured.py`
- `versive_api/versive_api/app/services/insights/base.py`
- `versive_api/versive_api/app/api/v0/endpoints/insights.py`

## AI Simulation And Tagger Recommendation

### Simulation

Do not support conjoint in AI simulations for v1.

Add `conjoint` to the unsupported simulation list, similar to:

- `prototypeTask`
- `treeTest`
- `mediaUpload`

This avoids fake conjoint results from simplistic persona heuristics.

### Tagger

Do not add conjoint to the tagger system in v1.

The current tagger is optimized for extracting structured answers from simpler formats like:

- `multipleChoice`
- `ranking`
- `allocation`

Conjoint responses should be analyzed through the structured insight pipeline, not through tagging.

## Metrics

We should instrument:

- conjoint question created
- conjoint design regenerated
- conjoint question launched
- conjoint task completed
- conjoint question completed
- conjoint insight generated

We should also track:

- average completion rate by conjoint question
- average drop-off within conjoint question
- frequency of `None` selection when enabled

## Rollout Plan

### Phase 1

- Manual builder support
- Respondent runtime
- Raw response persistence
- Structured insights with aggregate utilities
- Basic export
- No AI creation
- No simulation

### Phase 2

- AI-assisted conjoint authoring
- Better design generation controls
- Report editor blocks for conjoint charts
- Choice simulator

### Phase 3

- Respondent-level utilities
- Advanced simulators
- Prohibitions
- Adaptive designs

## Risks

### 1. Poor task design leads to bad analytics

Mitigation:

- generate tasks automatically
- constrain recommended ranges for attributes and levels
- show warnings for underpowered designs

### 2. Researchers may over-interpret v1 utilities

Mitigation:

- label results as aggregate utilities
- show diagnostics and warnings
- document limitations

### 3. Runtime complexity inside a single question

Mitigation:

- keep the participant flow linear
- store one structured answer payload
- avoid branching logic inside conjoint blocks

### 4. Integration sprawl across question-type surfaces

Mitigation:

- keep v1 manual-only
- explicitly mark simulation unsupported
- explicitly leave tagger and AI authoring out of scope

## Acceptance Criteria

- Researchers can add a `conjoint` question from the study builder
- Researchers can define attributes and levels and generate tasks
- Participants can complete all tasks within one question
- Responses persist as structured content
- The study insights page shows attribute importance and utilities
- The analysis gracefully falls back when estimation fails
- CSV export includes task-level raw responses
- Simulation skips conjoint questions with a clear warning
- Existing studies and existing question types are unaffected

## Open Questions

- Should `None of these` be on by default or off by default?
- Do we want 2-concept tasks only in the first release, or 2 and 3?
- Do we need researcher-editable generated tasks in v1, or only a preview/regenerate loop?
- Should conjoint results appear in the report editor automatically in v1, or only on the insights page?
- Do we want a sample-size calculator in the builder in v1?

## Final Recommendation

Ship `conjoint` as a manual-only, choice-based conjoint question with generated tasks and aggregate conjoint analysis.

Do not start with adaptive conjoint, MaxDiff, AI authoring, or simulation support.

That scope is large enough to be useful, small enough to fit Versive's current structured-question architecture, and narrow enough to keep the analytics defensible.
