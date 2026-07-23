# RaaS to XSLT Generator

> **v1.1** · MIT License · By Srini 

A single-file, browser-based tool that transforms any Workday RaaS XML export into a production-ready XSLT 3.0 stylesheet in minutes — no server, no installation, no dependencies.

---

**Disclaimer:**  Is an internal tool built to accelerate development. It is not affiliated with, endorsed by, or sponsored by Workday, Inc. “Workday” is a registered trademark of Workday, Inc., used here solely to describe the platform our team works on. 

---

## What problem does this solve?

Building XSLT transformations for Workday EIB (Enterprise Interface Builder) integrations is time-consuming. A developer typically:

1. Downloads a sample XML from a Workday RaaS report
2. Manually writes XPath expressions for every field
3. Handles namespace prefixes, date formats, ID attribute selectors
4. Iterates through Saxon error messages

This tool collapses that 2–4 hour process into 5–10 minutes. Upload your XML, click the fields you want, pick your output format, download the XSLT.

---

## Quick start

```
1. Download HTML file
2. Open it in any modern browser (Chrome, Edge, Firefox, Safari)
3. Upload your Workday RaaS XML export
4. Map your fields
5. Download the XSLT
```

No install. No server. No internet required after the page loads (Google Fonts are optional — the tool falls back to system fonts if offline).

---
## Bugs Fixed 04/08/2026

1. saveConfig drops otherwiseMode, otherwiseField, otherwiseTransform, otherwiseTransformParam
The OTHERWISE dropdown supports three modes (Text / Field / XPath) with field selection and transform. These are in mkCol and used in XSLT generation — but none of them are in the cols.map(c => ({...})) in saveConfig. After a save/load, the OTHERWISE always resets to plain Text mode, losing any Field or XPath otherwise values.
2. Fixed-width column widths don't survive save/load
S.fixedWidths is keyed by col.id (an auto-incrementing integer). On load, S.nid++ generates completely new IDs — so the saved width map {3: 15, 4: 20} never matches the new IDs {7: ?, 8: ?}. All widths silently reset to the default of 20 characters.
3. scanAllEntries doesn't re-run findRepeatingGroups
When the user clicks "Scan All Entries", it discovers new fields from deeper in the file — but findRepeatingGroups only ran on the first 5 entries during parseXML. If a repeating group only appears after entry 5, it won't be in S.repeatingGroups, so child group expansion and aggregation mode detection both miss it.
4. Fixed / JSON / HTML / XML / SpreadsheetML XSLT builders ignore aggregation
Only buildDelimitedXSLT has an isGroupingActive() check. If the user sets up aggregation then switches to JSON or HTML Table format — the downloaded XSLT ignores all the agg roles and generates a flat per-row output. The JS preview correctly aggregates (via buildOutputWithAgg) but the XSLT is wrong.
5. Multiple sibling repeating groups — only first handled
firstGroupPath() always returns [...S.repeatingGroups][0]. If a report has both Dependents_group AND Deductions_group at the same level, only whichever was detected first gets used for flattening, aggregation, and XSLT generation. The second group's fields appear in the field picker but produce empty output.
6. Conditional logic: test fields on parent when inside child-key aggregation
In child-key mode (grouping by a child field), the XSLT for-each-group iterates child elements. Any conditional test referencing a parent field (e.g. Home_Address_State contains 'CA') generates string(wd:Home_Address_State) — which doesn't exist at the child element level. Needs string(../wd:Home_Address_State) same as fp2xpath does for value columns.
7. No visual indicator on Step 3 columns showing their agg role
Once you assign roles in Step 3.5, there's no feedback on the Step 3 column table itself. Users don't know which columns have roles without re-opening Step 3.5.
8. Download Output button produces non-aggregated data even when agg is active
buildOutput → buildOutputWithAgg handles the JS download correctly for CSV. But for other formats when switching back from CSV, the raw download may not route through buildOutputWithAgg consistently.
---

## Features

### Step 1 — Upload Workday XML
- Drag & drop or browse for any Workday RaaS XML (`Report_Data` / `Report_Entry` structure)
- Auto-detects the Workday namespace (`xmlns:wd`)
- Scans up to 200 entries to detect all available fields
- **Auto-detects repeating child groups** (e.g. `Dependents_group`, `Deductions_group`) and handles nested row expansion automatically — no configuration needed

