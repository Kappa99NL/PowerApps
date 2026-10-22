# Declaratieformulier – Power Apps canvas app

A one-screen Power Apps canvas app for entering a declaration against a client invoice:

1. Enter a **postcode** and **house number** → **Zoek klant** looks up the matching **client number** (`Klantnummer`).
2. Search and select one of that client's **invoices** (`Factuurnummer`).
3. **Line 1:** a free-text summary, an amount, and a **BTW** switch.
4. **Line 2:** the fixed text **Administratiekosten**, an amount, and a **BTW** switch.
5. **Subtotal**, **BTW (21%)** and **Total**, which update as you type.
6. **Opslaan** writes the form to a separate table (`Declaraties`) and then clears the form.

```
┌──────────────────────────────────────────────────────────────────────┐
│ Declaratie invoeren                                                  │
│ [Postcode ] [Huisnr ] [Zoek klant]   Klantnummer: K1001              │
│ [ Zoek en selecteer een factuur                                  ▼ ] │
│ Omschrijving                         Bedrag (excl. BTW)   BTW 21%    │
│ [ Reiskosten klantbezoek          ]  [        125,00 ]    (●  ) BTW  │
│ [ Administratiekosten             ]  [         15,00 ]    (  ○) Geen │
│                                                 Subtotaal: € 140,00  │
│                                                 BTW (21%):  € 26,25  │
│                                                    Totaal: € 166,25  │
│                                                          [ Opslaan ] │
└──────────────────────────────────────────────────────────────────────┘
```

## Files

| File | Purpose |
|------|---------|
| `src/DeclaratieFormulier.pa.yaml` | All the screen's controls and formulas, ready to paste into Power Apps Studio |
| `data/Klanten.csv` | Sample client data (postcode + house number → client number) |
| `data/Facturen.csv` | Sample invoice data |

## 1. Create the data sources

The formulas assume three **SharePoint lists**. Dataverse tables or SQL also work if the table and column names match; see "Adapting" below.

### `Klanten` (read): client lookup
| Column | Type | Notes |
|--------|------|-------|
| `Postcode` | Single line of text | Store without spaces, uppercase: `1234AB` |
| `Huisnummer` | Single line of text | Text, so additions like `12A` work. Store without spaces, uppercase |
| `Klantnummer` | Single line of text | |

### `Facturen` (read): invoices
| Column | Type |
|--------|------|
| `Factuurnummer` | Single line of text |
| `Klantnummer` | Single line of text (must be the **same type** as in `Klanten`) |

### `Declaraties` (write): where submitted forms go
| Column | Type | Value written |
|--------|------|---------------|
| `Title` | Single line of text | Line 1 description |
| `Klantnummer` | Single line of text | Found client number |
| `Postcode` | Single line of text | |
| `Huisnummer` | Single line of text | |
| `Factuurnummer` | Single line of text | Selected invoice |
| `Bedrag` | Number | Line 1 amount (excl. BTW) |
| `BTW` | Yes/No | Line 1 BTW switch |
| `AdministratieOmschrijving` | Single line of text | Always `Administratiekosten` |
| `AdministratieBedrag` | Number | Line 2 amount (0 if left empty) |
| `AdministratieBTW` | Yes/No | Line 2 BTW switch |
| `Subtotaal` | Number | Bedrag + AdministratieBedrag |
| `BTWBedrag` | Number | 21% over the lines where BTW is on |
| `Totaal` | Number | Subtotaal + BTWBedrag |

The created date and the user who submitted the form are saved by SharePoint automatically (`Created` / `Created By`).

To get test data in, you can import the CSVs in `data/` with **SharePoint → New → List → From CSV**.

## 2. Build the app

1. Go to [make.powerapps.com](https://make.powerapps.com) → **Create** → **Blank canvas app** (tablet layout).
2. **Data** → **Add data** → SharePoint → add `Klanten`, `Facturen` and `Declaraties`.
3. In the **Tree view**, select `Screen1` and paste the code: right-click → **Paste code**, or press Ctrl+V, using the contents of `src/DeclaratieFormulier.pa.yaml`.
4. Press **F5** to test, then **Save** and **Publish**.

If Studio rejects a control version number while pasting (versions change over time), add one control of that type by hand, use **Copy code** on it to see the current version, and update the `Control:` line in the YAML.

## How it works

| Control | Key formula |
|---------|-------------|
| `btnZoekKlant.OnSelect` | Normalises the input (trims it, removes spaces, makes it uppercase), then `Set(gblKlant, LookUp(Klanten, Postcode = pc && Huisnummer = hn))`. Shows a warning if no client is found |
| `cmbFactuur.Items` | `Filter(Facturen, Klantnummer = gblKlant.Klantnummer)`: only invoices for the client that was found. Searchable on `Factuurnummer` |
| `lblTotaal.Text` | `b1 + b2 + 21%` on each line whose BTW switch is on, formatted as `€ 1.234,56` |
| `btnOpslaan.DisplayMode` | Enabled only when a client is found, an invoice is selected, and line 1 has a description and a valid amount |
| `btnOpslaan.OnSelect` | `Patch(Declaraties, Defaults(Declaraties), {...})`. If saving succeeds: success message, `Set(gblKlant, Blank())` and `Reset(...)` on every input. If it fails: error message, and the form keeps its values |

## Adapting

- **BTW rate:** `0.21` appears in `lblSubtotaal`/`lblBtwBedrag`/`lblTotaal` and `btnOpslaan.OnSelect`. Search for `0.21` to change it.
- **Other table or column names:** rename them in the formulas (`Klanten`, `Facturen`, `Declaraties` and the column names in the `Patch` record).
- **Dataverse instead of SharePoint:** use your description column in place of `Title`. Yes/No columns map to `BTW`/`AdministratieBTW` as-is.
- **Klantnummer as a number column:** use the same type in both `Klanten` and `Facturen`, otherwise the invoice filter returns nothing.
- **Show more invoice details in the dropdown:** e.g. `DisplayFields: =["Factuurnummer", "Omschrijving"]`.
- **Delegation:** all lookups compare a column directly with a value (`Postcode = pc`), so they can be delegated to SharePoint/Dataverse and keep working on large lists. Because of this, the stored postcodes and house numbers must use the same format as the normalised input (no spaces, uppercase).
