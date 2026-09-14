# Signal Notebook — learn Rails 8 by building a small data laboratory

This is a teaching plan, not an implemented application. Build a notebook in which users create mathematical experiments, enter or import numerical samples, display interactive plots, and download Excel workbooks and PNG/JPEG images. The lessons specify what to implement, why it belongs there, and how to check it; application source code is intentionally left to the learner.

The project is suitable for someone comfortable with basic programming and HTML who wants practical Rails experience. Familiarity with Ruby or JavaScript helps, but advanced mathematics is unnecessary until the optional Fourier lesson.

## 1. What you will build

The finished core application has five screens:

1. **Experiments:** searchable list with title, function type, sample count, and last update.
2. **Experiment editor:** create or edit a title, description, function type, and parameters.
3. **Experiment detail:** numerical table, plot, parameter controls, and download buttons.
4. **Excel import preview:** worksheet validation, row errors, and explicit confirmation.
5. **Import history:** original filename, status, row counts, and error details.

An experiment can contain imported measurements or generated samples. Start with `y = x²` and a sine wave. Add a two-frequency signal and its spectrum later. Excel files contain data and formulas only: do not read, create, or embed spreadsheet charts. Chart images are separate browser-generated downloads.

The main learning objective is to follow data from an HTML form through a Rails controller into the database, then back into HTML, JavaScript, a workbook, and an image.

## 2. Scope and tool choices

| Area | Choice | Teaching purpose |
| --- | --- | --- |
| Application | Rails 8.x, full-stack application | Routing, controllers, views, conventions |
| Database | SQLite initially | Migrations, constraints, associations, queries |
| HTML/CSS | ERB, partials, plain CSS | Understand server rendering before adding abstractions |
| Browser behavior | Turbo and Stimulus | Navigation, partial updates, focused JavaScript |
| JavaScript delivery | Importmap and Propshaft | Keep the first iteration simple |
| Excel input | `roo` | Read `.xlsx` rows and inspect formulas |
| Excel output | `caxlsx` | Create `.xlsx` data sheets and controlled formulas |
| Plotting | Chart.js | Numeric axes and canvas rendering |
| Interface animation | GSAP | Timelines, easing, lifecycle cleanup |
| Attachments | Active Storage | Retain the original uploaded workbook |
| Tests | Rails Minitest and system tests | Verify data boundaries and browser behavior |
| Later lessons | Active Job, Solid Queue, Rails authentication generator | Background work and ownership |