### Step 2 — Detected Fields
- All leaf fields displayed as clickable chips
- Real-time search/filter for reports with many fields
- Fields already added to the mapping show a green ✓
- WID (internal Workday reference IDs) hidden by default but available in column dropdowns

### Step 3 — Output Column Mapping

Five column source types:

| Type | Description |
|---|---|
| **Field** | Direct field value with optional transform |
| **Fixed Value** | Static text output for every row |
| **Conditional (if/else)** | Multiple WHEN/OTHERWISE branches with bracket-grouped AND/OR conditions |
| **Concat** | Combine multiple fields with a separator |
| **XPath Expression** | Raw XSLT 3.0 XPath for advanced use cases |

**Transforms available on any field:**

- Case: `UPPERCASE`, `lowercase`, `Trim whitespace`
- Date formats: `MM/DD/YYYY`, `YYYY-MM-DD`, `YYYYMMDD`
- Date parts: `Year (YYYY)`, `Month (MM)`, `Day (DD)`, `Month Name`, `Year +/- N`
- DateTime: strip to date only, extract time, timezone offset
- Numbers: `0.00`, integer, absolute value, comma-formatted, custom pattern
- Arithmetic: add, subtract, multiply, divide a constant
- Strings: strip non-digits, regex replace, substring, pad
- Payroll: zoned decimal, boolean Y/N

**Conditional logic** supports nested bracket groups:
```
( A = "Active" OR A = "Leave" )  AND  ( B contains "CA" )
```
Each bracket group has its own AND/OR logic, and groups are joined by a separate operator between them.

### Step 3.5 — Summarise & Roll Up (optional)

Aggregate repeating child rows into one output row per person or group — similar to a GROUP BY operation.

Assign each column a role:

| Role | Result |
|---|---|
| **Identity** | Forms the grouping key — one output row per unique combination |
| **Total (SUM)** | Adds all values across the group |
| **Count rows** | Counts how many rows are in the group |
| **Average** | Computes the mean |
| **Lowest / Highest value** | MIN or MAX across the group |
| **First value** | Takes the first value from the group |
| **Join all values** | Concatenates all values with a configurable separator |

Roles auto-assign when you open this step: parent-level fields (Employee ID, Name) become Identity keys; child numeric fields (amounts, hours) become Totals.

Generates `xsl:for-each-group` in the XSLT — fully supported by Workday EIB's Saxon engine.

### Step 4 — Output Format

| Format | Use case |
|---|---|
| **CSV** | Comma-delimited with configurable separator and quoting |
| **Delimited Text** | Any custom delimiter (pipe, tab, semicolon, SOH, custom) |
| **Fixed-width** | Column-padded fixed-length output (configurable width per column) |
| **JSON** | Array of objects or array of arrays, with optional pretty-print |
| **HTML Table** | Styled HTML table, optionally as full HTML page |
| **XML Records** | Custom root/row element names |
| **Excel-ready CSV** | Comma-delimited `.csv` that Excel opens natively |

> **Note on Excel:** Workday EIB cannot generate native `.xlsx` files via XSLT. The Excel-ready CSV format generates a standard CSV — configure EIB with `Get Data → Alternate Output Format: CSV` and `Deliver → File Type: CSV` with a `.csv` filename. Excel opens CSV files natively.

### Step 5 — Live Preview
- Table view and raw output view
- Shows first 10 rows using the actual mapped column logic
- Updates in real time as you change column mappings or formats
- Aggregated preview when Summarise & Roll Up is active

### Step 6 — Generate & Download
- **Download XSLT (.xsl)** — the XSLT 3.0 stylesheet ready to upload to EIB
- **Download Output** — the transformed data directly (CSV, JSON, etc.)
- **XSLT Header Comments** — document your transformation: purpose, author, target system, Workday report name, version history notes
- Comments are embedded at the top of every generated XSLT file

---

## Save & Load Config

Column mappings, output format settings, aggregation roles, and XSLT comments can be saved as a JSON config file and reloaded later — against the same or a different XML file.

Config saves:
- All column mappings (source type, field path, transforms, conditional logic, concat parts)
- Aggregation roles and concat separators
- Output format and all format-specific options
- XSLT header comment fields
- Grouping state

