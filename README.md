# StudyFlow AI — Smart PDF & Notes to Mock Test Generator

StudyFlow AI is a responsive, single-page study application that turns pasted notes or uploaded text files into an interactive five-question multiple-choice quiz. It runs in the browser and can be hosted directly on GitHub Pages without a build step.

The application combines a modern dark interface, an interactive study dashboard, difficulty selection, instant answer feedback, source-based explanations, final scoring, an OCR beta demonstration, and Starter/Pro pricing.

## Current capabilities

| Capability | Status |
| --- | --- |
| Paste study notes | Functional; sample biology notes included |
| Import UTF-8 `.txt` or `.md` files | Functional; local file selection or drag and drop |
| Generate five-question multiple-choice quizzes | Functional; JavaScript parsing heuristics |
| Easy, Medium, and Hard difficulty | Functional; progressively longer recall blanks |
| Immediate feedback, explanations, final score, and retakes | Functional |
| Upload scanned PDF | Local beta simulation with file validation |
| Pro Plan ($9/mo) | Informational modal; no checkout |
| Accounts, saved tests, OCR services, and billing | Planned |

“AI” is the product name. This version uses rule-based JavaScript rather than a language model or an external AI API.

## Architecture

### Frontend: HTML5, Tailwind CSS, and Vanilla JavaScript

- **HTML5** provides semantic page structure, the study form, quiz, pricing cards, and native modal dialog.
- **Tailwind CSS 3.4.17 via CDN** provides utilities without a build pipeline.
- **Embedded CSS** handles the complete app layout, dark theme, responsive behavior, focus states, and reduced-motion preference. The page remains usable if the CDN fails.
- **Vanilla JavaScript** reads files, parses notes, creates questions, manages quiz state, scores answers, and controls the OCR demonstration and pricing modal.
- **Static hosting** serves the complete application from `index.html`. No package manager or build tool is required.

Study content stays in browser memory. The application has no upload API, database, authentication, billing integration, analytics, or persistent storage. Refreshing the page clears imported notes and quiz progress and restores the sample. The Tailwind CDN receives the usual asset request; study content is never included in that request.

### Quiz generation pipeline

1. Normalize line endings and common Markdown syntax; omit fenced code and image syntax.
2. Segment sentences with `Intl.Segmenter` when available, with a regular-expression fallback. Separate lines can also serve as facts.
3. Deduplicate statements and keep facts with 6–90 words and at most 600 characters.
4. Rank factual statements using content-word density, length, and definition cues. Rank potential answers using term frequency and word length.
5. Select five distinct source statements and replace key terms or phrases with a blank.
6. Build four unique, shuffled options using the answer and phrases from other statements. Prefer distractors of matching length and exclude phrases already present in the question's source sentence.
7. Score each answer once, show the original source sentence as an explanation, and calculate the final percentage.

| Difficulty | Preferred recall blank |
| --- | --- |
| Easy | One word |
| Medium | Two-word phrase |
| Hard | Three-word phrase |

If no suitable phrase of the preferred length exists, the engine uses a shorter span. Difficulty measures recall length, rather than conceptual reasoning. Retakes reuse the test with reshuffled options; generating again can select different facts and spans.

The generator does not invent missing facts. It rejects input that cannot supply five usable questions and three distinct distractors per question. Clear definitions and complete factual sentences work better than isolated keywords, tables, code, or fragmented outlines. Questions ask for the wording in the notes; distractors are not semantically verified.

### Planned backend: Python FastAPI + Tesseract OCR + PostgreSQL

| Component | Intended responsibility |
| --- | --- |
| **Python FastAPI** | Validated uploads, OCR job submission/status, authentication, and payment webhook endpoints |
| **Tesseract OCR** | Extract text from rasterized scanned PDF pages |
| **PDF rasterizer and background workers** | Controlled page rendering, bounded OCR jobs, retries, cancellation, and progress reporting |
| **PostgreSQL** | Users, document/job metadata, tests, subscriptions, and processed webhook events |
| **Razorpay / Stripe** | Hosted subscription checkout and verified server-side billing events |

These services are planned and are not included in this static application. Deploy them separately from GitHub Pages. The intended flow is browser → FastAPI upload/job endpoint → bounded rasterization/OCR worker → extracted text → existing quiz generator, with PostgreSQL recording job and account state.

## Project structure

    index.html   Complete application with embedded CSS and JavaScript
    README.md    Project documentation
    PLANS.md     Implementation checklist

## Local setup

No npm installation, compilation, database, or container is required.

Open `index.html` in a current Chrome, Edge, Firefox, or Safari browser, or serve this directory with Python:

    python3 -m http.server 8000 --bind 127.0.0.1

Visit:

    http://localhost:8000

Internet access loads Tailwind from the CDN; the embedded CSS provides the app's layout and styling independently. A local static server is useful for checking the same delivery model as GitHub Pages.

## Deploy to GitHub Pages

1. Create a GitHub repository and commit `index.html`, `README.md`, and `PLANS.md` at its root.
2. Open **Settings → Pages** in the repository.
3. Under **Build and deployment**, select **Deploy from a branch**.
4. Choose the branch containing the files, such as `main`, and **`/ (root)`**.
5. Save and wait for GitHub Pages to publish the site.
6. Open the published URL and verify the study dashboard, file imports, and beta notices.

A project repository normally publishes at:

    https://YOUR_USERNAME.github.io/YOUR_REPOSITORY/