Use a supported Rails 8 release and a Ruby version compatible with that release. Record the exact versions in the project rather than treating “latest” as a reproducible version. The [Rails getting-started guide](https://guides.rubyonrails.org/v8.0/getting_started.html) provides a foundation; select documentation matching your installed Rails minor version.

Rails supports import maps and JavaScript bundling. Begin with import maps; add a bundler only if a chosen dependency genuinely requires it. [Rails JavaScript guide](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)

### Time budget

These are teaching estimates, including practice and debugging. A developer already familiar with Rails can compress the early lessons.

| Lesson | Hours | Deliverable |
| --- | ---: | --- |
| 0. Environment and orientation | 2 | Running application |
| 1. CRUD and request lifecycle | 3 | Experiment notebook |
| 2. Associations and generated data | 3 | Samples and mathematical presets |
| 3. Hotwire | 4 | Inline editing and partial updates |
| 4. JavaScript and plots | 3 | Responsive mathematical plots |
| 5. Excel import | 4 | Validated preview and atomic import |
| 6. Excel export and formulas | 4 | Downloadable workbooks |
| 7. PNG/JPEG export | 2 | Reliable image downloads |
| 8. GSAP | 2 | Small, accessible UI animations |
| 9. Integration and handoff | 3 | Tested demonstration |
| Optional: Fourier analysis | 4–6 | Frequency spectrum |
| Optional: jobs and ownership | 4–6 | Background processing and private notebooks |

Core total: approximately 30 hours. Optional extensions bring the total to 38–42 hours. Teach in two-hour sessions: brief reading, a small implementation task, demonstration, and review. Make one Git commit after each completed lesson.

## 3. Data design before implementation

Create these entities progressively; do not generate everything on day one.

| Entity | Suggested fields | Rules |
| --- | --- | --- |
| Experiment | title, description, kind, parameters JSON, x_unit, y_unit | Title required; kind allowlisted; parameters validated by kind |
| Sample | experiment reference, position integer, x float, y float, source_row integer, source_formula text | Required finite x/y; unique position within experiment; source fields optional |
| ImportBatch | experiment reference, status, row_count, errors JSON, confirmed_at | Has original workbook attachment; explicit lifecycle |

Use `kind` values such as `quadratic`, `sine`, `two_tone`, and `imported`. Avoid a column named `type`, which has a special meaning in Active Record inheritance.

An Experiment has many Samples and ImportBatches. Add foreign keys and a unique database index on `(experiment_id, position)`. Make sample ordering explicit. For this project, require strictly increasing x values; the Fourier extension additionally requires uniform spacing.

Store generated samples as a snapshot. Changing parameters does not silently rewrite them: provide a **Regenerate samples** action that validates and replaces the snapshot in one transaction. Mark unsaved slider previews clearly. Imported data remain authoritative until the user explicitly chooses a replacement operation.

Float columns are appropriate for this mathematical exercise. Discuss why accounting data would require a different precision decision.

### Proposed responsibilities

| Location | Responsibility |
| --- | --- |
| `app/models/` | Associations and persistent invariants |
| `app/controllers/` | Load records, accept permitted inputs, select responses |
| `app/views/experiments/` | Pages, forms, sample rows, plot container |
| `app/services/sample_generator.rb` | Deterministic mathematical sampling |
| `app/services/workbook_importer.rb` | Parse and validate without committing |
| `app/services/workbook_exporter.rb` | Build a new workbook from stored data |
| `app/javascript/controllers/` | Chart, preview, download, and animation behavior |

Service objects are ordinary Ruby objects introduced to separate substantial work; they are a project design choice, not a Rails requirement. Extract them when the corresponding lesson needs them.

## 4. Lesson 0 — environment and orientation

**Learn:** Bundler, Rails commands, the project directory, logs, and development workflow.

1. Install a compatible Ruby, Bundler, Rails 8.x, Git, and SQLite prerequisites. On Windows, keep Ruby, the project, and commands together inside WSL; open that folder through VSCode's WSL connection.
2. Record `ruby -v`, `bundle -v`, and `rails -v`. Use your version manager to pin Ruby for this folder.
3. Create the application with `rails new signal_notebook --database=sqlite3`. Check that the generated Gemfile selects Rails 8.x before continuing.
4. Enter the directory, run `bin/rails db:prepare`, and start `bin/rails server`.
5. Locate `config/routes.rb`, controllers, models, ERB views, `db/migrate`, JavaScript controllers, `config/importmap.rb`, and the development log.
6. Add a simple home page manually. Trace one browser request through the route, action, template, and log.
7. Commit the generated application and lockfile. Add the installed tool versions and launch instructions to your project notes.

**Done when:** the home page loads and the learner can identify where its URL, HTML, and CSS are defined.

**Explain back:** What does Bundler lock? What changes when you run a migration? Why use `bin/rails` inside this project?

## 5. Lesson 1 — CRUD, forms, and validation

**Learn:** REST, strong parameters, Active Record, ERB, partials, and validation errors.

1. Create Experiment and its migration with title, description, and kind first. Use one scaffold if helpful, but read every generated file.
2. Implement index, show, new, create, edit, update, and destroy. Inspect the routes with `bin/rails routes`.
3. Add title presence and length validation and allowed kinds. Permit only intended fields in the controller.
4. Extract a shared form partial and an experiment summary partial.
5. Render errors next to the form with an unsuccessful HTTP status suitable for Turbo, ordinarily 422. Redirect after successful writes; use 303 where appropriate after form submissions.
6. Add an empty state, success messages, and an intentional delete control. Deletion must use a mutation request, never GET.
7. Add a title search through query parameters. Use parameterized queries and allowlisted sorting rather than interpolating user input into SQL.

**Done when:** invalid submissions preserve input, valid edits survive refresh, and searching does not mutate data.

**Check:** create a blank title, edit an existing record, visit a nonexistent ID, delete a record, and inspect the requests in browser developer tools.

**Explain back:** How do model validation and permitted parameters solve different problems?

## 6. Lesson 2 — associations and mathematical samples

**Learn:** migrations, associations, transactions, enumerable operations, and independent Ruby calculations.

1. Add Sample and the remaining Experiment fields described above.
2. Display an ordered sample table. Add a basic form to enter a sample manually.
3. Implement a plain Ruby sample generator accepting a kind and validated parameters and returning x/y pairs.
4. Add a POST action for regenerating samples. Calculate and validate first; replace existing samples transactionally.
5. Add a summary containing sample count, minimum, maximum, and mean y.
6. Cap ordinary generated samples at 2,000 and reject invalid domains, zero sample rates, and non-finite numbers.

Use these reference experiments:

| Preset | Definition | Sampling | Expected result |
| --- | --- | --- | --- |
| Quadratic | `y = x²` | x from −5 to 5 inclusive, step 0.1 | 101 points; vertex (0,0); endpoints y=25 |
| Sine | `y = A sin(2πft + φ)` | A=1, f=2 Hz, φ=0 radians; 100 samples at 100 Hz | Two cycles; t runs 0 through 0.99 seconds |
| Two-tone, later | `y = sin(2π·5t) + 0.5 sin(2π·15t)` | 256 samples at 256 Hz | Components at 5 and 15 Hz |

Generate x from an integer index rather than repeated floating-point addition. For a time series use `t = n / sample_rate`, with `n = 0 ... N−1`; do not duplicate the endpoint of a periodic record.

**Done when:** regeneration is reproducible and an invalid request leaves the previous dataset intact.

**Check:** verify the quadratic count and a few known sine values with tolerances rather than exact float equality.

## 7. Lesson 3 — Hotwire in three separate steps

Turbo Drive handles navigation, Frames target regions, and Streams perform named DOM updates. Stimulus connects browser behavior to HTML. Learn their jobs separately. [Turbo handbook](https://turbo.hotwired.dev/handbook/introduction)

### A. Observe Turbo Drive

1. Navigate between experiments with the network panel open.
2. Observe document state and requests rather than assuming every visit reloads the page.
3. Verify browser Back and Forward, validation failures, and redirects.

### B. Add one Turbo Frame

1. Wrap the title/description editor in a stable frame ID.
2. Return a matching frame from the edit and failed-update responses.
3. Save or cancel and restore the read-only section.
4. Keep full-page rendering available for direct URL visits.

### C. Add a Turbo Stream response

1. Create a sample through a form.
2. Return stream actions that update the sample list, count, summary, and plot region from the same saved state.
3. Give targets stable IDs; use Rails record DOM IDs for repeated records.
4. Add the corresponding removal behavior for deleting a sample.
5. Keep an ordinary HTML redirect response as a fallback.

Start with stream responses to the submitting browser. Multi-user broadcasts and WebSockets are a later lesson, not a prerequisite.

**Done when:** one inline edit and one operation updating several regions work without duplicate rows or stale counts.

**Explain back:** When would a frame be sufficient? Why does a stream response not automatically require Action Cable?

## 8. Lesson 4 — JavaScript fundamentals and Chart.js

**Learn:** ES modules, objects, arrays, events, Stimulus targets/values, and cleanup.

1. Install/pin Chart.js through `bin/importmap pin chart.js/auto`. Inspect the generated pins and downloaded modules, including transitive dependencies. Verify the actual module imports in the browser.
2. Create a chart Stimulus controller with a canvas target and a value containing safely serialized x/y data. Use Rails escaping/JSON helpers; do not concatenate user text into inline JavaScript.
3. Build a line plot with a **linear numerical x axis** and explicit `{x, y}` points. A category axis would conceal unequal sample spacing.
4. Add axis labels, units, a meaningful title, and a legend. Keep the numerical table accessible alongside the canvas.
5. Create the Chart.js instance on connection and destroy it on disconnection. Keep the instance on the controller, not in a global variable.
6. Initially replace the entire chart region after saved changes so it reconnects using fresh server data. Optimize in-place updates only after that works.
7. Add amplitude and frequency inputs with Stimulus actions. Use `map`, `Math.sin`, `Number`, and finite-number checks for a local preview. Label it **Unsaved preview** and require an explicit save/regenerate action.
8. Compare the Ruby and JavaScript generators against the same sample cases. Rails remains authoritative for saved values.

Chart.js documents update, destroy, and image-export methods in its [API reference](https://www.chartjs.org/docs/latest/developers/api.html). Stimulus can reconnect controllers after DOM changes, so initialization and cleanup must tolerate repeated visits. [Lifecycle reference](https://stimulus.hotwired.dev/reference/lifecycle-callbacks)

**Done when:** quadratic spacing is correct, sliders preview a sine wave, and repeated navigation produces one chart and one set of handlers.

**Practice:** explain `map` versus `forEach`, a module versus a global script, and an event handler versus a server request.

## 9. Lesson 5 — Excel reading with preview and confirmation

**Learn:** multipart forms, Active Storage, external data validation, service boundaries, and transactions.

Add `roo` to the bundle. Roo reads `.xlsx` values and exposes formula text; it is the reader in this project. [Roo documentation](https://github.com/roo-rb/roo)

### Workbook contract

Accept `.xlsx` only in the core project. Create a tiny fixture manually in Excel or LibreOffice using a worksheet named `Samples`:

| A: x | B: y |
| ---: | ---: |
| -2 | 4 |
| -1 | 1 |
| 0 | 0 |
| 1 | 1 |
| 2 | 4 |

Row 1 contains headers. Normalize header whitespace and case, but reject missing or duplicate required headers. Ignore entirely empty rows; reject rows missing only x or y. Require numeric, finite values. Report physical spreadsheet row numbers, not the index of a filtered array.

### Implementation sequence

1. Install Active Storage if not already configured, migrate, and attach one original workbook to each ImportBatch.
2. Add an upload form using Rails form helpers. Start with a 5 MB compressed upload limit and a 2,000-data-row limit; also bound worksheet dimensions and decompressed content before processing an untrusted workbook.
3. Validate that the upload is a parseable workbook. A filename extension alone is insufficient. Handle unsupported/encrypted/corrupt files as readable errors.
4. Let the importer return candidate rows and errors without writing Samples.
5. Display the first 20 candidates, the full error list, and total accepted/rejected counts. Preview must not replace existing samples.
6. On **Confirm import**, re-read the retained attachment, validate again, and replace samples in one transaction. Explicitly label replacement in the UI.
7. Lock the ImportBatch while confirming; reject repeat confirmation after success so a double-click cannot import twice.
8. Set status to `imported` and confirmed_at inside the successful transaction. On failure preserve the previous samples and show the error.

**Done when:** a valid workbook produces five samples; a bad row prevents the whole replacement and identifies its worksheet row; an accidental second confirmation does nothing.

**Discuss:** extension validation versus content validation, temporary uploads versus retained attachments, and all-or-nothing versus partial imports.

## 10. Lesson 6 — Excel writing and formulas

**Learn:** binary responses, workbook structure, numerical results versus formula expressions, and a clear interoperability contract.

Use `caxlsx` to generate a new workbook. It writes Excel files; do not use it as an arbitrary existing-workbook editor or a calculation engine. [Caxlsx project](https://github.com/caxlsx/caxlsx)

### A. Values-only export first

1. Add a read-only workbook download action.
2. Build `Samples` with x/y headers and the current saved samples; preserve numerical cell types.
3. Add `Metadata` with title, kind, units, and generation parameters. Force user-written metadata to literal text.
4. Give numbers reasonable display precision without rounding the underlying saved values unnecessarily.
5. Send the workbook with the `.xlsx` MIME type, an attachment disposition, and a sanitized filename. Disable Turbo interception for the download link if necessary.
6. Re-import the export and compare x/y pairs within a documented floating-point tolerance.

### B. Formula exercise

Add a separate `Calculations` sheet. Retain `Samples` as numeric values so re-import does not depend on spreadsheet recalculation.

| Column | Meaning | Example in row 2 |
| --- | --- | --- |
| A | x or time | Numeric sample x |
| B | Saved y | Numeric sample y |
| C | Recomputed quadratic | `=A2^2` |
| D | Recomputed sine | `=SIN(2*PI()*2*A2)` |

Also add summary cells using `SUM`, `AVERAGE`, `MIN`, and `MAX` over the populated y range. Adapt row references to the actual dataset and handle an empty dataset explicitly.

1. Consult the pinned caxlsx release's formula handling. Enable formula interpretation only for trusted, application-generated formula cells; leave user text escaped/literal.
2. Open the workbook in Excel or LibreOffice and verify calculated values, then change one input to demonstrate recalculation.
3. Reopen the saved workbook with Roo and compare formula text and the available stored result.
4. Show formula provenance in the import preview. Store source_formula and source_row if desired; source_formula is descriptive text, not an instruction to execute.
5. In the basic importer, reject formula cells in required x/y columns with a clear message. In the optional formula-aware mode, accept a numeric cached result only with a visible notice that freshness cannot be established.

**Important distinction:** writing a formula does not calculate its result. Roo reads stored workbook data; it does not run Excel formulas. A cached result can be absent or stale. A newly generated formula workbook may need to be opened, recalculated, and saved in a spreadsheet application before a reader sees updated results. Never substitute zero for a missing result.

Do not evaluate uploaded expressions with Ruby `eval` or JavaScript `eval`. Do not reactivate imported formulas on export. Preserve their text as literal provenance; generate active formulas only from the project's own templates. General formula round-tripping would require preserving referenced sheets, positions, and dependencies and is outside this course.

**Done when:** values-only export round-trips, application-generated formulas recalculate in a spreadsheet application, and formula text/result differences are visible to the learner.

## 11. Lesson 7 — export plots as PNG and JPEG

**Learn:** canvas pixels, MIME types, asynchronous callbacks, Blob objects, object URLs, and download lifecycle.

1. Add **Download PNG** and **Download JPEG** buttons near the plot.
2. Start with Chart.js `toBase64Image` to understand a data URL; then implement downloads with canvas `toBlob` and a temporary object URL. Revoke the URL after the browser has initiated the download.
3. Specify `image/png` or `image/jpeg`; use JPEG quality around 0.92. Handle a null Blob as an error. [Canvas toBlob reference](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toBlob)
4. Export from a dedicated temporary canvas with a fixed size, for example 1600 × 900 pixels, and Chart.js devicePixelRatio 1. Do not confuse CSS size with actual output pixel dimensions.
5. Re-render the current selected dataset on that canvas with animation disabled. Include title, legend, and axis labels inside the chart configuration; external HTML headings will not appear in the canvas image.
6. Paint an explicit white background before encoding JPEG. For consistency, use white for PNG too unless transparency is an intentional option.
7. Await any required fonts and finish the chart render before capture. Disable the button while exporting and destroy the temporary chart afterward.
8. Include experiment name and chart type in the filename. If exporting a local unsaved preview, say so in the UI and filename.

The export is a browser operation; it does not require uploading the image to Rails. Avoid external images in the canvas, which can create cross-origin export restrictions.

**Done when:** both files open in an ordinary image viewer, are exactly 1600 × 900 pixels, have readable labels, and match the selected plotted data. Repeat after changing parameters and after Turbo navigation.

**Explain back:** Why does a CSS white background not necessarily become a white JPEG background? Why is exporting halfway through animation unreliable?

## 12. Lesson 8 — GSAP without tangled state

**Learn:** animation properties, duration, easing, timelines, scoping, and teardown.

1. Pin GSAP with `bin/importmap pin gsap` and verify its import in the browser.
2. Create a separate animation Stimulus controller. Keep numerical calculations and chart rendering in their own controller.
3. Animate an import success message with opacity and a small vertical translation over about 0.3 seconds.
4. Build a short timeline to reveal the summary and download controls. Use a small stagger only where it clarifies the order of results.
5. Scope animations to the controller element using `gsap.context`; call `revert()` during cleanup. [GSAP context documentation](https://gsap.com/docs/v3/GSAP/gsap.context()/)
6. Respect `prefers-reduced-motion`: show the final state immediately when enabled.
7. Before Turbo caches the page, reset transient animation styles and dispose of chart/animation state as needed; verify reconnect behavior rather than relying only on `DOMContentLoaded`.
8. Do not let GSAP animate an element that Turbo is simultaneously replacing. Trigger animation after the new content connects.

Use GSAP for interface feedback. Let Chart.js manage its own canvas, and disable chart animation for image export.

**Done when:** animation runs once per intended event, navigation does not accumulate effects, and reduced-motion users retain every feature.

## 13. Lesson 9 — integration, tests, and teaching assessment

Add focused tests while building, then use this lesson to connect the pieces. Prefer tests of observable outcomes over tests that merely repeat implementation details.

| Test level | Cases |
| --- | --- |
| Model | Invalid kind, missing title, missing sample values, duplicate positions |
| Generator | Quadratic count and vertex; sine reference values; invalid sample rate |
| Import service | Valid fixture, bad header, partial row, wrong cell type, missing formula cache |
| Transaction | Failed replacement leaves existing samples unchanged; repeated confirmation is harmless |
| Export | Workbook opens through Roo; x/y remain numeric; formulas appear only where intended |
| Request | Successful downloads, validation response, invalid record ID, HTML/stream responses |
| Browser | Inline edit, import preview, updated plot, repeated navigation, image downloads |

Run `bin/rails test` and `bin/rails test:system`. Inspect the browser console and Rails log during the final scenario. Formula-calculation verification must also use a spreadsheet application; rereading with Roo alone cannot prove recalculation.

### Final demonstration

1. Create a quadratic experiment and generate 101 samples.
2. Edit its title inline with a Turbo Frame.
3. Show a sample update affecting the table, count, and chart.
4. Import the five-row workbook, preview it, and confirm replacement.
5. Attempt a malformed import and show that saved data remain intact.
6. Export values to Excel and re-import them.
7. Open the formula export and demonstrate recalculation.
8. Generate a sine experiment, adjust an unsaved preview, then explicitly save it.
9. Download its PNG and JPEG and open them outside the browser.
10. Navigate away and back, then repeat an export with reduced motion enabled.

**Completion standard:** the learner can explain each request, identify the authoritative data, change a validation rule, add one parameter, and diagnose a deliberately broken frame ID or invalid workbook row.

## 14. Optional extension — Fourier frequency analysis

This is a separate mathematics lesson after the core Rails workflow works. Start with a direct discrete Fourier transform (DFT) for clarity; an FFT computes the same transform more efficiently and can be introduced afterward.

### Reference dataset

Use `N = 256`, sample rate `fs = 256 Hz`, and `t[n] = n/fs` for `n = 0…255`:

`y[n] = sin(2π·5·t[n]) + 0.5 sin(2π·15·t[n])`

For bin k, compute:

- `Re[k] = Σ y[n] cos(2πkn/N)`
- `Im[k] = −Σ y[n] sin(2πkn/N)`
- `frequency[k] = k·fs/N`
- `magnitude[k] = sqrt(Re[k]² + Im[k]²)`

For even N, display bins 0 through N/2. Divide magnitudes by N, then double interior bins only; do not double DC or Nyquist. The reference signal should produce amplitude peaks near 1 at 5 Hz and 0.5 at 15 Hz, within floating-point tolerance.

### Tasks

1. Implement the transform as an independent Ruby object with a small input cap, such as 1,024 samples, because direct DFT takes O(N²) work.
2. Validate uniform time spacing and infer or verify sample rate. Do not silently analyze irregular x values as regularly sampled time data.
3. Add a spectrum panel with a numerical frequency axis and amplitude axis; reuse the chart export lesson.
4. Label it **Single-sided amplitude spectrum**, not power spectral density.
5. Discuss frequency resolution `fs/N` and the Nyquist frequency `fs/2`.
6. Change 5 Hz to 5.5 Hz to demonstrate leakage. Introduce a Hann window afterward, with its amplitude normalization, rather than expecting a single-bin peak.
7. Optionally compare with a JavaScript implementation or a Web Worker. Keep normalization and fixtures identical.

A quadratic function `x²` and a square wave are different examples. An optional square-wave lesson can sum odd harmonics, show overshoot near discontinuities, and keep the highest generated harmonic below Nyquist.

**Done when:** the reference peaks appear at the expected frequencies and amplitudes, and the spectrum exports with the same fidelity as the time-domain plot.

## 15. Optional extension — background work and user ownership

### Background imports

Move import confirmation to Active Job with Solid Queue only after synchronous import is reliable. Pass the ImportBatch ID, not a temporary upload path. Retain the workbook with Active Storage; use queued, running, completed, and failed statuses. Configure and run the worker for the actual development environment—do not assume production queue configuration also runs locally.

Begin with polling a status frame. Add Turbo broadcasts later if desired. Preserve atomic sample replacement and idempotency when jobs retry; show failure status without discarding the original workbook or old samples.

### Authentication and authorization

Explore the Rails 8 authentication generator, inspect its generated code, and create initial users deliberately; do not assume it supplies a complete registration product. Add Experiment ownership and scope all experiment, sample, import, and export lookups through the signed-in user. Test that one user cannot read or mutate another user's records or retrieve their attached workbook.

## 16. Troubleshooting exercises

| Symptom | Inspect first |
| --- | --- |
| Turbo reports missing content | Frame IDs in request and response |
| A successful save leaves stale plot data | Stream targets and the saved dataset sent to the chart |
| Duplicate animations after Back | Stimulus reconnects and Turbo cache cleanup |
| Blank exported image | Render completion, canvas dimensions, export timing |
| Dark JPEG background | Pixel background painting, not CSS |
| Uneven x values look equally spaced | Category versus numerical x axis |
| Formula import is blank | Missing cached result, not automatically zero |
| Bad workbook partly replaces data | Validation order and transaction boundary |
| JavaScript import fails | Importmap pins, transitive modules, network requests |
| Spectrum amplitudes are doubled | DC/Nyquist exceptions and normalization |

## 17. Reading order

Read only the relevant sections before each lesson, then return to the documentation when debugging.

1. [Rails getting started](https://guides.rubyonrails.org/v8.0/getting_started.html) — lessons 0–1; use the version selector for your installed Rails release.
2. [Active Record associations](https://guides.rubyonrails.org/association_basics.html) and [validations](https://guides.rubyonrails.org/active_record_validations.html) — lesson 2.
3. [Turbo handbook](https://turbo.hotwired.dev/handbook/introduction) — Drive, Frames, then Streams for lesson 3.
4. [Stimulus handbook](https://stimulus.hotwired.dev/handbook/introduction) and [lifecycle callbacks](https://stimulus.hotwired.dev/reference/lifecycle-callbacks) — lessons 4 and 8.
5. [Rails JavaScript guide](https://guides.rubyonrails.org/working_with_javascript_in_rails.html) — dependency delivery.
6. [Chart.js documentation](https://www.chartjs.org/docs/latest/) and [API](https://www.chartjs.org/docs/latest/developers/api.html) — lessons 4 and 7.
7. [Active Storage guide](https://guides.rubyonrails.org/active_storage_overview.html), [Roo](https://github.com/roo-rb/roo), and [Caxlsx](https://github.com/caxlsx/caxlsx) — lessons 5–6.
8. [Canvas toBlob](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toBlob) — lesson 7.
9. [GSAP documentation](https://gsap.com/docs/v3/) and [context cleanup](https://gsap.com/docs/v3/GSAP/gsap.context()/) — lesson 8.
10. [Rails testing guide](https://guides.rubyonrails.org/testing.html) — throughout the course and lesson 9.

Documentation checked while preparing this plan on September 13, 2026. Pin dependencies when starting the project and consult their matching documentation if an API differs.
