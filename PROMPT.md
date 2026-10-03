You are a senior .NET architect and DevExpress Reporting expert. Help me design and build a production-quality tool, step by step.

## Context
- A business application (ASP.NET Core backend, SQL Server, Angular frontend) that has used **Stimulsoft** reports for years. Templates are `.mrt` files in **JSON** format, stored either as a blob in a templates table (takes priority) or as files in a templates folder.
- We are adding **DevExpress Reporting 26.1** side by side (both engines stay forever; a global setting chooses one).
- Every report gets its data from a **SQL Server stored procedure**. Hard requirement: DevExpress reports must use **SqlDataSource bound directly to the SP** (StoredProcQuery). No per-report C# data classes.
- Arabic (RTL) is the main language; reports must also work in English.
- Screens open a report through a URL query string (`?fromDate=...&companyId=...&storeId=...`). The backend copies every query key into the report parameter **with the same name**, converted to that parameter's declared type.
- There are hundreds of templates across modules (accounting, inventory, point of sale, HR...). The tool must handle them in bulk, not one by one.

## Goal
Build **"ReportConverter"**: a tool that takes a Stimulsoft `.mrt` and produces a DevExpress `.repx` that looks identical and returns identical data. It then registers the report in the DB and **verifies it automatically**. Converting and verifying a whole module must be one command that runs unattended.

## A working prototype already exists (console app). Rebuild it properly. It does:
1. Parse the `.mrt` JSON: `Dictionary.DataSources` (SqlCommand, Columns, Parameters), `Dictionary.Relations`, and `Pages[].Components` (bands → StiText / StiImage, `ClientRectangle` in mm "x,y,w,h", Font ";10;Bold;", Border "All;r,g,b;width;Double;...", Brush/TextBrush "solid:r,g,b", HorAlignment/VertAlignment, TextOptions.RightToLeft, TextFormat (StiNumberFormatService DecimalDigits/GroupSeparator, StiCustomFormatService StringFormat), WordWrap).
2. Map every StiText to an XRLabel at the **same rectangle**: mm × 3.937 = 1/100 inch. Same page size, orientation and margins.
3. Translate expressions:
   - `{Common_Keywords.X}` / `{Reports_Keywords.X}` / `{Company_Info.X}` → literal text, read by executing the keyword SPs with `@lang='ar'`.
   - `{MainDs.Field}` → `[Field]`; template variables → `?param`.
   - `ToString(dateParam)` → `FormatString('{0:yyyy-MM-dd}', ?p)`; `X.ToString("fmt")` → `FormatString('{0:fmt}', X)`.
   - `Today` → `Today()`; arithmetic and `Sum()` kept as is.
   - `Arabic(x)` (Stimulsoft: print x with Arabic-Indic digits) must always be translated, or the cell renders EMPTY with no error. Follow the business rule: either nested `Replace(ToStr(x), '0', '٠') ... '9' → '٩'`, or (common in practice) keep Western digits in both languages and emit `ToStr(x)`.
   - `Convert.ToDateTime(x).ToString("fmt")` → `FormatString('{0:fmt}', x)`. Left as is, the cell renders EMPTY.
   - `Sum(DataBand3, Source.Field)` (Stimulsoft names the band it aggregates over) → `Sum([Field])`. Keeping the first argument turns it into an unknown `?DataBand3` parameter and the total prints EMPTY.
   - Mixed literal and expression text → `Concat(...)`.
   - `{PageNumber}/{TotalPageCount}` → XRPageInfo (NumberOfTotal).
4. Map bands:
   - PageHeader / PageFooter → one band each (several Stimulsoft page headers are **stacked** into one).
   - GroupHeader → GroupHeaderBand with GroupFields from `Condition` and SortDirection.
   - `StiHeaderBand` (the column-header row of the data band that follows it) → GroupHeaderBand with `RepeatEveryPage=true`, attached to that data band. With several header bands in a row, the first one is the outermost (highest Level).
   - Main DataBand → DetailBand (sort fields only if the field exists in the schema).
   - DataBand with a `DataRelationName` → **DetailReportBand**: second StoredProcQuery plus a SqlDataSource Relation, DataMember = "MasterQuery.RelationName".
   - DataBand on another data source **without** a relation (for example a bill print: header SP, items SP and payments SP, all by `@id`) → an **independent** DetailReportBand over its own StoredProcQuery, no relation.
   - Footer and free DataBands after the detail → ReportFooterBand (stacked).