All page navigation uses local section anchors, so repository subpaths work without routing configuration. GitHub Pages cannot run FastAPI, Tesseract, or PostgreSQL. Backend and payment secrets belong on the backend, never in publicly served HTML or JavaScript.

## Using the app

1. Edit the sample notes, paste your own material, or select a UTF-8 `.txt`/`.md` file.
2. Provide at least five distinct facts, each with six or more words.
3. Choose **Easy**, **Medium**, or **Hard**.
4. Click **Generate Mock Test**.
5. Answer each question, read the instant explanation, and continue to the results.
6. Expand the result review entries, practice again, or edit the notes and generate a new test.

The quiz uses a snapshot of the notes at generation time. Editing or importing new material does not silently change a test in progress; a notice prompts you to generate again.

### Input limits

| Input | Limit / validation |
| --- | --- |
| Pasted or imported notes | 60,000 characters |
| Text / Markdown file | 256 KiB; `.txt` or `.md`; valid UTF-8; nonempty; no NUL bytes |
| PDF beta preview | 20 MiB; `.pdf`; `%PDF-` header check |

Invalid imports leave the existing notes intact. PDF header validation is a lightweight preview check, not complete PDF validation. The preview reads only the file header and metadata, does not render pages, and does not extract text.

### Scanned PDF beta

The **Upload Scanned PDF (OCR Beta)** tab displays this notice exactly:

> Beta: Large mobile PDF uploads may time out—backend FastAPI OCR worker optimization in progress.

After choosing a PDF, **Preview OCR processing** runs a staged local simulation: header check, page extraction preview, and OCR queue preview. Progress is explicitly labelled as simulated; cancellation is supported. The final state explains that OCR extraction is not connected and offers a return to pasted notes.

No PDF is uploaded, no real OCR job is started, and no extracted content is fabricated. The timeout notice describes the planned backend work, not a measured failure of a deployed worker.

### Pro plan preview

The **Pro Plan ($9/mo)** button opens a keyboard-accessible modal containing:

> Razorpay/Stripe webhook integration pending final testing.

The price and Pro features are planned. This version does not collect payment, initiate checkout, create subscriptions, or contact either provider.

## Current Engineering Roadmap & Known Issues

- [ ] **OCR timeout optimization:** Implement the FastAPI OCR worker and optimize large mobile PDF workloads with page limits, controlled rasterization resolution, bounded worker concurrency, asynchronous jobs, cancellation, and measurable timeouts. Define upload/job limits and benchmark representative scanned documents before enabling real uploads. The current frontend only simulates processing.
- [ ] **Razorpay webhook idempotency:** Verify signatures against the raw request body and uniquely record provider event IDs in PostgreSQL. Apply subscription changes and event records transactionally so duplicate deliveries and retries cannot process the same payment twice. Handle out-of-order events and complete equivalent Stripe verification and provider sandbox testing before checkout is enabled.
- [ ] **Automated Playwright E2E tests:** Add a maintained regression suite and CI for pasted notes, supported and invalid files, all difficulty levels, scoring, retakes, OCR cancellation/completion, and the pricing modal. Include keyboard navigation, modal focus, mobile widths, and safe rendering of untrusted notes.
- [ ] **Quiz quality:** Improve abbreviation handling, numerical facts, multilingual notes, Markdown tables, and plausible distractor selection. Heuristics are strongest on English prose. Current questions test recall of source wording and can contain awkward blanks.
- [ ] **Persistent study history:** Add authenticated accounts, saved tests, and clear document retention/deletion controls once the backend exists.
- [ ] **Production asset delivery:** Replace the Tailwind runtime CDN with a reviewed, pre-generated CSS asset if a future asset workflow is introduced. The current release keeps the requested no-build CDN approach and includes independent CSS styling.

OCR, billing, authentication, and persistence remain unavailable. Source explanations allow users to check generated questions against their material.

## Accessibility and content handling

The page includes semantic sections, labelled form controls, keyboard-operable tabs with arrow/Home/End navigation, visible focus states, live status messages, a native modal with Escape dismissal and focus restoration, mobile layouts, and reduced-motion support.

Imported notes and filenames are rendered using `textContent` or textarea values, never interpreted as HTML. Files are read locally with browser APIs. The app uses no application cookies or local storage.

## Verification

Before deployment or generator changes, verify that:

- Valid pasted notes and supported files create exactly five questions with four unique choices each.
- Empty, insufficient, oversized, or unsupported input produces a helpful error.
- Each answer is scored once and receives immediate feedback.
- Explanations match the source notes and final scores are correct, including 0/5 and 5/5.
- Retakes reset scoring and imported notes do not overwrite the current test.
- OCR remains clearly identified as a local simulation and supports cancellation.
- The Pro modal displays the required notice and closes with Escape.
- Forms, answer choices, tabs, and the modal support keyboard use at desktop and mobile widths.
- Untrusted imported content cannot execute HTML or JavaScript.

A maintained, automated Playwright regression suite is a roadmap item. One-off release verification does not replace that future suite.

Release verification used native headless Chromium through Playwright with `--no-sandbox`. It passed 90 randomized quiz generations across all difficulties, full 5/5 and 0/5 sessions, retakes, Markdown imports, empty/oversized file rejection, PDF header validation, OCR cancellation/completion, modal Escape/focus restoration, 320px and 390px overflow checks, and safe content rendering. No uncaught browser JavaScript errors were reported. The temporary verification script is not a shipped test suite.
