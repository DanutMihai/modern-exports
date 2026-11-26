# Export inventar pe țară, fără Office Scripts: ghid pas cu pas

Template: `SIM_Inventory_Export_TEMPLATE.xlsx` (în acest folder).

> **Versiunea 2 (2026-11-26).** Țara se ia din **numele fișierului**. Flow-ul nu mai scrie nimic
> în Excel: fără conector Excel, fără pauză, fără fișier temporar. Prima versiune scria țara cu
> „Update a row”, dar fișierul era citit pentru atașament înainte ca modificarea să fie salvată,
> așa că atașamentul ajungea fără țară și tabelul rămânea gol.

## Cum funcționează

```
Power Apps: dropdown țară + buton Export
   │  ExportInventory.Run(țara)
   ▼
Power Automate (4 acțiuni standard)
   1. compune numele: SIM_Inventory_<Țara>_<data>_<ora>.xlsx
   2. citește template-ul
   3. îl trimite pe mail ca atașament, cu numele de la pasul 1
   4. răspunde aplicației
   ▼
Utilizatorul deschide atașamentul în Excel desktop
   → celula Export!B7 scoate țara din numele fișierului
   → Power Query descarcă din listă DOAR țara respectivă, cu contul utilizatorului
   → Table_query se umple; validările, formulele și culorile merg ca în template-ul de import
```

Fișierul trimis pe mail nu conține date. Datele vin la deschidere, cu drepturile utilizatorului.
Atașamentul are ~55 KB indiferent de mărimea țării.

**Numele fișierului nu se schimbă înainte de prima deschidere.** Excel poate adăuga singur
` (1)` la salvare, iar asta nu strică nimic. Dacă fișierul a fost redenumit, Excel afișează mesajul
*No country to download…*: scrii țara în **Export!B4** și dai Data → Refresh All.

---

## Pasul 0: o singură dată, în SharePoint

1. Pe site-ul `simri`, în **Documents**, creează folderul `ExportTemplate` și încarcă acolo
   `SIM_Inventory_Export_TEMPLATE.xlsx`. Dacă îl încărcasesei deja, înlocuiește-l cu versiunea
   nouă.
2. Verifică faptul că `SIM_Country` e coloană indexată în listă (List settings → Indexed columns).

## Pasul 1: Flow

Folosește flow-ul existent `SIM Inventory - Export`, sau creează unul nou: **Instant cloud flow**
→ trigger **When Power Apps calls a flow (V2)**.

**Dacă ai construit deja flow-ul după prima versiune a ghidului:** șterge acțiunile
`Create export file`, `Update a row`, `Delay`, `Get export file` și `Delete file`. Apoi, în mail,
pune la Attachments Content conținutul din `Get template` (pasul 4).

## Pasul 2: Trigger

Input **Text** cu numele `Country`.

- Țara este `triggerBody()?['text']`.
- Mailul celui care a apăsat butonul este `triggerOutputs()?['headers']?['x-ms-user-email']`.

## Pasul 3: Compose `File name`

Inputs (expression):

```
concat('SIM_Inventory_', replace(trim(triggerBody()?['text']), ' ', '_'), '_', formatDateTime(convertFromUtc(utcNow(), 'GTB Standard Time'), 'yyyy-MM-dd_HHmmss'), '.xlsx')
```

Rezultatul e `SIM_Inventory_Bosnia_and_Herzegovina_2026-11-26_143015.xlsx`. Template-ul transformă
la loc `_` în spațiu și ignoră ultimele două segmente, data și ora.

> Formatul trebuie păstrat exact: prefixul `SIM_Inventory_`, spațiile din țară înlocuite cu `_`, apoi
> `_data_ora`. Formula din template se bazează pe el.

## Pasul 4: SharePoint `Get file content`

- Site Address: `simri`
- File Identifier: `/Shared Documents/ExportTemplate/SIM_Inventory_Export_TEMPLATE.xlsx`

Redenumește acțiunea `Get template`.

## Pasul 5: Office 365 Outlook `Send an email (V2)`

| Câmp | Valoare |
|---|---|
| To | `triggerOutputs()?['headers']?['x-ms-user-email']` |
| Subject | `SIM inventory export – @{triggerBody()?['text']}` |
| Attachments Name | `outputs('File_name')` |
| Attachments Content | **File Content** din `Get template` |

Body:

```
Hi,

Your inventory export for <b>@{triggerBody()?['text']}</b> is attached.

1. Save the attachment WITHOUT renaming it, then open it in the Excel desktop app (not in the browser).
2. The data is downloaded from SharePoint when the file opens (a few seconds). The first time,
   Excel may ask you to sign in: choose "Organizational account", Sign in, Connect.
   If you see a yellow "Enable Content" or "Enable Editing" bar, click it, then Data > Refresh All.
3. Edit, save, and upload the file through the app as usual.
```

> Mailul pleacă din căsuța celui care deține conexiunea Outlook a flow-ului. Pentru o căsuță
> comună, folosește **Send an email from a shared mailbox (V2)**.

## Pasul 6: `Respond to a PowerApp or flow`

