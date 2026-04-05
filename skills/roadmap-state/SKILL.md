---
name: roadmap-state
description: Manages roadmap state files — initialization, phase transitions, step tracking, and schema validation
---

# Roadmap State Management

This skill manages the `roadmap/.state/<feature>.json` lifecycle: creation, phase transitions, step tracking, and validation.

## State File Location

- Roadmap document: `roadmap/<feature-slug>.md`
- State file: `roadmap/.state/<feature-slug>.json`
- Global coordination: `roadmap/.state/_global.json`

All paths are relative to the project root.

## State Schema

```json
{
  "version": "1.0",
  "feature": "<feature-slug>",
  "feature_title": "<Human Readable Feature Title>",
  "roadmap_file": "roadmap/<feature-slug>.md",
  "created_at": "<ISO 8601>",
  "updated_at": "<ISO 8601>",
  "current_phase": 1,
  "total_phases": 4,
  "phases": {
    "1": {
      "title": "Phase Title",
      "status": "not-started",
      "branch": "feature/<slug>-phase-1",
      "depends_on": [],
      "started_at": null,
      "completed_at": null,
      "steps": {
        "1": {
          "title": "Step description",
          "status": "not-started",
          "parallel": true,
          "exclusive": false,
          "conflicts_with": [],
          "files": ["path/to/file.py"],
          "assigned_session": null,
          "started_at": null,
          "completed_at": null
        }
      },
      "steps_total": 5,
      "steps_completed": 0
    }
  },
  "locks": {},
  "sessions": {},
  "last_activity": "<ISO 8601>"
}
```

### Step Status Values

| Status | Meaning |
|--------|---------|
| `not-started` | Step has not been picked up |
| `in-progress` | Step is locked by a session and being worked on |
| `complete` | Step finished successfully |
| `failed` | Step failed after agent pipeline execution |
| `blocked` | Step cannot proceed due to dependency |

### Phase Status Values

| Status | Meaning |
|--------|---------|
| `not-started` | No steps have been started |
| `in-progress` | At least one step is in-progress or complete, but not all |
| `complete` | All steps are complete |
| `blocked` | Phase depends on an incomplete phase |

## Operations

### Initialize State

When `/on-prepare` creates a new roadmap:

1. Generate `<feature-slug>` by slugifying the feature title (lowercase, hyphens, max 50 chars)
2. Create `roadmap/.state/<feature-slug>.json` with all phases set to `not-started`
3. Populate step entries from the roadmap document
4. Set `current_phase` to 1
5. Write the state file

### Read State

When any command reads state:

1. Read `roadmap/.state/<feature-slug>.json`
2. Validate the JSON parses correctly
3. Validate `version` field is `"1.0"`
4. Return the parsed state object

If the file does not exist or is invalid JSON, report an error and suggest running `/on-prepare` first.

### Identify Next Step

When `/on-continue` needs the next available step:

1. Read the state file
2. Get the current phase (by `current_phase` index)
3. If current phase status is `complete`, advance `current_phase` to the next non-complete phase
4. Within the current phase, iterate steps in order:
   a. Skip steps with status `complete`, `in-progress`, or `blocked`
   b. For each `not-started` step, check parallelism constraints:
      - If `exclusive: true` and any other step in the phase is `in-progress`, skip
      - If `conflicts_with` lists step IDs that are `in-progress`, skip
      - If step is `parallel: true` (and not exclusive/conflicting), it is eligible
   c. Return the first eligible step
5. If no eligible step exists in any phase, return null (nothing to do)

### Mark Step In-Progress

When `/on-continue` starts a step:

1. Set step status to `in-progress`
2. Set `assigned_session` to the current session ID
3. Set `started_at` to current timestamp
4. If this is the first step started in the phase, set phase status to `in-progress` and phase `started_at`
5. Write the state file

### Mark Step Complete

When `/on-continue` finishes a step:

1. Set step status to `complete`
2. Set `completed_at` to current timestamp
3. Increment phase `steps_completed`
4. Clear `assigned_session`
5. If `steps_completed` equals `steps_total`, set phase status to `complete` and `completed_at`
6. If all phases are complete, the feature is done
7. Write the state file

### Mark Step Failed

When a step's agent pipeline fails:

1. Set step status to `failed`
2. Clear `assigned_session`
3. Log the failure reason in the step
4. Write the state file

The step can be retried by a subsequent `/on-continue` invocation (it will be treated like `not-started`).

### Advance Phase

When all steps in a phase are complete:

1. Set phase status to `complete`
2. Set phase `completed_at`
3. Find the next phase where all `depends_on` phases are complete
4. Set `current_phase` to that phase index
5. Write the state file

### Update Timestamps

Every write to the state file must:

1. Update `updated_at` to current ISO 8601 timestamp
2. Update `last_activity` to current ISO 8601 timestamp

## Validation Rules

1. `version` must be `"1.0"`
2. `feature` must be a non-empty string matching the slug format
3. `roadmap_file` must point to an existing file
4. `current_phase` must reference a valid phase key
5. All step statuses must be one of the valid values
6. All phase statuses must be one of the valid values
7. `steps_completed` must equal the actual count of steps with status `complete`
8. `depends_on` must reference valid phase keys

## Slugify Algorithm

```
input: "User Management API v2"
1. lowercase: "user management api v2"
2. replace non-alphanumeric with hyphens: "user-management-api-v2"
3. collapse multiple hyphens: "user-management-api-v2"
4. trim leading/trailing hyphens: "user-management-api-v2"
5. truncate to 50 chars: "user-management-api-v2"
output: "user-management-api-v2"
```

## Feature Discovery

To find which features exist:

1. List all `.json` files in `roadmap/.state/` (excluding `_global.json`)
2. Each file corresponds to one feature
3. Read each to get feature title, current phase, and completion status
