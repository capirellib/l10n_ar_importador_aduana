# AGENTS.md — l10n_ar_importador_aduana

Single-module Odoo addon (v19.0.1.0.0) for importing Argentine customs
clearances (OM-1993 / DUA) from PDF into supplier invoices.

## Repo structure

```
l10n_ar_importador_aduana/          # Odoo addon directory
├── data/account_data.xml           # Creates accounts 509.71, 509.55
├── models/res_company.py           # Per-company account config fields
├── models/res_config_settings.py   # Exposes config in Settings UI
├── security/ir.model.access.csv    # CRUD for wizard model
├── views/res_config_settings_views.xml
├── wizards/importar_dua_wizard.py  # ★ ALL business logic (~930 lines)
├── wizards/importar_dua_wizard_views.xml
└── __manifest__.py
```

## Key facts

- **License**: OPL-1 (commercial $850 USD) — proprietary
- **Dependencies**: `base`, `account`, `l10n_ar` + Python `pdfplumber`
- **No tests**, no CI, no lockfiles, no dev tooling
- **No JS/OWL/controllers** — pure server-side XML + Python
- **Menu**: Contabilidad → Proveedores → Importar DUA desde PDF
- **Optional**: `l10n_ar_payment_bundle` (capirellib/account-payment) for payment
- **Migrated from 15.0**: `attrs` → inline expressions, `tree` → `list`,
  `account_payment_group` → `l10n_ar_payment_bundle` (is_main_payment)

## Architecture

1. User uploads PDF via `ImportarDUAWizard` (transient model)
2. `_extract_from_pdf()` (line 198) uses **pdfplumber** to extract plain text,
   then **regex** to parse: tributes (codes 010/011/061/500/415/424),
   reference number, date, exchange rate, IIBB provincial withholdings
3. `action_import_pdf()` (line 580) creates `account.move` (in_invoice, USD)
   with clean expense lines, then performs **surgical atomic adjustment**
   (`_ajustar_asiento_dua`, line 290) via `savepoint()` + `check_move_validity=False`
   to correct tax lines and guarantee Debit = Credit
4. PDF attached as `ir.attachment`; optional **l10n_ar_payment_bundle** creation
   (main payment with `is_main_payment=True` + linked payment)

## Regex-based PDF parsing

All extraction relies on the OM-1993 fixed layout — no OCR, no coordinates:
- Tributes: `\(\s*(\d{3})\s*\)\s+[\w\s\.]+?\s+P\s+([\d.,]+)`
- Reference: `(\d{2})\s+(\d{3})\s+(\w{4})\s+(\d{6})\s+(\w)\s+\d+ de \d+`
- Date: `(\d{2}/\d{2}/\d{4})\s+\d{2}\s+\d{3}`
- Rate: `Cotiz\s*=\s*([\d.,]+)`
- IIBB: `^(\d{3})\s+[\d,]+\s+([\d.,]+)\s+(\d{3})\s+-\s+`

## Account classification (enforced)

| Pattern        | Type                |
|----------------|---------------------|
| 1.1.4.04.xxx   | IVA Crédito Fiscal  |
| 1.1.4.05.xxx   | Percepción Ganancias|
| 1.1.4.02.xxx   | Percepción IIBB     |

Any tax_line outside these patterns raises `UserError`.