---

## EIB Integration

The generated XSLT is designed for Workday EIB's **Transform** step:

1. In Workday: go to your Outbound EIB → **Transform** step → upload the `.xsl` file
2. Set the **Deliver** step file type to match your chosen output format
3. Schedule or launch the integration

### Namespace handling

The tool auto-detects the Workday report namespace (e.g. `urn:com.workday.report/CR_Active_Employees`) and correctly wires `xmlns:wd` in the generated stylesheet. You do not need to manually set any namespace values.

### Repeating child groups in EIB

When your report has a repeating child group (e.g. `Dependents_group`, `Deductions_group`), the tool generates a nested `xsl:for-each` structure:

```xml
<xsl:for-each select="wd:Report_Data/wd:Report_Entry">
  <xsl:for-each select="wd:Dependents_group">
    <!-- one output row per dependent, parent fields resolved via ../ -->
  </xsl:for-each>
</xsl:for-each>
```

Parent fields (Employee ID, Name) resolve correctly using `../wd:FieldName` — no manual XPath required.

### Aggregation in EIB

Aggregated output uses `xsl:for-each-group` which is supported in Workday EIB's Saxon-HE processor:

```xml
<xsl:for-each select="wd:Report_Data/wd:Report_Entry">
  <xsl:for-each-group select="wd:Deductions_group"
    group-by="concat(../wd:Employee_ID,'|§|',../wd:Name)">
    <xsl:value-of select="../wd:Employee_ID"/>
    <xsl:value-of select="sum(current-group()/wd:Deduction_Amount)"/>
  </xsl:for-each-group>
</xsl:for-each>
```

---

## Supported XML Structure

The tool expects standard Workday RaaS output:

```xml
<?xml version='1.0' encoding='UTF-8'?>
<wd:Report_Data xmlns:wd="urn:com.workday.report/YOUR_REPORT_NAME">
  <wd:Report_Entry>
    <wd:Employee_ID>21001</wd:Employee_ID>
    <wd:firstName>Logan</wd:firstName>
    <!-- ... -->
    <wd:Dependents_group>           <!-- repeating child group -->
      <wd:Dependent wd:Descriptor="Megan McNeil">
        <wd:ID wd:type="Dependent_ID">Megan_McNeil</wd:ID>
      </wd:Dependent>
    </wd:Dependents_group>
  </wd:Report_Entry>
</wd:Report_Data>
```

Works with any Workday module: HCM, Payroll, Benefits, Recruiting, Absence, Compensation, Finance.

---

## Browser Requirements

Any modern browser released after 2020:

- Chrome 90+
- Edge 90+
- Firefox 88+
- Safari 14+

No extensions or plugins required. Works fully offline after the initial page load.

---

## Technical Notes

- **XSLT version:** 3.0 (Saxon-HE compatible — the engine used by Workday EIB)
- **XPath functions used:** `string()`, `format-number()`, `format-date()`, `format-dateTime()`, `contains()`, `starts-with()`, `matches()`, `string-join()`, `sum()`, `min()`, `max()`, `count()`, `current-group()`
- **No external dependencies** — pure HTML, CSS, and vanilla JavaScript
- **Single file** — everything is self-contained in one `.html` file
- **Config format:** JSON (version 4), backward-compatible with v2 and v3 configs

---

## Limitations

- **Native Excel output** is not supported via XSLT in Workday EIB — use Excel-ready CSV instead (see note above)
- **Maximum scan depth:** The tool scans up to 200 entries for field detection. Use the "Scan All Entries" button to scan beyond that if your report has fields that only appear on some workers
- **Single repeating group per report:** The tool handles one level of repeating child group. Multiple sibling groups at the same level are detected but only the first is used for row expansion
- **XSLT 3.0 only:** The generated stylesheets use XSLT 3.0 features. They will not work with XSLT 1.0 processors

---

## Contributing

Pull requests welcome.

Key areas for contribution:
- Additional transforms (custom date formats, new string operations)
- Support for multiple simultaneous repeating groups
- Additional output formats
- Bug reports with sample XML attached

---

## License

MIT License — Copyright (c) 2026 Srini · Kelly Services · Workday Dev

Free to use, modify, and share. See license text at the top of either HTML file.
