# Course Copilot (n8n)

A self-hosted **n8n** workflow that connects to my university's **Moodle** LMS through its official Web Services API, detects new assignments, picks the relevant lecture files, and emails me an **interactive study guide** (Word document) with page-level citations to the course slides.

Nothing is ever submitted. The output is a study aid: concepts, step-by-step reasoning, worked examples, self-check questions, and exactly where in the lectures each topic is covered.

```
Triggers ─► Settings ─► Get assignments ─► Clean up list ─► Pick new assignments ─► Loop (one assignment at a time)
   │
   └─ per assignment:
      Has a file? ─┬─ yes ─► Download assignment file ─┐
                   └─ no  ─► Instructions as file ────┴─► Get course files ─► List course files
      ─► Choose files (Gemini #1) ─► Split chosen ─► Download chosen ─► Bundle
      ─► Analyze document (Gemini #2) ─► Make HTML file ─► Google Doc ─► .docx ─► Email
      ─► Delete temp Doc ─► Mark as sent ─► next assignment
```

## Highlights

- **Moodle Web Services under Microsoft SSO**: the university login is Microsoft-only, so the classic username/password token endpoint doesn't work. The workflow uses a token obtained through Moodle's official mobile-app login hand-off (`admin/tool/mobile/launch.php`), the same mechanism the Moodle app uses. See [How the Moodle token works](#how-the-moodle-token-works).
- **Retrieve, then generate (lightweight RAG)**: Gemini #1 reads the assignment and a numbered list of the course's lecture PDFs and returns the 3–4 most relevant (`[9, 10, 12]`). Gemini #2 gets the assignment plus only those files and writes the guide, citing `file, page`.
- **Handles real-world assignment formats**: PDF attachments, questions pasted as **images** in the description, or **typed instructions only**.
- **Word output without a Word library**: Gemini writes HTML → Google Drive imports it as a Google Doc (`convert=true`) → exported as `.docx` with real heading styles, tables and superscripts.
- **Production-minded**: three triggers (startup / schedule / manual), per-assignment loop, "mark as sent" only after a successful email, retries on network/AI nodes, temp-file cleanup, and a separate **error-alert workflow** that emails the failing node and error.
- **Prompt guardrails**: never invent references, URLs or datasets (illustrations must be labeled `[EXAMPLE FORMAT ONLY]`), `[TO FILL]` for decisions only the student can make, explicit constants in proofs, no LaTeX.

## Model comparison (same homework, three asymptotic-notation proofs)

| | Gemini Flash-Lite | Gemini Flash |
|---|---|---|
| Explicit constants (c, n₀) | "sufficiently large n₀" | c = 1, n₀ = 8 with a numeric table |
| Correctness of thresholds | two false claims | all correct, verified numerically |
| Conceptual errors | "Ω = best case" | none |
| Citations | one lecture file | slides + instructor notes |

The picker (Gemini #1) stays on Flash-Lite; the study guide (Gemini #2) uses Flash. Both are configurable in **Settings**.

## Setup (Windows, ~30 minutes)

1. **Docker Desktop**: install, then in an empty folder save [`docker-compose.yml`](docker-compose.yml) and run
   ```powershell
   docker compose up -d
   ```
   Open http://localhost:5678 and create the local owner account.
2. **Import** `workflow/Error_Alert.json` and `workflow/Course_Copilot.json` (⋯ → Import from file).
3. **Create credentials** in n8n:
   | Credential | Type | Notes |
   |---|---|---|
   | Moodle API token | Query Auth, name `wstoken` | value = your Moodle token |
   | Moodle file token | Query Auth, name `token` | same value (file downloads expect `token`) |
   | Gemini | Google Gemini (PaLM) API | free key from aistudio.google.com, no billing needed |
   | Google Drive / Gmail | OAuth2 | your own Google Cloud project, Drive + Gmail APIs enabled, redirect URI `http://localhost:5678/rest/oauth2-credential/callback` |
4. **Edit the Settings node**: your email, Moodle URL, models, max lecture files.
5. In the main workflow: **⋯ → Settings → Error workflow → "Course Copilot – error alert"**, save, then **Publish** both workflows.

### How the Moodle token works

1. Check the site supports it (no login needed): `tool_mobile_get_public_config` should report `enablemobilewebservice: 1`.
2. While logged in to Moodle in your browser, open
   `https://<your-moodle>/admin/tool/mobile/launch.php?service=moodle_mobile_app&passport=12345&urlscheme=moodlemobile&confirmed=1`
3. Right-click "Click here if the app does not open automatically" → copy link. It looks like `moodlemobile://token=<base64>`.
4. Base64-decode it: `hash:::TOKEN:::privatetoken`. Use only the **middle** part. Never share it; anyone with it has app-level access to your account.

The token is opaque (not a JWT), scoped to the mobile web service, and can be revoked any time under Moodle → Preferences → Security keys → Reset. Note: the Moodle phone app uses the same service, so resetting it also logs the phone app out.

## Settings reference

| Field | Purpose |
|---|---|
| `email` | where study guides are sent |
| `moodleUrl` | base URL of your Moodle site |
| `pickerModel` / `studyModel` | Gemini models for file picking / writing the guide |
| `maxLectureFiles` | how many lecture PDFs are attached (cost/speed trade-off) |
| `alreadySentIds` | comma-separated assignment IDs to never send |
| `testOnlyAssignmentId` | reprocess one assignment during a manual test; leave empty normally |

"Sent" state is stored in the workflow's static data, which n8n saves during automatic (published) runs only.

## Known limitations

- Only **PDF** lecture files are used (no .pptx/.docx yet).
- Only the **first** file of an assignment is read.
- Very long assignments can hit the Gemini node's request time limit.
- Runs only while the host machine and Docker are on (a VPS would make it 24/7).
- Model output must be reviewed: check cited pages and every `[TO FILL]` / `[VERIFY]`.

## Roadmap

- Extract lecture text with page markers instead of sending full PDFs (fewer tokens, faster)
- Call Gemini through an HTTP Request node with a longer timeout for long assignments
- Split long assignments into one model call per question
- Convert .pptx/.docx lectures to PDF via Google Drive
- Merge the two Moodle credentials into a single Custom Auth credential

## Responsible use

This uses an API the university enabled for its mobile app, for the student's own account, at a low request rate (a few calls, three times a day). Check your institution's IT acceptable-use and AI policies. The output is a study guide meant to help you learn the material, not a submission.

## Docs

A 12-page explainer of the architecture (Moodle API, token flow, Docker, every node, AI pipeline, security and cost): [`docs/How-it-works.pdf`](docs/How-it-works.pdf).
