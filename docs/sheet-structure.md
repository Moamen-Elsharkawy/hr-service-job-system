# Google Sheet Structure

One Google Spreadsheet with two tabs. The spreadsheet is linked to the Google Form.

---

## Sheet 1: "استجابات الفورم" (Raw Form Responses)

Auto-created and auto-populated by Google Forms. **Do not edit manually.**
n8n reads from this sheet only — it never writes to it.

Column order matches the form field order. Google Forms inserts a "Timestamp" column as column A.

| Column | Header | Content |
|--------|--------|---------|
| A | Timestamp | Submission timestamp (auto, ISO format) |
| B | اسم الشركة | Company name |
| C | المسمى الوظيفي | Job title |
| D | المجال | Field/industry (dropdown value) |
| E | المدينة | City (dropdown value) |
| F | نوع الشغل | Job type (multiple choice value) |
| G | الراتب | Salary (optional, may be empty) |
| H | طريقة التواصل أو رابط التقديم | Contact method or apply link |
| I | وصف الوظيفة | Job description (optional, may be empty) |

> Note: n8n reads columns by header name, not letter. If the form is recreated or fields
> reordered, column letters may shift — this does not break the workflow as long as header
> text matches exactly.

---

## Sheet 2: "الوظايف المنظمة" (Organized Jobs)

Written by n8n. This is the source of truth for deduplication and the daily digest.
Header row in row 1. Data starts at row 2. n8n always appends — never updates or deletes.

| Column | Header | Type | Notes |
|--------|--------|------|-------|
| A | التاريخ | Date string YYYY-MM-DD | Date row was processed (Cairo timezone) |
| B | اسم الشركة | Text | Trimmed from form submission |
| C | المسمى الوظيفي | Text | Trimmed from form submission |
| D | المجال | Text | One of the 9 dropdown values |
| E | المدينة | Text | One of the 5 dropdown values |
| F | نوع الشغل | Text | One of the 4 multiple choice values |
| G | الراتب | Text | May be empty |
| H | التواصل | Text | Contact method or apply link |
| I | الوصف | Text | May be empty |
| J | الحالة | Text | "active" on insert — editable manually to expire a job |

---

## Google Form Fields

Create the form manually at forms.google.com with these exact fields:

| Field Label | Type | Required | Options |
|---|---|---|---|
| اسم الشركة | Short answer | Yes | - |
| المسمى الوظيفي | Short answer | Yes | - |
| المجال | Dropdown | Yes | IT/تقنية, محاسبة ومالية, مبيعات, هندسة, تسويق, موارد بشرية, طب وصحة, تعليم, أخرى |
| المدينة | Dropdown | Yes | القاهرة, الإسكندرية, الجيزة, المنصورة, أخرى |
| نوع الشغل | Multiple choice | Yes | دوام كامل, دوام جزئي, عن بعد, فريلانس |
| الراتب | Short answer | No | - |
| طريقة التواصل أو رابط التقديم | Short answer | Yes | - |
| وصف الوظيفة | Paragraph | No | - |

---

## Duplicate Detection Logic

Before appending, n8n reads all existing rows from "الوظايف المنظمة" and checks:

- `اسم الشركة` matches (case-insensitive, leading/trailing whitespace stripped)
- **AND** `المسمى الوظيفي` matches (case-insensitive, trimmed)

If both match → skip (do not append, do not error, do not notify).
If either differs → treat as new job and append.

This means:
- Same company, different job title → allowed (two rows)
- Different company, same job title → allowed (two rows)
- Same company + same title → blocked (duplicate)

---

## Telegram Digest Format

Single Telegram message in Arabic plain text. Sent at 8 PM Cairo daily.
Not sent if zero new jobs were added today.

```
وظايف النهارده - DD/MM/YYYY

IT/تقنية:
- اسم الشركة | المسمى الوظيفي | المدينة | نوع الشغل
- اسم الشركة | المسمى الوظيفي | المدينة | نوع الشغل | راتب: X

محاسبة ومالية:
- ...

المجموع: N وظيفة جديدة
```

Rules:
- Date shown as DD/MM/YYYY (e.g., 08/05/2026)
- Fields listed in fixed order: IT/تقنية, محاسبة ومالية, مبيعات, هندسة, تسويق, موارد بشرية, طب وصحة, تعليم, أخرى
- Fields with zero jobs today are omitted entirely
- Salary shown only when non-empty
- If no jobs today, no message is sent
