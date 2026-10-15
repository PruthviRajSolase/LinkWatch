Gemma Exit

Gemma 4 Dependency Resilience Agent: a command-line tool that scans a Python repository's dependencies, asks Gemma 4 what to do with each one, tests the baseline and the proposed replacements in disposable copies, and writes an evidence-labelled report called a Dependency Exit Capsule.

Version 0.4.0 · Target repository used in development: LinkWatch

1. Problem and solution overview

Problem. Dependencies decay quietly. Packages become unmaintained, get pinned to old versions, conflict between manifests, or are used through private APIs that make them hard to replace. Finding CVEs does not answer the harder questions: which dependencies are fragile, what could replace them, and is there any evidence that a replacement works?

Solution. Gemma Exit builds a Dependency Exit Capsule (artifacts/capsule.json and artifacts/capsule.md). It first collects facts without running the project's code, then runs the project's own test suite to get a real baseline, and only then asks Gemma 4 for a per-dependency recommendation (retain, replace, remove or investigate) grounded in that evidence. Replacements are tried in disposable copies and compared with the baseline.

Every claim in the capsule carries one label, so a reader can see how it is known:

Label	Meaning
observed	read directly from the repository or a registry, or recorded by deterministic code
verified	an executed command whose outcome was recorded (this does not prove a migration is correct)
ai_recommendation	Gemma's judgement; advisory until confirmed by an executed run
unverified	a claim with no supporting evidence yet

Honesty rules enforced in code: only the two documented Gemma 4 model IDs can be called and Gemini or any other model is never substituted; a mock is always labelled (MOCK); a test is reported as passed only if it ran; API failures, blocked validation and inconclusive comparisons are reported as such and never filled in.

Observed on LinkWatch (static scan and executed baseline, not model output):

pydantic is >=2 in pyproject.toml:17 but ==1.10.13 in requirements.txt:5.
certifi is imported in linkwatch/checker.py but declared in no manifest.
setuptools is used through import pkg_resources.
The baseline test run fails on Python 3.13 with ImportError: cannot import name 'soft_unicode' from 'markupsafe' (Jinja2==2.11.3 with an unpinned MarkupSafe that resolves to 3.0.4). Result counts from the executed run: 1 failed, 9 passed.
2. Setup instructions

Written for Windows with PowerShell and VS Code. Validation needs Python 3.13 or 3.12 (check with py -0); Python 3.14 is rejected for validation by design.

powershell
# 1. Put the target repo next to this folder
git clone https://github.com/rishrek/LinkWatch ..\LinkWatch

# 2. A venv containing LinkWatch's own dependencies (used by the metadata scan)
py -3.13 -m venv ..\lw-venv
..\lw-venv\Scripts\python -m pip install -r ..\LinkWatch\requirements.txt -r ..\LinkWatch\requirements-dev.txt

# 3. A venv for Gemma Exit itself, then run its tests
py -3.13 -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -m pytest -q

# 4. Create your settings file
copy .env.example .env

Open .env and set:

GEMINI_API_KEY=your_key_here
# optional
GEMMA_MODEL=gemma-4-31b-it
GITHUB_TOKEN=your_token_here

Create the API key in Google AI Studio. Never commit or share .env; it is git-ignored. GITHUB_TOKEN is optional and raises the GitHub rate limit used by the metadata scan from 60 to a higher number of requests per hour.

Expected test result on Windows: 62 passed, 1 skipped (the skipped test is a POSIX-only symlink check). All tests are offline and use mocks; none calls the real API.

3. Usage instructions

Full pipeline (scan, metadata, baseline, Gemma analysis, candidate validation, capsule):

powershell
.venv\Scripts\python main.py run --repo ..\LinkWatch --metadata-python ..\lw-venv\Scripts\python.exe

A run takes several minutes: one Gemma call per dependency, plus a fresh virtual environment and full test run for the baseline and for each replace candidate. Output goes to artifacts\; open artifacts\capsule.md.

If some Gemma calls fail with transient 500 INTERNAL errors, ask again only for the failed or changed dependencies:

