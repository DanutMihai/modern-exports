# Export inventar pe țară, fără Office Scripts: ghid pas cu pas

Template: `SIM_Inventory_Export_TEMPLATE.xlsx` (în acest folder).

## Cum funcționează

```
Power Apps: dropdown țară + buton Export
   │  ExportInventory.Run(țara)
   ▼
Power Automate (doar acțiuni standard, niciun Office Script)
   1. ia template-ul și creează o copie temporară cu nume nou
   2. scrie țara în tblParams (Excel "Update a row", 1 apel)
   3. citește fișierul și îl trimite pe mail ca atașament
   4. șterge copia temporară, răspunde aplicației
   ▼
Utilizatorul deschide atașamentul în Excel desktop
   → Power Query descarcă din listă DOAR țara respectivă, cu contul lui
   → Table_query se umple; validările, formulele și culorile merg ca în template-ul de import
```

**Fișierul trimis pe mail nu conține date**, doar țara. Datele vin la deschidere, cu drepturile
utilizatorului. Atașamentul are ~55 KB indiferent de mărimea țării, iar flow-ul are mereu
aceleași 7 acțiuni, fie că țara are 50 de rânduri, fie 20.000.

---

## Pasul 0: o singură dată, în SharePoint

1. Pe site-ul `simri`, în **Documents**, creează două foldere: `ExportTemplate` și `ExportTemp`.
2. Încarcă `SIM_Inventory_Export_TEMPLATE.xlsx` în `ExportTemplate`.
   Utilizatorii nu au nevoie de acces la aceste foldere, flow-ul lucrează cu conexiunea lui.
3. Verifică faptul că `SIM_Country` e coloană indexată în listă (List settings → Indexed columns).
   Importul o cere deja, așa că ar trebui să fie.

> Dacă biblioteca sau folderele au alt nume, ajustează doar căile de la pașii 3, 4 și 8.

---

## Pasul 1: Flow nou

Power Automate → **Create** → **Instant cloud flow** → nume `SIM Inventory - Export`
→ trigger **When Power Apps calls a flow (V2)** → Create.

## Pasul 2: Trigger

În trigger: **+ Add an input** → **Text** → numele `Country`.

În restul ghidului:
- țara este `triggerBody()?['text']`;
- mailul celui care a apăsat butonul este `triggerOutputs()?['headers']?['x-ms-user-email']`.

> Dacă mai adaugi un input înaintea lui Country, cheia devine `text_1` etc. Verifică în
> Peek code al trigger-ului.

## Pasul 3: Compose `File name`

**+ New step** → **Compose**, redenumește-l `File name`. Inputs (expression):

```
concat('SIM_Inventory_', replace(trim(triggerBody()?['text']), ' ', '_'), '_', formatDateTime(convertFromUtc(utcNow(), 'GTB Standard Time'), 'yyyy-MM-dd_HHmmss'), '.xlsx')
```

Rezultat: `SIM_Inventory_Taiwan_2026-11-26_143015.xlsx`. Secundele din nume evită coliziunile
când doi oameni exportă aceeași țară în același timp.

## Pasul 4: SharePoint `Get file content` (template)

- Site Address: site-ul `simri`
- File Identifier: răsfoiește la `/Shared Documents/ExportTemplate/SIM_Inventory_Export_TEMPLATE.xlsx`

Redenumește acțiunea `Get template`.

## Pasul 5: SharePoint `Create file`

- Site Address: `simri`
- Folder Path: `/Shared Documents/ExportTemp`
- File Name: `outputs('File_name')`
- File Content: **File Content** din `Get template`

Redenumește acțiunea `Create export file`.

## Pasul 6: Excel Online (Business) `Update a row`

Asta e singura atingere a fișierului Excel. E un conector standard, nu Office Script.

| Câmp | Valoare |
|---|---|
| Location | site-ul `simri` |
| Document Library | Documents |
| File | *Enter custom value* → expression `outputs('Create_export_file')?['body/Id']` |
| Table | *Enter custom value* → `tblParams` |
| Key Column | `Key` |
| Key Value | `Country` |
| Row (Item) | `{"Value": "@{triggerBody()?['text']}"}` |

Pentru că fișierul e dinamic, Power Automate nu cunoaște coloanele și îți dă un singur câmp
pentru tot rândul. Lipește acolo JSON-ul de mai sus.

**Settings → Retry policy:** Exponential, Count 4, Interval `PT10S`. Un fișier abia creat poate
da o dată „file not found” sau „locked”, iar retry-ul rezolvă asta.

## Pasul 7: `Delay` 20 de secunde

Conectorul Excel salvează modificarea în fișier cu o mică întârziere. Fără pauză, pasul următor
poate citi fișierul fără țară, iar utilizatorul primește un tabel gol.

## Pasul 8: SharePoint `Get file content` (exportul)

- File Identifier: `outputs('Create_export_file')?['body/Id']`

Redenumește acțiunea `Get export file`.

## Pasul 9: Office 365 Outlook `Send an email (V2)`

| Câmp | Valoare |
|---|---|
| To | `triggerOutputs()?['headers']?['x-ms-user-email']` |
| Subject | `SIM inventory export – @{triggerBody()?['text']}` |
| Attachments Name | `outputs('File_name')` |
| Attachments Content | **File Content** din `Get export file` |

Body, de exemplu:

```
Hi,

Your inventory export for <b>@{triggerBody()?['text']}</b> is attached.

1. Save the attachment, then open it in the Excel desktop app (not in the browser).
2. The data is downloaded from SharePoint when the file opens. The first time Excel may ask you
   to sign in: choose "Organizational account", Sign in, Connect. If you see a yellow
   "Enable Content" bar, click it.
3. Edit, save, and upload the file through the app as usual.
```

