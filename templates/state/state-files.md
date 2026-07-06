# State file contracts

State files are pointers and status only. Task content lives in the plan document; file knowledge lives in domain memory. Never duplicate either into state.

`current-session.json`
```json
{ "startedAt": "", "command": "", "artifact": "", "stage": "", "classification": { "domains": [], "technicalAreas": [], "complexity": "", "greenfield": false } }
```

`current-spec.json`
```json
{ "artifact": "", "spec": "prompt-spec.md", "blockingQuestions": false }
```

`current-plan.json` — status maps only:
```json
{ "artifact": "", "tasks": { "T1": "done", "T2": "pending" }, "dependsOn": { "T2": ["T1"] }, "parallelizable": [["T2","T3"]], "nextTask": "T2" }
```

`changed-files.json` — current workflow only, resets with each new artifact:
```json
{ "artifact": "", "files": [{ "path": "", "change": "added|modified|deleted", "task": "T1" }] }
```

`execution-history.json`
```json
{ "lastSyncCommit": "", "workflows": [{ "artifact": "", "completedAt": "", "memorySynced": true }] }
```

`progress.json` — transient checkpoint for resumable long operations (/f67-init batches); delete when done.
