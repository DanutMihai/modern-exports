# Table_query validation — reference

Companion to `SIM_Data_Validation_DEMO.xlsx`. Now covers **PhoneNr, ICC_ID, IMEI, StartDate,
Status and SIM Type**. The full formulas are long, so copy them straight out of row 2 of the
demo workbook (click the cell, copy from the formula bar). What's below is the parts you'll
actually want to read and edit.

## Column map in the demo

| Col | Header | | Col | Header |
|---|---|---|---|---|
| C | PhoneNr | | V | PhoneClean *(helper)* |
| E | ICC_ID | | W | ICC_Check |
| G | StartDate | | X | IMEI_Check |
| P | Status | | Y | Date_Check |
| Q | SIM Type | | Z | Status_Check |
| T | IMEI | | AA | SIMType_Check |
| U | IsPhoneValid | | AB | RowErrors |

IMEI and IsPhoneValid swapped places on 2026-10-13, so the input columns now run A–T in one
block (ID, then IMEI) and the check columns U–AB follow. Hide the check columns if you like, the
formatting still works.

---

## The colour convention

Every check column returns `""` (empty input), `"OK"`, or a message. Messages beginning with the
literal `WARNING:` colour amber; anything else colours red. Keep that prefix and any rule you add
later colours itself.

- **Green** — OK
- **Red** — wrong, someone must fix it
- **Amber** — probably wrong, but not certainly

Amber exists so you don't send people to the wrong cell. A blank SIM_Country doesn't mean the
phone number is bad; a failed Luhn digit doesn't *prove* the ICCID is bad.

---

## Two reusable building blocks

Both check formulas are built from these. Substitute your cell for `x`.

**Every character is a digit** — a true per-character test, unlike the `ISNUMBER(x+0)` trick from
the first version (which would have accepted `1E5`):

```excel
SUMPRODUCT(--ISNUMBER(--MID(x,ROW(INDIRECT("1:"&LEN(x))),1)))=LEN(x)
```

**Luhn mod-10 checksum over the whole string, check digit included:**

```excel
MOD(SUMPRODUCT(
  MID(x,ROW(INDIRECT("1:"&LEN(x))),1)*1*(1+MOD(LEN(x)-ROW(INDIRECT("1:"&LEN(x))),2))
  -MOD(LEN(x)-ROW(INDIRECT("1:"&LEN(x))),2)*9*(MID(x,ROW(INDIRECT("1:"&LEN(x))),1)*1*2>9)
),10)=0
```

Written with arithmetic instead of `IF()` deliberately — that way it needs no Ctrl+Shift+Enter in
older Excel. Both blocks need `LEN(x)>0`, which the surrounding `IF(cell="","",…)` guarantees.

---

## Rules per field

**PhoneNr** (→ `IsPhoneValid`, via `PhoneClean`)
Strips `+ space nbsp - ( ) .`, converts a leading `00` to bare digits, then: digits only →
length 8–15 → must start with the SIM_Country dial code from `tblCountries`.
Amber when SIM_Country is blank or unknown — the number itself may be fine. The country problem
is reported separately as a red error on SIM_Country (see "SIM_Country must be in the list").

**ICC_ID** (→ `ICC_Check`)
Digits only → starts with `89` → length 18–22 → Luhn (amber).
Plus a hard error if the cell holds a *number* — see the warning below.

**IMEI** (→ `IMEI_Check`)
Digits only → exactly 15 → Luhn (amber).
The Luhn part is an addition you didn't ask for; an IMEI's 15th digit is also a Luhn check digit
and the formula already existed. Delete that branch if you don't want it.

**StartDate** (→ `Date_Check`)
Accepts a real Excel date *or* text in day-month-year form.
Text must be 10 characters, `dd-mm-yyyy`, with matching separators from `- / .`, day 1–31,
month 1–12, year 2000–2100, and a real calendar date (so `31-02-2026` is caught). `5-3-2026` is
rejected — pad it. `2026-03-15` gets its own message naming the problem.
Real dates go amber if outside 2000-01-01 … today+2 years, which is the usual fingerprint of a
day/month swap Excel already swallowed.