5. Build the SqlDataSource from **XML** (SqlDataSource.LoadFromXml), not the C# object API, because Expression query parameters don't serialize from the object API:
   - Connection name "MS SQL" FromAppConfig.
   - StoredProcQuery with **every** SP parameter, read from `sys.parameters`.
   - Each query parameter is `Type="DevExpress.DataAccess.Expression"` with value `(System.Type)(?name)`, pointing to a hidden report parameter of the same name.
   - Include a ResultSchema built from the .mrt columns.
6. Logo:
   - A StiImage becomes the company logo at the same rectangle.
   - With no StiImage, put the logo top-right (20 mm high, keeping its aspect ratio). If header text collides there, use the widest free gap of at least 30 mm in the top strip.
7. Register the new template in the templates table (upsert, flagged as DevExpress), copying the screen-link columns from the Stimulsoft row. Use parameterized SQL so Arabic names stay intact.

## Hard-won rules (each must be an explicit, unit-tested rule)
1. The live template is the DB blob first, then the folder file. They can differ; always convert the live one.
2. DevExpress validates a StoredProcQuery against the SP's **full** signature. A missing parameter raises `StoredProcNotInSchemaValidationException` at runtime ("Query X failed to execute").
3. SP parameters that the Stimulsoft data source does NOT pass must reach the SP as **NULL**: a nullable report parameter (`AllowNull=true`, `Value=null`). Sending `''` filters everything out (we got 0 rows).
4. Never overwrite the Value of an Expression query parameter at runtime with a raw value. DevExpress then sends NULL silently. This was the root cause of "reports render 0 rows with no error".
5. Set GroupHeaderBand.Level **after** `Bands.Add`, because DevExpress re-levels bands on add. Outer group = higher level.
6. A label whose Text is literally `[Field]` ignores TextFormatString. Use an ExpressionBinding `FormatString('{0:N2}', [Field])`.
7. With no Stimulsoft format, print the raw value (don't trim zeros); Stimulsoft shows 1.00 as 1.00.
8. Summary and total cells: `CanShrink=false`, `CanGrow=false`, so all cells in a row keep the same height.
9. When editing .repx XML by hand: collection items must be sequential `Item1..ItemN`, with no whitespace nodes and unique `Ref`s. Otherwise DevExpress silently drops the later items.
10. `XRPageInfo` does not inherit from `XRLabel`; use the `XRControl` base for shared properties.
11. Stimulsoft "Left" alignment inside an RTL text box renders as right.
12. Arabic literals inside SQL migration scripts: use `NCHAR(code)+...` concatenation, never `N'...'`, because they get corrupted when run.
13. DevExpress swallows data errors (a report just renders empty). Register `DevExpress.XtraReports.Web.ClientControls.LoggerService` and send errors to the app's ILogger.
14. The DevExpress viewer's CSV/text export defaults to the server's ANSI code page (Windows-1256), so Arabic separators U+066B/U+066C and RTL marks become '?'. Force UTF-8 with a BOM, which also opens correctly in Excel.
15. Multi-row bands (a summary row above a detail row) must share the same atomic column-width units, so vertical borders line up exactly.
16. Cross-query lookups (`GetValue('Query.Field')`, `Sum([Field],'OtherQuery')`) render blank silently. Compute them server-side and inject them as a report Parameter instead.
17. Stimulsoft functions with no DevExpress equivalent must never be dropped silently. Each one becomes a named, logged "unsupported construct" with its band, component name and original text.
18. "Query X failed to execute" is often a **command timeout or server memory pressure**, not a conversion bug. Some SPs take 80+ seconds; the default ~30 s timeout kills them. Always read the inner exception. Make CommandTimeout configurable per report, and classify failures as `Timeout`, `SchemaMismatch`, `SqlError` or `ConversionBug` before reporting them.
19. Row order inside a group can differ: DevExpress re-sorts, while Stimulsoft keeps the SP order. When the template has no explicit sort, preserve the SP order (no SortFields).
20. Stimulsoft can have known bugs (for example a filter parameter that is never passed). Reproduce Stimulsoft behavior exactly by default and list each such case in the report. Never "fix" business behavior silently.
21. A `StiDateFormatService` with no pattern means Stimulsoft's **short date** (`{0:d}`), not the full date-time.
22. Company data in templates (name, tax number, address, phone, email, logo) must stay **live**: emit report parameters filled at render time from the company-info source, never literals read at conversion time. Otherwise every customer prints the development company's address.
23. Language:
    - Keyword SPs may disagree on the English code (`en` vs `en-us`). Read both and emit `Iif(StartsWith(Lower(?lang), 'en'), 'English', 'Arabic')`, so any English code works.
    - Arabic text typed directly into a template has no keyword behind it. Translate it through a small, explicit map, and report anything left untranslated.
    - A field printed only in Arabic (`NameAr`) switches to `NameEn` (falling back to Arabic when empty) only when the source has it, the template doesn't already print it, and no caption explicitly names the Arabic one (for example "Name (Arabic)").
    - Fix missing or misspelled keyword texts **in the layout**, so they stay editable in the Designer, instead of silently changing shared SPs.
24. Formatting culture: requests that default to an Arabic culture format numbers with `٫` and `٬` and put hidden RTL marks in dates. Render DevExpress reports with an explicit report culture (invariant number format, date patterns without marks) on the report routes only, without changing the UI culture.
25. Any batch tool that loads and re-saves layouts (for example a header rewriter) must load the assembly that defines the report's ObjectDataSource types. Otherwise DevExpress silently saves `<ObjectDataSource Name="x" />` with no type, method or parameters, and the report prints no rows with no error. After every batch, scan all stored layouts for an ObjectDataSource without a DataMember.
26. Designer "Save As" copies have a new name. Any server-side logic keyed by the report's name (query-key mapping such as `levelId` → `level`, server-computed parameters) silently skips the copy. Store the origin report inside the copy's layout (kept through copies of copies) and key that logic on the origin or on the parameters the layout declares.
27. Don't add an extra load/save cycle to every served layout just to set an option (for example the export encoding). Set it where the layout is already re-saved.

## A. Intelligence: make the tool think, not just translate
1. **Pre-flight analysis** before converting. Parse the template into an intermediate model (IR: pages → bands → components → expressions) and produce a **complexity score** plus a list of features used: groups, relations, sub-reports, charts, conditions, cross-band totals, custom functions. Each report is then classified as `Auto` (convert and verify unattended), `Assisted` (convert, but flag specific spots for a human) or `Manual` (explain why).
2. **Expression engine**: a real tokenizer and parser for Stimulsoft (C#-like) expressions into an AST, then an emitter to DevExpress Criteria Language. No regex chains. Unknown nodes raise rule 17. Unit-test it with a corpus of every distinct expression found across all templates. Add a `corpus` command that extracts that corpus and shows which expressions are unsupported, ranked by frequency, so the most valuable rule is always implemented first.
3. **Smart verification inputs**: the verifier must find parameter values that actually return data. Probe date ranges (current year, then previous years, then all-time) and the most common ids from the real tables. A report verified on 0 rows is marked `Unverified-NoData`, never `Passed`.
4. **Auto-pick the verification key**: choose the key column and numeric column per data band automatically (the most distinct text/code column, plus the first non-null numeric column), with an override in config.
5. **Self-diagnosis loop**: when verification fails, the tool explains why by category:
   - missing rows → filter or param problem;
   - wrong numbers → format or expression;
   - an empty column → an untranslated function;
   - a fault → timeout or schema.
   It then suggests the exact rule or config change. Optionally re-convert with the fix and re-verify (max 2 attempts), logging every attempt.
6. **Visual diff** against the original: render the first N pages in both Stimulsoft and DevExpress to PNG, then compare per page with SSIM plus a text-position diff (the same text within ±1.5 mm). Report the differences as a heatmap image in the HTML report.

## B. Speed: must not take long
1. Convert in parallel (`Parallel.ForEachAsync`, configurable degree). Verify with bounded concurrency (default 2), so the DB server isn't overloaded.
2. Cache per run: keyword SP results per language, `sys.parameters` per SP and the result schemas. Load each one once.
3. **Incremental**: store a hash of (template bytes + converter version + rule-set version). Skip unchanged reports unless `--force` is given.
4. Fail fast: a pre-flight schema check (SP exists, parameters match, the columns used exist in the SP result via `sp_describe_first_result_set`) runs before any rendering.
5. Time-box every stage (convert, render, export, SP re-run) with per-report timeouts. A hung report never blocks the batch; mark it `Timeout` and continue.
6. Verification of very large reports: compare the full SP result set by streaming it (no full in-memory load), and cap exports by page count with a sampled check, clearly labeled as sampled.

## C. Professional quality
1. Idempotent and safe:
   - `--dry-run` shows what would change.
   - Every DB write is in a transaction, with a backup of the previous template blob.
   - A `rollback <report>` command restores it.
2. Structured logging (Serilog-style, JSON plus console) with a correlation id per report.
3. Exit codes suitable for CI.
4. Never log or print connection strings or passwords.
5. A single **HTML dashboard** per run: one row per report with status, pages (Stimulsoft vs DevExpress), matched/missing rows, a visual-diff score, duration, unsupported constructs, and links to the generated files and diff images.
6. Plugin rules: each mapping rule is a class implementing `IConversionRule` with a name, a priority and tests, so a new Stimulsoft construct is supported by adding one class.
7. Versioning: the converter version is written into each .repx (Report.Tag or a comment), so you can always tell which tool version produced a file.

## D. CLI commands
- `list` — templates per module and screen, with their DevExpress status.
- `analyze <id|--module>` — the pre-flight IR, complexity score, features used and the Auto/Assisted/Manual class.
- `corpus` — the expression corpus and the unsupported-construct ranking.
- `dump <id>`.
- `convert <id|--module> [--force] [--dry-run]`.
- `register`.
- `rollback <report>`.
- `verify <report>`.
- `run-all --module X` — convert, register and verify, then produce the HTML/JSON dashboard.

## E. Verifier (strict, end-to-end)
1. Start a SQL Extended Events session and capture the exact SP call the DevExpress viewer makes.
2. Export the full document through the viewer (headless browser) to CSV/XLSX, forcing UTF-8.
3. Re-run the captured call through SqlClient (handle embedded newlines and invariant number formatting).
4. Check that **every SP row** appears in the export, using a key column plus a numeric column. Use **maximum bipartite matching per key**, not greedy matching, and numeric tolerance 0.006. Normalize Arabic digits, RTL marks, trailing minus signs and lost separators.
5. Verify every data band, not only the main one, including independent and related DetailReportBands.
6. Pass only with 0 missing rows, no fault, real data (rows > 0), a visual-diff score above the threshold, and the build time within a threshold.
7. Run every report in **both languages** and with the **filters the screens really send**, using the screens' own query-key names and casing (for example `collectorId` vs `CollectorId`, `lang=en-us`).
8. Filtered cases: run the SP again without the filter. Any value that exists only in the unfiltered result must not appear in the export (no leaked rows).
9. Totals: every `Sum([F])` must print the SP total, and it must appear once more than the detail rows that already print the same number. A bare "the number exists somewhere" check lets a wrong total pass.
10. Count fields printed in a group header once per group, not once per row. Restrict a related detail query's rows to the master rows before matching.
11. Learn the layout's display rules before comparing: zero rows hidden by a `Visible` expression, sign conventions (for example revenue shown sign-flipped), and absolute values on subtotal lines.
12. Header and footer: company name in the requested language, tax number, live address, "printed by", no page numbers, Western digits only, UTF-8 export.
13. **Negative controls:** deliberately corrupt a passing export (drop a row, change a total, an Arabic decimal separator, a label in the wrong language, a leaked row, a wrong company name, a page number) and require the checker to FAIL each one. A checker that misses a corruption is a bug in the checker.
14. Reports with no data in the test period: seed clearly marked test rows, verify, then delete them and confirm that nothing is left.
15. Designer "Save As": copy each report, render the copy with the same screen parameters, and require it to match the original cell for cell.

## Required deliverables
1. **Architecture**: solution layout (Core / Expressions / Rules / Verifier / Cli / optional Admin UI), main classes and interfaces, and the IR model.
2. **Config** (appsettings.json): connection string, logo path, keyword SP names, default language, DevExpress version, output folder, concurrency, timeouts, thresholds, and per-report overrides (key column, CommandTimeout, verify params).
3. **Tests**:
   - Unit tests per rule and for the expression parser.
   - Golden-file tests per converted report.
   - A CI job that reconverts and verifies all reports against a test DB.
4. **Packaging**: single-file exe and/or dotnet global tool, versioned with the DevExpress version.
5. **A known-limitations list**, for example row order inside a group, Stimulsoft conditions/styles, sub-reports, charts and cross-band totals, each with the planned handling.

## How to answer
- Start with the architecture, the IR model and a short phased implementation plan. Phase 1 must already convert and verify simple reports end-to-end. Then wait for my confirmation.
- Then give the code file by file (C# 13 / .NET 10), production quality, with no placeholders.
- Point out risks and anything you are unsure about, rather than guessing DevExpress API behavior. Mark assumptions clearly.