powershell
.venv\Scripts\python main.py run --repo ..\LinkWatch --metadata-python ..\lw-venv\Scripts\python.exe --resume
# optionally allow the other documented Gemma 4 model to answer after API failures
.venv\Scripts\python main.py run ... --resume --fallback-model gemma-4-26b-a4b-it
Commands
Command	What it does
inspect	static scan (and metadata scan if enabled); no model, no validation
analyse	scan, then Gemma analysis
validate	scan, baseline validation, and candidate validation if recommendations exist
capsule	scan, analysis, and write the capsule
run	everything above in order
Options
Option	Purpose
--repo PATH	repository to analyse (default ../LinkWatch)
--model	gemma-4-31b-it (default) or gemma-4-26b-a4b-it; also settable via GEMMA_MODEL
--python PATH	Python 3.12/3.13 for validation. Optional: it is auto-detected, including through the Windows py launcher. If given, use the real path with no < > brackets; the path and version are checked before anything else runs
--metadata-python PATH	python.exe of a venv that has the repo's dependencies installed; enables the PyPI/GitHub metadata scan
--usage-json PATH	optional usage-analysis JSON for the metadata scan
--skip-validation	do not run baseline or candidate validation
--resume	reuse successful answers from artifacts/recommendations.json only when the evidence sent to the model is identical
--fallback-model	explicit second Gemma 4 model, tried only when the primary model's API keeps failing; the model that answered is recorded
--timeout SECONDS	per-step timeout (default 600)
Outputs (artifacts/)

scan.json, metadata_scan.json, baseline.json, recommendations.json, capsule.json, capsule.md. If validation cannot run, the capsule says blocked; if there is no API key, it says the AI stage was not run.

4. Architecture
            +------------------+      +----------------------+
 repo  ---> | scanner.py       | ---> |                      |
 (read-     | static: manifests|      |                      |
  only)     | imports, conflicts      |   ScanResult         |
            +------------------+      |   (Dependency list)  |
            +------------------+      |                      |
            | enrichment.py -> | ---> |                      |
            | metadata_scanner |      +----------+-----------+
            | PyPI/GitHub facts|                 |
            +------------------+                 v
            +------------------+      +----------------------+
 temp copy> | validator.py     | ---> | baseline ValidationRun| (venv, pip, full pytest)
            +------------------+      +----------+-----------+
                                                 | failure signature + test counts
                                                 v
            +------------------+      +----------------------+
            | gemma_client.py  | <--- | analyzer.py          | one call per dependency,
            | google-genai     |      | strict JSON checks   | retries, resume, fallback
            +------------------+      +----------+-----------+
                                                 | replace recommendations
                                                 v
                                      +----------------------+
                                      | validator.py          | candidate runs in fresh
                                      | (candidates)          | copies, compared to baseline
                                      +----------+-----------+
                                                 v
                                      +----------------------+
                                      | capsule.py            | capsule.json + capsule.md
                                      +----------------------+
Module	Role
main.py	CLI, argument checks, progress output
src/models.py	shared data classes (Dependency, Recommendation, ValidationRun, ...)
src/scanner.py	static, read-only scan of requirements files, pyproject.toml (PEP 621 and Poetry), setup.cfg, setup.py literals, Pipfile and lock files; line references, constraint conflicts, import evidence
src/metadata_scanner.py	teammate's registry/GitHub scanner (PyPI, GitHub, pip check, installed metadata, undeclared imports); kept unmodified
src/enrichment.py	thin adapter: runs the metadata scanner read-only, merges its facts into the scan, records disagreements instead of overriding either side, summarises lookup failures
src/gemma_client.py	Gemma 4 access through google-genai; allows only the two documented model IDs; key from the environment only
src/analyzer.py	builds the evidence prompt, validates the JSON answer, retries, resume, optional fallback, rejects a package proposed as its own replacement
src/validator.py	disposable-copy validation with fixed command templates and timeouts; Python 3.12/3.13 only; baseline and candidates run the same commands
src/capsule.py	builds and writes the labelled capsule (JSON and Markdown)
schemas/evidence_schema.json	schema for evidence records
tests/	offline tests with a mocked model

Design decisions are recorded in DECISIONS.md.

5. Dependencies

Python packages for Gemma Exit (requirements.txt):

Package	Used for
google-genai >= 1.0	calling Gemma 4 through the Gemini API
packaging >= 23.0	version and requirement parsing (scanner and metadata scanner)
pytest >= 8.0	the project's own offline tests

Everything else is the Python standard library.

Environment requirements

Python 3.13 or 3.12 for validation (Windows py launcher supported). The tool itself runs in the venv you created in step 3.
git, to clone the target repository.
Network access to pypi.org, api.github.com and the Gemini API (generativelanguage.googleapis.com).
A Gemini API key (required for the AI stage) and optionally a GitHub token.