**Status** (→ `Status_Check`) and **SIM Type** (→ `SIMType_Check`)
Allowed values live on the Config sheet, `G2:G4` and `H2:H3`. Both columns get a dropdown.
Matching is case-sensitive via `EXACT()`. Right value / wrong capitalisation is amber and names
the correct spelling; a value not on the list at all is red. Short enough to paste:

```excel
=IF($P2="","",IF(SUMPRODUCT(--EXACT($P2,Config!$G$2:$G$4))>0,"OK",IF(NOT(ISNA(MATCH($P2,Config!$G$2:$G$4,0))),"WARNING: wrong capitalisation - should be """&INDEX(Config!$G$2:$G$4,MATCH($P2,Config!$G$2:$G$4,0))&""" exactly","Must be one of: Active, Deactivated, Suspended")))
```

**RowErrors** — every non-OK check in the row on one line, separated by ` // `. Filter it for
"is not equal to OK" to get your whole worklist. Plain concatenation, not `TEXTJOIN`, so it
works in Excel 2016.

---

## ⚠ Format ICC_ID as Text before any data goes in

This is the one that will silently corrupt your data. Excel keeps only **15 significant digits**,
so an 18–22 digit ICCID typed or pasted into a General cell is truncated on the spot:

```
8935201234567890123   →   8935201234567900000
```

No error, no warning, and the lost digits are unrecoverable. Row 35 of the demo shows it happening.
`ICC_Check` flags a numeric ICC_ID as a hard error, but by then the damage is done — the check can
only tell you the cell needs re-entering from source. Select column E → Number Format → **Text**,
first thing, before anyone types.

Same applies to IMEI (15 digits sits exactly on the boundary) and to PhoneNr.

## ⚠ And the StartDate equivalent

Once Excel has parsed an entry into a real date, the typing order is gone and no formula can
recover it. On an en-US locale, typing `03-04-2026` gives **March 4th**, not 3 April — and
`Date_Check` sees a perfectly ordinary date. Formatting StartDate as **Text** before entry is the
only real defence, and it also makes the strict `dd-mm-yyyy` check do all the work.

---

## Three Excel constraints that shaped the design

**Data Validation can't read the check columns.** When validation fires, the check column still
holds the result for the *previous* value — the sheet hasn't recalculated. So each DV rule carries
its own simplified copy of the logic. The DV rules in the demo are deliberately looser than the
check formulas; they're a courtesy popup, not the enforcement.

**Data Validation never fires on paste**, fill-down, or from a macro. Conditional formatting does,
on every recalculation. The colours are the real safety net.

**Structured references are rejected inside DV and conditional formatting.** `tblCountries[Name]`
is fine in a check formula but not in a rule, so the country rules go through two workbook names,
`CountryNames` (`=tblCountries[Name]`) and `CountryDialCodes` (`=tblCountries[DialCode]`). Those
grow with the table — add a country as a new row at the bottom of `tblCountries` and the dropdown,
the rules and the check formulas all pick it up. The Status / SIM Type rules still use the static
ranges `Config!$G$2:$G$4` and `Config!$H$2:$H$3`.

---

## Pasted data (updated 2026-10-13)

- **tblCountries** now holds every ISO 3166-1 country and territory plus Kosovo — 250 rows. The
  original 50 names are unchanged. `SIM_Country` must match a `Name` exactly, capitalisation aside;
  anything else is a red error (see "SIM_Country must be in the list").
- **The check columns (U–AB) are real calculated columns.** Rows pasted under the table, or
  added by a script, get every formula. Before, new rows got none and simply showed no errors.
- **Validation and conditional formatting cover the whole column**, not rows 2–2000.
- **PhoneNr, ICC_ID, StartDate and IMEI are Text for the whole column**, matching what the export
  script applies. A values-paste keeps every digit, leading zeros and the day-month order.
