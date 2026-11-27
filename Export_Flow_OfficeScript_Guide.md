# Export inventar pe țară cu Office Script, un singur apel pe export: ghid pas cu pas

Script: `CopyRowsIntoTable_fixed.ts`, cel existent, nemodificat. Știe deja să scrie toate rândurile
într-un singur apel.
Template: `SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx`.

## De ce consumă mult mai puțină cotă

Limita Office Scripts în Power Automate e de **1.600 de apeluri pe zi pentru fiecare utilizator**
(contul conexiunii Excel a flow-ului). Cota se termina repede pentru că exportul trimitea rândurile
în bucăți mici, câte un apel pe bucată. Acum flow-ul trimite **8.000 de rânduri pe apel**, sub forma
unor array-uri compacte:

| Rânduri în țară | Apeluri |
|---|---|
| până la 8.000 | **1** |
| 20.000 | 3 |
| 60.000 | 8 |

Scriptul scrie oricum intern în loturi (`cellsPerBatch`). Calculul e oprit cât durează scrierea și
pornit o singură dată, la ultimul apel.

## Cum funcționează

```
Power Apps → flow (conexiunile proprietarului: SharePoint, Excel, Outlook; toate standard)
  1. răspunde imediat aplicației
  2. copiază template-ul în ExportFiles
  3. Get items: rândurile țării
  4. Select: array-uri de 20 de valori (A–T)
  5. Run script CopyRowsIntoTable: 1 apel la fiecare 8.000 de rânduri
  6. SharePoint "Grant access": utilizatorul primește acces DOAR la fișierul lui
  7. mail cu linkul
```

Utilizatorul nu are nevoie de drept pe listă și nu vede adresa ei. Fișierul vine plin și merge și în
browser. **Nu e nevoie de niciun conector premium.**

---

## Pasul 0: o singură dată

1. **`Documents/ExportFiles`.** Creează folderul. Manage access → Advanced → **Stop inheriting
   permissions**, apoi păstrează doar proprietarul flow-ului și administratorii.
2. **Template-ul.** Încarcă `SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx` în `Documents/ExportTemplate`.
3. **Scriptul.** Verifică că `CopyRowsIntoTable_fixed.ts` e salvat ca Office Script în OneDrive-ul
   proprietarului conexiunii Excel: Excel pe web → Automate → New script → lipești codul → Save
   as `CopyRowsIntoTable`. Dacă îl ai deja din exportul vechi, e același cod.

## Pasul 1: Trigger

**When Power Apps calls a flow (V2)**, input **Text** `Country`.

## Pasul 2: `Respond to a PowerApp or flow`

Output **Text** `message`:

```
Export for @{triggerBody()?['text']} started - you will receive an email with the link shortly.
```

## Pasul 3: Variabile

**Initialize variable** de două ori:
- `varStart`, Integer, valoarea `0`;
- `varCalls`, Integer, valoarea `0`.

## Pasul 4: Compose `File name`

```
concat('SIM_Inventory_', replace(trim(triggerBody()?['text']), ' ', '_'), '_', formatDateTime(convertFromUtc(utcNow(), 'GTB Standard Time'), 'yyyy-MM-dd_HHmmss'), '.xlsx')
```

## Pasul 5: SharePoint `Get file content` → `Get template`

`/Shared Documents/ExportTemplate/SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx`

## Pasul 6: SharePoint `Create file` → `Create export file`

- Folder Path: `/Shared Documents/ExportFiles`
- File Name: `outputs('File_name')`
- File Content: din `Get template`

## Pasul 7: SharePoint `Get items` → `Get inventory`

| Câmp | Valoare |
|---|---|
| Filter Query | `SIM_Country eq '@{replace(triggerBody()?['text'],'''','''''')}'` |
| Order By | `ID` |
| Top Count | `5000` |

**Settings → Pagination: On, Threshold `100000`.**

## Pasul 8: `Select` → `Shape rows`

- From: `body('Get_inventory')?['value']`
- Map: **Switch to text mode**, apoi expresia:

```
createArray(
  item()?['FirstName'], item()?['LastName'], item()?['PhoneNr'], item()?['Plan'],
  item()?['ICC_ID'], item()?['GD_ID'], item()?['StartDate'], item()?['SIM_Country'],
  item()?['Provider'], item()?['CostCenter'], item()?['UBR_Code'], item()?['Legalentity'],
  item()?['IsVR'], item()?['BAN'], item()?['SOC'], item()?['Status']?['Value'],
  item()?['SIMType'], item()?['Email'], item()?['ID'], item()?['IMEI']
)
```

- Valorile goale pot rămâne `null`: scriptul le scrie ca celule goale.
- `Status` e Choice. Orice altă coloană Choice primește și ea `?['Value']`.
- Dacă `StartDate` e coloană de tip Date:
  `if(empty(item()?['StartDate']), '', formatDateTime(convertFromUtc(item()?['StartDate'], 'GTB Standard Time'), 'dd-MM-yyyy'))`
- Coloanele U–AB nu apar aici. Scriptul scrie doar coloanele numite în `columnsCsv` și nu atinge
  formulele.

## Pasul 9: `Compose` → `Total rows`

