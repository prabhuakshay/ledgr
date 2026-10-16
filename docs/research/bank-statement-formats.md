# Indian bank and card statement exports: formats and parsing

Research for [#17](https://github.com/prabhuakshay/ledgr/issues/17). Researched 2026-10-16.
The samples from [#16](https://github.com/prabhuakshay/ledgr/issues/16) must check every row marked *unverified*.

## Short answer

- **Bank accounts** can usually be downloaded from netbanking as a spreadsheet (XLS, sometimes CSV or delimited text) and always as PDF. **Emailed statements are PDF only** and are password-protected.
- **Credit cards** are in practice **PDF only**. Every Indian card parser found reads PDFs; none reads a card CSV. The statement arrives by email, password-protected.
- **Amounts are almost never signed.** Bank exports use separate **Withdrawal/Debit** and **Deposit/Credit** columns, plus a running **Balance**. Card PDFs use one amount column with a **Cr / Dr / C / D** marker.
- **Dates are always day-first**, but the style varies by bank: `DD/MM/YY`, `DD/MM/YYYY`, `DD-MM-YY`, `DD Mon YYYY`, `DD Mon YY`.
- **Spreadsheets carry a preamble and a footer** (name, address, account, "End of Statement") around the header row. "Excel" exports are often legacy `.xls`, and SBI's is reported to be tab-separated text named `.xls`.
- **Library choice:** use pdfplumber for PDFs (MIT, active, takes a password, gives word coordinates), python-calamine for `.xls`/`.xlsx`, and the stdlib `csv` module for CSV. Avoid PyMuPDF (AGPL) and tabula-py (needs Java). camelot is optional and heavy.
- **Approach:** the usual answer is both. Spreadsheets and CSV suit a **column-mapping step**: find the header row, map columns to roles, choose a date format and a sign convention, and save it per bank. **PDFs need per-bank parsers.** Every working Indian PDF parser is per-bank code built on a shared base, and each one checks its balances.

## Per-bank and per-issuer table

Confidence: **official** = the bank's own site; **code** = a working open-source parser encodes it; **unverified** = third-party blogs only.

| Issuer | Download formats | Transaction columns (as exported) | Dates | Sign convention | Password |
|---|---|---|---|---|---|
| **HDFC Bank account** | PDF, Excel, Text, Delimited, MSMoney. Email is PDF only (official [1]) | Excel/`.xls`: `Date, Narration, Chq./Ref.No., Value Dt, Withdrawal Amt., Deposit Amt., Closing Balance`. The header is preceded by an address block and followed by a row of `****`; the footer reads `End Of Statement` (code [6][7]). Delimited: `Date, Narration, Value Dat, Debit Amount, Credit Amount, Chq/Ref Number, Closing Balance` (code [8]). PDF: same columns; `0.00` on the unused side (code [4]) | `DD/MM/YY` in XLS/CSV (code [6][7]); `DD/MM/YYYY` in PDF (code [4]) | Separate withdrawal and deposit columns | Account-statement PDF: numeric Customer ID (official [1][2]) |
| **HDFC credit card** | PDF by email (code [5]); any download option is unverified | Date (with time), narration, reward points, amount; supplementary-card sections (code [5]) | `DD/MM/YYYY` | Amount plus `Cr` marker for credits (code [5]) | First 4 letters of the name in capitals + DDMM of birth, or + last 4 card digits. Sources conflict (unverified [9][13]) |
| **ICICI Bank account** | PDF, XLS (unverified [3]); CSV implied by the Firefly config (code [8]) | XLS/CSV: `S No., Value Date, Transaction Date, Cheque Number, Transaction Remarks, Withdrawal Amount (INR), Deposit Amount (INR), Balance (INR)` (code [7]). The file has rows before the header and after the last transaction that must be stripped, and its first row is an opening-balance row (code [8]). The PDF is positioned text rather than a table, with a `B/F` opening row (code [4]) | `DD/MM/YYYY` or `DD-MM-YYYY` | Separate withdrawal and deposit columns | First 4 letters of the name + DDMM of birth. Letter case is disputed (unverified [10][13]) |
| **ICICI credit card** | PDF by email | Date, reference, narration, points, amount (code [5]) | `DD/MM/YYYY` | `CR` marker (code [5]) | As for the account (unverified) |
| **SBI account** | PDF, Excel; some sources also list CSV (unverified [11]) | E-statement PDF: `Date, Transaction Reference, Ref.No./Chq.No., Credit, Debit, Balance`, with `0` on the unused side (note: **Credit before Debit**). YONO PDF: `Txn Date, Value Date, Description, Chq/Ref, Debit, Credit, Balance`, with `-` on the unused side (code [4]). The "Excel" export is reported to be **tab-separated text with an `.xls` extension**, with the header at a varying row (code [12]) | `DD-MM-YY` (e-statement), `DD/MM/YYYY` (YONO), `DD Mon YYYY` (XLS) | Separate debit and credit columns, in a different order per variant | Last 5 digits of mobile + DDMMYY of birth; other channels differ (unverified [11][13]) |
| **SBI Card** | PDF by email | Amount ending in a bare `C`/`D`; only the last 2 card digits shown (code [5]) | `DD Mon YY` as three tokens | `C` / `D` marker | DDMMYYYY of birth + last 4 card digits (unverified [13]) |
| **Axis Bank account** | PDF, and XLS/CSV per third-party guides (unverified [14]) | CSV: `Tran Date, CHQNO, PARTICULARS, DR, CR, BAL, SOL` (code [7]) | `DD-MM-YYYY` | Separate DR and CR columns | Unverified (guides say DOB) |
| **Axis credit card** | PDF; XLS/CSV per one guide (unverified [14]) | Amount + `Cr`/`Dr`; Flipkart variant adds a cashback column; card masked with `*` (code [5]) | `DD/MM/YYYY` | `Cr` / `Dr` marker | First 4 letters of the name in capitals + DDMM (unverified [7]) |
| **Kotak account** | PDF (code [4]); other formats unknown | `#, Date, Description, Chq/Ref. No., Withdrawal (Dr.), Deposit (Cr.), Balance`, with an opening-balance first row (code [4]) | `DD Mon YYYY` | Separate Dr and Cr columns | CRN (unverified; the official pages only explain how to find the CRN [15]) |
| **Amex India** | Unknown for India. Amex US offers CSV for recent activity and the last 6 statements, and PDF for older ones [16] | Unknown | Unknown | Unknown | Unknown |
| Also supported by the parsers [4][5] | IDFC FIRST, IndusInd, Union Bank, Slice; card side also HSBC, BoB, Yes, Equitas, Jupiter | IDFC: `Date and Time, Value Date, Transaction Details, Ref/Cheque No., Withdrawals (INR), Deposits (INR), Balance (INR)`, with the balance suffixed `CR` | `DD Mon YY` | | |

**The evidence is thin.** Banks do not publish their export layouts. The only official statements found are HDFC's format list and HDFC's password. Every column layout above comes from open-source code written against real files. That code is recent and small (5 to 34 stars), so treat it as a strong hint, not a spec. Layouts change without notice: one ICICI parser notes a layout change in July 2026 [4].

## Recurring quirks a parser must handle

- **Preamble and footer rows** around the header, in both XLS and CSV. The header row has to be found by matching known names such as `Date`, `Narration` and `Closing Balance` [6][7][8].
- **An opening-balance row** that looks like a transaction (ICICI `B/F`, Kotak `Opening Balance`, the first ICICI CSV row) [4][8].
- **An empty side shown as `0.00`, `0`, `-` or blank** [4].
- **Indian digit grouping** in amounts (`1,23,456.78`), sometimes with a trailing `Cr`/`Dr`, and balances suffixed `CR` (IDFC) [4][5].
- **Narrations that wrap across lines or cells.** In HDFC's PDF a whole page sits in one table cell [4].
- **Value date versus transaction date.** Both are present; the map must pick one.
- **Card PDFs:** supplementary-card sections, EMI conversion lines, reward columns, and summary pages with sample totals that look like transactions [5].
- **Reconciliation as the safety net.** All three serious parsers check that opening balance + credits − debits = closing balance (or the total due) and flag any non-zero difference [4][5][17].

## Parsing libraries

PyPI data as of 2026-10-16.

| Library | Use | Licence | Latest | Notes |
|---|---|---|---|---|
| **pdfplumber** | PDF text, words with coordinates, tables | MIT | 0.11.10 (2026-06) | `open(path, password=...)`; table strategies `lines` and `text`; no OCR, works best on machine-generated PDFs [18]. Built on pdfminer.six and pypdfium2. Used by every Indian parser found [4][5][7] |
| pdfminer.six | Low-level PDF text | MIT | 20260107 | Used underneath pdfplumber |
| pypdf | Decryption check, text | BSD-3 | 6.20.1 (2026-10) | AES needs the `crypto` extra (`cryptography`). cc-parser uses it to detect encryption [5] |
| pikepdf | Decryption (qpdf) | MPL-2.0 | 10.17.0 (2026-10) | burnrate tries candidate passwords with it [7]. Binary wheels |
| PyMuPDF | Fast text and blocks | **AGPL-3.0** or commercial | 1.28.2 (2026-08) | Used by [4][5] as an extra extractor. The AGPL is a problem for an MIT project that might ever be distributed. **Avoid** |
| camelot-py | Table extraction | MIT | 2.0.0 (2026-06) | lattice/stream/network/hybrid/ml flavours; pulls in OpenCV, pandas and pypdfium2. Heavy, and not used by any Indian parser found |
| tabula-py | Table extraction | MIT | 2.10.0 (2024-10) | Wraps tabula-java, so it **needs a JVM**. Last release two years ago. **Avoid** |
| monopoly-core | Statement-PDF parser framework | **AGPL-3.0** | 0.23.2 (2026-10) | Per-bank configs plus a "generic parser (caveat emptor)", totals safety check, `PDF_PASSWORDS` setting. No Indian banks. Needs poppler (`pdftotext`) [17]. Read it as a design reference; don't depend on it |
| **python-calamine** | `.xls`, `.xlsx`, `.xlsb`, `.ods` | MIT | 0.8.3 (2026-10) | One reader for every spreadsheet format. pandas also accepts it as `engine="calamine"` [19] |
| openpyxl | `.xlsx` only | MIT | 3.1.5 (2024-06) | Slow release cadence |
| xlrd | `.xls` only (2.x dropped xlsx) [20] | BSD | 2.0.2 (2025-06) | Maintenance mode |
| msoffcrypto-tool | Decrypting password-protected Office files | MIT | 6.0.0 (2026-01) | Only needed if a bank emails encrypted XLSX. None found so far |
| stdlib `csv` | CSV and delimited text | PSF | — | Needs `utf-8-sig` (BOM) and sniffing for tab-separated `.xls` files |
| ofxparse / ofxtools | OFX/QFX | MIT / GPL-3.0 | 2021 / 2026-06 | Only relevant if a bank offers OFX. HDFC's "MSMoney" option may be OFX (unverified) |

pandas is not needed for parsing. A Django app can read rows directly. pandas would only make sense if it were already a dependency.

## Per-bank parser vs column mapping

How established projects do it:

- **Column mapping, configured by the user:** Firefly III's data importer assigns a *role* to each CSV column (date, description, amount, a separate debit and credit amount pair, or a bank debit/credit indicator) and saves it as a reusable JSON configuration. A community repo holds per-bank configs, including India's HDFC and ICICI [8][21]. hledger's CSV *rules* files do the same declaratively: `skip`, `fields`, `date-format`, and `amount-in`/`amount-out` for two-column amounts [22].
- **Per-bank code plugins:** ofxstatement has one plugin package per bank [23]. Monopoly has one config class per bank plus a generic fallback [17].
- **Indian projects:** burnrate is hybrid. It has a per-bank CSV class that is really just a header-alias table plus date parsing, a generic CSV fallback, and per-bank PDF parsers [7]. fipro describes each bank as a declarative `BankLayout` (signature headers, column aliases, date parser) [12]. akhilnarang's bank and card parsers are pure per-bank PDF code on a shared word-line extractor, with reconciliation [4][5].

Trade-off for ledgr:

| | Column mapping (data) | Per-bank parser (code) |
|---|---|---|
| Fits | CSV, XLS, XLSX | PDF, especially card PDFs |
| A new bank costs | The user maps columns once in the UI and the mapping is saved | Writing and testing code against a real sample |
| Breaks when | Headers are renamed, which is easy to detect and re-map | Layouts or coordinates shift, which is harder to notice without reconciliation |
| Handles | Header detection, column roles, date format, sign convention (single signed, split debit/credit, or Dr/Cr marker), rows to skip | Wrapped narrations, multi-line cells, card sections, summary pages |

**Suggestion for #18:** make column mapping the main path for spreadsheet and CSV files. Ship preset mappings for the user's own banks, and auto-detect the header row and the bank from known header sets. Treat PDF as a separate, later path: one parser per issuer, starting with the user's credit cards, since cards have no spreadsheet export. Both paths must end in the same reconciliation check against the statement's balances or total due.

## Open questions the real samples (#16) must settle

1. Which banks and cards does the user actually hold? This table should be cut down to them.
2. For each bank account: what does the "Excel" download really contain (`.xls` BIFF, `.xlsx`, HTML, or tab-separated text)? What is the exact header text, how many preamble rows are there, and is there a footer?
3. Is any spreadsheet export password-protected, or only PDFs?
4. Do the password rules above hold for the user's statements? Should ledgr store the password per account (burnrate stores it encrypted), or ask on every import?
5. Does any card issuer offer a CSV or XLS of billed or unbilled transactions in netbanking? If so, the card path could avoid PDF parsing.
6. Amex India: which formats does the en-in site offer? Nothing India-specific was found.
7. What is in HDFC's "MSMoney" and "Delimited" downloads: OFX, and which delimiter?
8. Which date is the transaction date in each file (Txn vs Value), and does any export include a time?
9. Are any PDFs scanned images? No evidence of that was found, and all parsers assume a text layer.
10. Does the HDFC narration carry a stable reference (`Ref`, UPI RRN) that could be used for duplicate detection?

## Fog noticed

- RBI's **Account Aggregator** framework delivers account data as structured JSON/XML via ReBIT schemas, with no files to parse. It needs a licensed AA or FIU relationship, so it is probably out of scope for a self-hosted app, but nobody has decided that.
- **Narration parsing** (UPI/NEFT/IMPS channel, counterparty, reference) is substantial per-bank work in [4]. It feeds categorisation and payee rules, which is a separate question from file parsing.

## Sources

1. HDFC Bank, NetBanking FAQs: "Account statements are available in 5 different formats for Download- PDF, Excel, Text, Delimited and MSMoney… via email only in PDF format… The password is your Customer ID". https://www.hdfc.bank.in/need-help/net-banking-faqs
2. HDFC Bank, "Download account statement via NetBanking": "Use your numeric Customer ID as the password". https://www.hdfc.bank.in/blogs/digital-banking/download-account-statement-via-netbanking
3. Groww, ICICI statement download guide, PDF/XLS (third-party). https://groww.in/banking/icici-bank-mpassbook
4. akhilnarang/bank-statement-parser (MIT, pdfplumber + PyMuPDF + pypdf; HDFC, ICICI, IDFC, IndusInd, Kotak, SBI, Slice, Union Bank). Parser docstrings in `bank_statement_parser/parsers/*.py`. https://github.com/akhilnarang/bank-statement-parser
5. akhilnarang/cc-parser (MIT; 14 card issuers), `docs/parsing-notes.md` and `cc_parser/parsers/{axis,sbi,hdfc,icici}.py`. https://github.com/akhilnarang/cc-parser
6. khanfarhan10/bank-ledger-dashboard, `src/parsers/hdfc_excel_parser.py` (HDFC `.xls` layout read with xlrd). https://github.com/khanfarhan10/bank-ledger-dashboard
7. pratik-devx/burnrate (Apache-2.0), `backend/parsers/*_bank_csv.py`, `bank_csv_base.py`, `services/pdf_unlock.py`, `config.py`. https://github.com/pratik-devx/burnrate
8. firefly-iii/import-configurations, `in/hdfc/default.json` and `in/icici/{default.json,README.md}`. https://github.com/firefly-iii/import-configurations/tree/main/in
9. ClearTax, HDFC statement password (third-party). https://cleartax.in/s/hdfc-bank-statement-password
10. ClearTax, ICICI statement password (third-party). https://cleartax.in/s/icici-bank-statement-password
11. ClearTax, SBI statement password (third-party). https://cleartax.in/s/sbi-statement-password
12. suryanshsrivastava/fipro, `src/parsers/sbi.py` ("SBI statements are often tab-separated text with an .xls extension"). https://github.com/suryanshsrivastava/fipro
13. Adobe India, credit card statement password guide (third-party). https://www.adobe.com/in/acrobat/roc/blog/credit-card-statement-password-guide
14. Axis Bank blog (says PDF only); third-party guides claim XLS/CSV. https://www.axis.bank.in/blogs/digital-banking/how-to-check-bank-statement-online
15. Kotak help centre (CRN lookup, statement of account). https://www.kotak.com/en/help-center/loans--hl--pl--bl--lap--wc-/home-loan/account-information-and-statement/statement-of-account.html
16. American Express US, Download Transactions & Statements. https://www.americanexpress.com/us/customer-service/faq.download-export-transactions-software.html
17. benjamin-awd/monopoly (AGPL-3.0), README. https://github.com/benjamin-awd/monopoly
18. pdfplumber README. https://github.com/jsvine/pdfplumber
19. pandas `read_excel`, `engine` parameter. https://pandas.pydata.org/docs/reference/api/pandas.read_excel.html
20. xlrd README: "will no longer read anything other than .xls files". https://pypi.org/project/xlrd/
21. Firefly III docs, data importer roles (debit/credit columns, "Bank specific credit/debit indicator"). https://github.com/firefly-iii/docs/blob/main/docs/docs/references/data-importer/roles.md
22. hledger manual, CSV rules (`amount-in`/`amount-out`: "hledger assumes both CSV fields contain unsigned numbers"). https://hledger.org/1.50/hledger.html
23. kedder/ofxstatement README (per-bank plugins). https://github.com/kedder/ofxstatement
24. Licence and release data: PyPI JSON API for each package, fetched 2026-10-16.