- **Paste as values** (Ctrl+Shift+V). A plain Ctrl+V from another workbook brings the source
  cell's format, validation and conditional formatting with it and overwrites the ones here.
- The amber / red / green **background fills** now render; previously only the font colour did.

## Required fields (added 2026-10-13)

**PhoneNr, SIM_Country, Provider, Plan, ICC_ID and Email** must be filled in on every row that is
in use. A row is "in use" as soon as any input cell (A–T) holds something, so completely
blank spare rows stay quiet:

```excel
SUMPRODUCT(LEN(Table_query[@[FirstName]:[IMEI]]))>0
```

(Changed 2026-11-26. It used to be `$A2:$T2`, but deleting every row of the table made Excel
rewrite that range as `$A1:$S3`, and empty rows then showed false "Required" errors. The
structured reference does not move. The conditional-formatting rules keep `$A2:$T2`, because
rules cannot use structured references, and they are not affected by the deletion.)

A missing required field is a red error:

- the empty cell itself turns red (a cell holding only spaces counts as missing);
- `RowErrors` lists it as `<field>: Required - this field is missing`;
- for PhoneNr and ICC_ID the same text appears in `IsPhoneValid` and `ICC_Check`.

No column was added — the rule lives in `IsPhoneValid`, `ICC_Check`, `RowErrors` and nine
conditional-formatting rules. The sample rows were given a Plan, Provider and Email so they still
show what they were written for; row 36 shows all six missing at once.

`ValidateAndMatchImport.ts` already rejects an empty PhoneNr or ICC_ID. It does **not** check
Provider, Plan or Email, and a blank SIM_Country is skipped as "another country" rather than
failed — so today the workbook is stricter than the import.

## SIM_Country must be in the list (added 2026-10-13)

A SIM_Country that is not a `Name` in `tblCountries` is now a red **error**, not a warning:

- the SIM_Country cell turns red and `RowErrors` gets `SIM_Country: "X" is not in the country
  list - pick it from the dropdown` (or the stray-spaces variant);
- the dropdown is a **stop** rule, so typing an unknown country is refused. A paste still gets
  through — that is what the red cell is for;
- `IsPhoneValid` stays amber (`WARNING: country code not checked ...`), because the number itself
  may be fine.

Why it matters: the import writes warning rows to SharePoint and only holds back errors.
`ValidateAndMatchImport.ts` was changed to match — an unknown country now goes to `errors`, so
the row fails instead of being imported with its country code unchecked. **The script in this
folder is updated; the copy inside the Power Automate "Run script" action is not until you paste
it in.** In practice this only fires when the country chosen for the run is itself missing from
`tblCountries`, because rows for any other country are skipped before validation.

All formulas use `INDEX`/`MATCH` not `XLOOKUP`, nested `IF` not `IFS`, and no `LET`, so everything
runs on Excel 2016 and 2019 as well as 365.

---

## Not implemented — decide whether you need it

- **Shared dial codes.** `+1` is the US, Canada and much of the Caribbean; `+7` is Russia and Kazakhstan. A US number validates cleanly as Canada.
- **Trunk zeros.** `+40 0712…` (dial code plus a national leading `0`) passes the dial-code test but isn't valid E.164.
- **Per-country phone length.** 8–15 is the global E.164 range; real national plans are narrower. Add MinLen/MaxLen columns to tblCountries.
- **ICC_ID ↔ SIM_Country consistency.** Digits 3–4 of an ICCID are the Mobile Country Code and could be cross-checked against SIM_Country. Needs an MCC column.
- **Cross-field rules.** e.g. eSIM rows arguably shouldn't carry an IMEI; a Deactivated row might be expected to have an end date.
- **Uniqueness.** No duplicate check on ICC_ID, IMEI, PhoneNr or ID. That one's a `COUNTIF` away — say the word.