```
length(body('Shape_rows'))
```

## Pasul 10: `Apply to each` → `Write chunks`

- Input: `chunk(body('Shape_rows'), 8000)`
- **Settings → Concurrency control: On, Degree of parallelism `1`.**

### 10a. Excel Online (Business) `Run script` → `Write rows`

| Parametru | Valoare |
|---|---|
| Location / Document Library | `simri` / Documents |
| File | *custom value*: `outputs('Create_export_file')?['body/Id']` |
| Script | `CopyRowsIntoTable` |
| tableName | `Table_query` |
| rowsJson | `string(items('Write_chunks'))` |
| clearExisting | `false` |
| startRowIndex | `variables('varStart')` |
| totalExpectedRows | `outputs('Total_rows')` |
| cellsPerBatch | `100000` |
| textColumnsCsv | `PhoneNr,ICC_ID,IMEI,StartDate` |
| columnsCsv | `FirstName,LastName,PhoneNr,Plan,ICC_ID,GD_ID,StartDate,SIM_Country,Provider,CostCenter,UBR_Code,Legal entity,IsVR,BAN,SOC,Status,SIM Type,Email,ID,IMEI` |
| spareRows | `200` |
| deferCalculationRestore | `less(add(variables('varStart'), length(items('Write_chunks'))), outputs('Total_rows'))` |
| repairFormulas | `true` |

- **`columnsCsv` folosește numele din Excel** (`Legal entity`, `SIM Type`), nu numele interne din
  SharePoint.
- **`deferCalculationRestore`** e `true` la toate apelurile în afară de ultimul. Recalcularea se
  face o singură dată, la final.
- **Settings → Retry policy: None.** Un retry după un timeout ar putea scrie același lot de două ori.

### 10b. `Increment variable`

- `varStart` crește cu `length(items('Write_chunks'))`.
- `varCalls` crește cu `1`.

## Pasul 11: SharePoint `Grant access to an item or a folder`

| Câmp | Valoare |
|---|---|
| Site / Library | `simri` / Documents |
| Id | `outputs('Create_export_file')?['body/ItemId']` |
| Recipients | `triggerOutputs()?['headers']?['x-ms-user-email']` |
| Role | **Can edit** |
| Notify recipients | **No** (trimitem noi mailul) |

Utilizatorul primește acces **doar la acest fișier**.

## Pasul 12: SharePoint `Get file properties` → `Export file properties`

- Id: `outputs('Create_export_file')?['body/ItemId']`

Linkul spre fișier este `outputs('Export_file_properties')?['body/{Link}']`.

## Pasul 13: `Send an email (V2)`

- To: `triggerOutputs()?['headers']?['x-ms-user-email']`
- Subject: `SIM inventory export – @{triggerBody()?['text']}`
- Body:

```
Your inventory export for <b>@{triggerBody()?['text']}</b> is ready (@{outputs('Total_rows')} rows):
<a href="@{outputs('Export_file_properties')?['body/{Link}']}">open the file</a>

Open it in Excel (desktop or browser), edit, then upload it through the app as usual.
```

## Pasul 14: erori și curățenie

- **Dacă `Write chunks` eșuează.** Pe o ramură cu *run after: has failed*:
  - un mail către tine cu `result('Write_chunks')`;
  - **Delete file** pe `outputs('Create_export_file')?['body/Id']`.
  
  Un fișier rămas la jumătate poate avea calculul pe Manual și nu trebuie trimis.
- **Dacă țara are 0 rânduri.** Pune un `Condition` înainte de pasul 10, `outputs('Total_rows')` = 0.
  Pe ramura „da”: mail „no items for this country”, apoi Delete file.
- **Curățenie.** Un flow programat zilnic șterge din `ExportFiles` fișierele mai vechi de 14 zile
  (Get files → filtru pe `Modified` → Delete file).

---

## Primul test

1. Exportă o țară mică. Rularea trebuie să arate `varCalls` = 1.
2. Deschide linkul și verifică:
   - **U–AB au formule** și **culorile apar**;
   - ICC_ID are toate cifrele, iar PhoneNr păstrează `+`;
   - **Formulas → Calculation Options** e pe **Automatic**.
3. Exportă țara cea mai mare și uită-te în istoric la durata fiecărui `Write rows`.
   - Limita e 120 de secunde pe apel. Ultimul apel e cel mai lung, pentru că face și recalcularea.
   - Dacă se apropie de limită, coboară lotul de la 8000 la 5000.
   - Dacă primești o eroare de mărime a cererii (payload prea mare), coboară lotul la 5000.
4. Deschide linkul cu un cont **fără** acces la listă. Fișierul se deschide, lista nu.
5. Încarcă fișierul prin flow-ul de import.

## Cotă, în total

- **Exportul:** 1 apel pentru o țară de până la 8.000 de rânduri.
- **Importul:** consumă în continuare, ~2 apeluri la 10.000 de rânduri cu `pageSize` 5000.
- Ambele se numără pe contul conexiunii Excel. Dacă tot e strâns, folosește pentru export
  o conexiune Excel cu alt cont, de exemplu un cont de serviciu. Cota se socotește separat
  pentru fiecare cont.
