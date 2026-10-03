# Stimulsoft → DevExpress Report Converter: Build Prompt

A ready-to-use prompt for an AI assistant (ChatGPT, Claude, …). It asks the assistant to design and build a production-quality tool that converts **Stimulsoft `.mrt` report templates into DevExpress `.repx` reports**. The converted report looks the same and returns the same data, and the tool verifies that automatically.

The prompt is based on a real migration of production reports in an Arabic (RTL) business application. A working prototype converted 19 reports and passed a strict, row-by-row check: tens of thousands of stored-procedure rows were each found in the exported DevExpress output.

## What is in it

- **Exact layout mapping.** Every `StiText` becomes an `XRLabel` at the same millimetre rectangle, with the same fonts, borders, colours, RTL alignment and number formats.
- **Expression translation.** Stimulsoft (C#-like) expressions are translated into the DevExpress Criteria Language, ideally through a real parser/AST.
- **Band mapping.** Groups, column headers, master-detail via relations, and independent detail bands.
- **Data from stored procedures directly.** `SqlDataSource` with `StoredProcQuery`, and no per-report C# code.
- **27 hard-won rules.** These are silent DevExpress pitfalls that produce empty reports, missing columns or wrong numbers with **no error**.
- **A strict end-to-end verifier:**
  - an SQL Extended Events trace of the real call;
  - a full export from the viewer;
  - bipartite row matching against the stored-procedure result;
  - both languages and the screens' real filters, with leaked-row and total checks;
  - negative controls that prove the checker catches deliberate corruptions;
  - Designer "Save As" copies compared to the original cell for cell;
  - an optional visual diff.
- **Speed and safety:**
  - parallel and incremental conversion;
  - time-boxing;
  - dry-run, transactions and rollback;
  - an HTML dashboard of the results.

## How to use

1. Open [`PROMPT.md`](PROMPT.md) and copy all of it.
2. Paste it into your AI assistant. For Arabic answers, add: `Answer in Arabic; keep code and identifiers in English`.
3. The assistant first answers with an architecture and a phased plan. Review it, confirm, and then get the code file by file.
4. Adapt the **Context** section to your system: where templates are stored, how screens pass parameters, and your keyword/company-info sources.

## Some of the pitfalls it covers

| Symptom | Real cause |
|---|---|
| Report renders 0 rows, no error | An Expression query parameter was overwritten at runtime, so DevExpress sent NULL |
| 0 rows on some filters | An optional SP parameter was sent as `''` instead of NULL |
| "Query X failed to execute" | Often a command timeout or memory pressure on the server, not a conversion bug |
| A cell is empty | An untranslated Stimulsoft function (for example `Arabic(x)`) |
| Group order is wrong | `GroupHeaderBand.Level` was set before `Bands.Add` |
| Arabic separators appear as `?` in CSV | The viewer exports CSV as Windows-1256 by default |
| A total row is empty | `Sum(DataBand3, Source.Field)` was kept as is; DevExpress needs `Sum([Field])` |
| From/To dates are empty | `Convert.ToDateTime(x).ToString("fmt")` was not translated |
| Every customer prints the same address | Company footer fields were frozen at conversion time instead of staying live |
| English report shows `135٫7` | The request culture was Arabic; the report needs its own formatting culture |
| An ObjectDataSource report suddenly prints no rows | A batch tool re-saved the layout without the assembly that defines its data type |
| A "Save As" copy shows the wrong level/company | Server logic was keyed by the report's name, so the copy (new name) skipped it |
| A checker passes a wrong total | It only checked that the number appears somewhere; negative controls catch this |

## Contributing

Found another Stimulsoft → DevExpress pitfall? Open an issue or a PR that adds it as a numbered rule, with the symptom and the fix.

## License

[MIT](LICENSE)