Not dependencies of Gemma Exit: the packages of the repository being analysed (LinkWatch's requirements.txt and requirements-dev.txt). They are installed into ..\lw-venv for the metadata scan and into throwaway virtual environments for validation.

6. AI model details
Field	Details
Model name	Gemma 4 (Google). Default model: gemma-4-31b-it. Optional second model: gemma-4-26b-a4b-it
Model version	The Gemma 4 generation, identified by the model ID sent to the API (gemma-4-31b-it or gemma-4-26b-a4b-it). No other model or version can be called: the client raises an error for any other ID. The Gemma 4 model card is at https://ai.google.dev/gemma/docs/core/model_card_4
Model source	Google (Gemma open models, https://ai.google.dev/gemma). Hosted by Google through the Gemini API; model weights are not downloaded or self-hosted by this project
Access method	Gemini API through the official google-genai Python SDK: genai.Client(api_key=...) and client.models.generate_content(...) with the model ID, a system instruction and temperature 0.2. The key is read only from the GEMINI_API_KEY environment variable or .env. Documented route: https://ai.google.dev/gemma/docs/core/gemma_on_gemini_api
License and usage terms	Google lists Gemma 4 under its Apache 2.0 "Gemma 4 license" (https://ai.google.dev/gemma/apache_2) alongside the Gemma Terms of Use (https://ai.google.dev/gemma/terms) and the Prohibited Use Policy (https://ai.google.dev/gemma/prohibited_use_policy); see also the Intended Use Statement (https://ai.google.dev/gemma/intended_use_statement). The Gemma documentation pages were checked on 2026-10-15 (page last updated 2026-07-02); confirm the current terms at those links before redistributing. Use through the Gemini API is also subject to the terms that apply to the Google AI Studio account that created the key
Role in the project	Gemma 4 is the analysis step. For each scanned dependency it receives structured evidence (declarations, usage with file and line, registry/GitHub metadata, the other dependencies, and the executed baseline's failure signature with test counts) and returns one JSON object: recommendation, confidence, rationale, replacement candidate and import name, migration steps, required tests, compatibility risks, evidence gaps and evidence references. Its output is labelled ai_recommendation and is never treated as proof. It cannot run commands or install packages; executed validation is done separately by deterministic code

How the model is constrained:

The evidence is passed as untrusted data, and the system prompt forbids inventing versions, compatibility guarantees or support statements; unknowns must be listed under evidence_gaps.
Output must be one JSON object. It is validated locally (allowed values, valid package names, replace requires a candidate, a package cannot replace itself). Invalid output gets one retry, then is reported as an error.
API errors are retried four times with backoff. Failures are written to the capsule as failures.
The model that produced each recommendation is stored (model_id) and listed in the capsule (models_used).
No thinking level or JSON-mode setting is configured, so API defaults apply.
7. Known limitations

About the model

The Gemma API returned intermittent 500 INTERNAL errors in our runs. Jinja2 failed repeatedly after retries while other dependencies succeeded; we have not identified why. Failed dependencies are reported as failed in the capsule. --resume and the explicit --fallback-model reduce but do not remove this.
Recommendations are advisory. The verified label means a command ran and its outcome was recorded, not that a migration is correct.
JSON is enforced by prompt and local validation only.
Model output can vary between runs.

About validation

Candidate validation swaps one requirements line (version dropped). It does not rewrite imports or bump versions, so "same package, newer version" changes (for example Jinja2 or pydantic) cannot yet be validated.
On LinkWatch the baseline itself fails (Jinja2/MarkupSafe). A candidate that leaves that failure in place produces the same failure, so such comparisons are reported INCONCLUSIVE rather than as evidence for or against the candidate.
Validation covers pytest-based repositories with requirements*.txt manifests; other manifest types are reported blocked rather than guessed.
Only Python 3.12 and 3.13 are used for validation.
Validation is slow by design (fresh environment per run).

About the scans

Static usage evidence misses dynamic imports, plugins and entry points; appears_unused always carries that caveat.
The metadata scan is only as accurate as the target venv: a dependency that is not installed there can be reported as unused (disagreements with the static scan are recorded in the capsule).
GitHub allows 60 unauthenticated requests per hour; set GITHUB_TOKEN for more. In an early run 14 of 15 lookups failed for a cause we did not confirm; a later run had none.
The teammate's metadata scanner has known issues that were not changed because it is their file: it follows a symlinked venv interpreter to the base Python on Linux/macOS (the adapter refuses such interpreters), hardcodes the source roots linkwatch and tests, aborts on any unparsable requirement line, and can write a cache into the scanned repository when caching is on (the adapter turns it off).

About testing and scope

Development and live testing used one target repository (LinkWatch) on Windows. The 63 automated tests are offline and use mocks.
Nothing is guaranteed about other repositories, languages or test runners.
Project files

main.py · src/ · tests/ · schemas/ · README.md · DEMO.md (demo script) · SUBMISSION.md (submission text) · DECISIONS.md · CHANGELOG.md · HANDOVER.md · PROJECT_STATE.md · TASKS.md

Security

The API key is read only from the environment or .env. Never paste it into chats, commits or screenshots. If a key was ever exposed, delete it in Google AI Studio and create a new one.