Output **Text** `message`:

```
Export for @{triggerBody()?['text']} sent to @{triggerOutputs()?['headers']?['x-ms-user-email']}
```

## Pasul 7: Power Apps

Butonul **Export**, la **OnSelect**:

```
If(
    IsBlank(ddExportCountry.Selected),
    Notify("Pick a country first", NotificationType.Warning),
    Set(varExporting, true);
    Set(varExportResult, 'SIMInventory-Export'.Run(ddExportCountry.Selected.Value));
    Set(varExporting, false);
    Notify(varExportResult.message, NotificationType.Success)
)
```

- Pe buton: `DisplayMode = If(varExporting, DisplayMode.Disabled, DisplayMode.Edit)`.
- `'SIMInventory-Export'` e numele afișat de Power Apps după **Add flow**.
- `.Selected.Value` sau `.Selected.Result`, după tipul dropdown-ului.

---

## Primul test

1. Exportă o țară mică din app și salvează atașamentul fără să-l redenumești.
2. Deschide-l în **Excel desktop**. Dacă apare bara **Protected View → Enable Editing**, apasă-o.
3. Foaia **Export**, celula **B7**, trebuie să arate țara.
4. Logarea la SharePoint, doar prima dată: Organizational account → Sign in → Connect.
5. Foaia **query** se umple. Dacă nu se întâmplă nimic după câteva secunde, dă **Data → Refresh All**.
6. Dacă apare un mesaj de eroare, el spune ce lipsește:
   - **No country**: numele fișierului a fost schimbat. Scrie țara în B4, apoi Refresh All.
   - **No items**: SharePoint n-a găsit niciun rând cu `SIM_Country` exact ca țara din app.
     Valoarea din dropdown și cea din listă diferă, sau nu ai drept de citire pe listă.
   - O eroare cu **Access / 401 / 403**: logarea nu e făcută. Data → Get Data →
     Data Source Settings → șterge sursa simri → Refresh All și loghează-te cu Organizational account.
7. Compară numărul de rânduri cu lista filtrată în SharePoint.
8. Modifică ceva, salvează, închide și redeschide fișierul. Modificarea trebuie să fie încă acolo.

---

## Ce face template-ul (pentru întreținere)

- **Foaia Export → `tblParams`:**

  | Key | Ce conține |
  |---|---|
  | `Country` | B4, gol. Se completează doar manual și are prioritate. |
  | `SiteUrl` | Site-ul listei. |
  | `ListGuid` | ID-ul listei. |
  | `CountryFromFileName` | B7, o formulă cu `CELL("filename")` care scoate țara din numele fișierului. |

- **Interogarea `qInventory`**, în Data → Queries & Connections:
  - cere `/_api/web/lists(guid'…')/items` cu `$select` pe cele 20 de coloane,
    `$filter=SIM_Country eq '<țara>'` și `$top=5000`;
  - urmează `odata.nextLink` până la ultima pagină;
  - sortează după ID;
  - păstrează identificatorii ca text;
  - transformă o dată ISO în `dd-mm-yyyy`.
- **Refresh-ul la deschidere rulează în fundal.** Așa pornește după ce Excel a calculat B7. În
  primul plan pornea înainte și nu găsea țara.
- **Regula de siguranță:** dacă `Table_query` are deja date, refresh-ul le întoarce neschimbate.
  - Fișierul se umple deci o singură dată; editările nu sunt suprascrise niciodată.
  - Pentru o descărcare nouă: șterge toate rândurile tabelului, apoi Data → Refresh All.
- **Erori explicite** în loc de tabel gol: *No country*, *No items*, *Missing setting*.
- **`ColumnMap`** din `qInventory` = maparea Excel → SharePoint, aceeași ca în
  `payload_and_fieldmap.md`.
- **Coloanele U–AB, validările și cele 50 de reguli de culoare** sunt identice cu
  `SIM_Data_Validation_DEMO.xlsx`. Am comparat regulă cu regulă după construcție.

### Testat local (Excel desktop, cu o sursă simulată în locul SharePoint)

- două pagini cu `nextLink`, ID-uri nesortate, câmpuri goale, boolean, dată ISO;
- ICC_ID de 19–21 de cifre: păstrate integral;
- editări + refresh: editările rămân;
- rând lipit sub tabel + refresh: rândul rămâne și primește formulele;
- toate rândurile șterse + refresh: se descarcă din nou;
- țara din numele fișierului, inclusiv `Cote_d'Ivoire` cu ` (1)` la final;
- fișier redenumit: mesajul *No country*;
- deschis prin dublu-click: se umple singur, din numele fișierului.

### Netestat (nu am acces la listă)

- apelul real către SharePoint și logarea;
- tipul real al coloanelor `Status`, `IsVR`, `StartDate`. Dacă vreuna e Lookup sau Person,
  trebuie ajustată maparea.

### Limitări

- **Excel pe web** nu face refresh la surse SharePoint autentificate. Fișierul trebuie deschis în
  Excel desktop.
- Exportul nu consumă cota de Office Scripts. **Importul** o consumă în continuare.