> Mailul pleacă din căsuța celui care deține conexiunea Outlook a flow-ului. Dacă vreți să plece
> din căsuța de suport, folosește **Send an email from a shared mailbox (V2)** cu aceleași câmpuri.

## Pasul 10: SharePoint `Delete file`

- File Identifier: `outputs('Create_export_file')?['body/Id']`

Ca să nu rămână fișiere în `ExportTemp` când mailul eșuează: **Configure run after** → bifează
*is successful*, *has failed*, *is skipped* și *has timed out*.

## Pasul 11: `Respond to a PowerApp or flow`

Adaugă output **Text** `message` cu valoarea:

```
Export for @{triggerBody()?['text']} sent to @{triggerOutputs()?['headers']?['x-ms-user-email']}
```

## Pasul 12: Power Apps

1. **Power Automate** (panoul din stânga) → **Add flow** → `SIM Inventory - Export`.
2. Dropdown-ul de țară, de exemplu `ddExportCountry`, rămâne cel pe care îl ai deja.
3. Butonul **Export**, la **OnSelect**:

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

- Pe buton: `DisplayMode = If(varExporting, DisplayMode.Disabled, DisplayMode.Edit)`, ca să nu
  fie apăsat de două ori.
- Numele flow-ului în formulă e cel afișat de Power Apps după Add flow. Fără spații, de obicei
  `'SIMInventory-Export'`.
- Dacă dropdown-ul e un Choice din SharePoint, `.Selected.Value` e corect. Dacă e o listă de
  texte simple, folosește `.Selected.Value` sau `.Selected.Result`, după cum o ai.

---

## Primul test

1. Exportă o țară mică din app.
2. În mail, deschide atașamentul în **Excel desktop**.
3. Pe foaia **Export**, celula **B4** trebuie să conțină țara. Dacă e goală, mărește pauza de la
   pasul 7.
4. Prima dată Excel îți cere logarea la SharePoint: Organizational account → Sign in → Connect.
5. Foaia **query** se umple cu rândurile țării, iar culorile apar ca în template-ul de import.
6. Compară numărul de rânduri cu lista filtrată în SharePoint.
7. Fă o modificare, salvează, închide și redeschide fișierul. Modificarea trebuie să fie încă
   acolo: refresh-ul nu suprascrie datele existente.
8. Încarcă fișierul prin flow-ul de import, ca de obicei.

---

## Ce face template-ul (pentru întreținere)

- **Foaia Export → `tblParams`:** `Country` (scris de flow), `SiteUrl`, `ListGuid`.
- **Interogarea `qInventory`**, în Data → Queries & Connections:
  - citește `tblParams`;
  - cere `/_api/web/lists(guid'…')/items` cu `$select` pe cele 20 de coloane,
    `$filter=SIM_Country eq '<țara>'` (apostroful e dublat) și `$top=5000`;
  - urmează `odata.nextLink` până la ultima pagină;
  - sortează după ID;
  - păstrează identificatorii ca text.
  - Dacă `StartDate` vine din SharePoint ca dată ISO, o transformă în `dd-mm-yyyy`.
- **Funcția `fnFetchPage`:** un singur apel `Web.Contents` care citește o pagină JSON.
- **Regula de siguranță:** dacă `Table_query` are deja date, refresh-ul le întoarce neschimbate.
  - De aceea „refresh la deschidere” umple fișierul o singură dată și apoi nu mai atinge editările.
  - Pentru o descărcare nouă: șterge toate rândurile tabelului, apoi **Data → Refresh All**.
- **Maparea coloanelor** Excel → SharePoint e lista `ColumnMap` din `qInventory`, aceeași ca în
  `payload_and_fieldmap.md`: `Legal entity` → `Legalentity`, `SIM Type` → `SIMType`.
  - Dacă adăugați o coloană în listă și în template, adaug-o și acolo, la poziția ei.
- **Coloanele U–AB, validările și cele 50 de reguli de culoare** sunt identice cu
  `SIM_Data_Validation_DEMO_NEW.xlsx`. Am comparat regulă cu regulă după construcție.

### Testat local (Excel desktop, cu o sursă simulată în locul SharePoint)

- două pagini cu `nextLink`, ID-uri nesortate, câmpuri goale, boolean, dată ISO;
- valori de 19–21 de cifre: păstrate integral, ca text;
- coloanele de verificare, culorile și `RowErrors` pe toate rândurile încărcate;
- editări + refresh: editările rămân;
- rând lipit sub tabel + refresh: rândul rămâne și primește formulele;
- toate rândurile șterse + refresh: se descarcă din nou;
- țară cu apostrof (`Cote d'Ivoire`): filtrul e corect;
- fișierul deschis prin dublu-click: se umple singur la deschidere.

### Netestat (nu am acces la listă)

- apelul real către SharePoint și logarea;
- tipul real al coloanelor `Status`, `IsVR`, `StartDate`. Dacă vreuna e **Lookup** sau
  **Person**, vine ca ID sau obiect, nu ca text, și trebuie ajustată maparea.

### Limitări

- **Excel pe web** nu face refresh la surse SharePoint autentificate. Fișierul trebuie deschis în
  Excel desktop, altfel tabelul rămâne gol.
- Exportul nu mai consumă deloc cota de Office Scripts. **Importul** o consumă în continuare
  (`ValidateAndMatchImport`, ~12 apeluri la 60.000 de rânduri cu `pageSize` 5000).
