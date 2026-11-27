# Export inventar pe țară prin Microsoft Graph: ghid pas cu pas

Template: `SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx` (în acest folder). E template-ul de import, fără
rândurile de exemplu, **fără Power Query și fără adresa listei în fișier**.

## Diferența față de varianta Power Query

| | Power Query (`Export_Flow_Guide.md`) | Graph (acest ghid) |
|---|---|---|
| Cine citește lista | utilizatorul, cu contul lui | flow-ul, cu contul proprietarului |
| Utilizatorul are nevoie de drept pe listă | da (Read) | **nu** |
| Adresa listei apare în fișier | da | **nu** |
| Fișierul vine cu datele în el | nu, se umple la deschidere | **da** |
| Merge în Excel pe web | nu | **da** |
| Licență | standard | **premium** (HTTP with Microsoft Entra ID) |
| Office Scripts | 0 | 0 |
| Durata exportului | secunde | depinde de numărul de rânduri (vezi „Viteză”) |

## Cum funcționează

```
Power Apps: dropdown țară + Export
   ▼
Flow (proprietarul flow-ului are acces la listă; utilizatorul nu)
   1. răspunde imediat aplicației: "exportul a pornit, primești mail"
   2. copiază template-ul în folderul ExportFiles (vizibil doar pentru proprietar)
   3. SharePoint Get items: rândurile țării
   4. Select: fiecare rând devine un array de 28 de valori, în ordinea coloanelor din tabel
   5. Graph: deschide o sesiune pe fișier, adaugă rândurile câte 1000, închide sesiunea
   6. Graph: dă utilizatorului acces doar la ACEST fișier
   7. mail cu linkul
```

Utilizatorul deschide linkul. Fișierul e deja plin, cu validările, formulele și culorile pe toate
rândurile, și merge și în browser.

---

## Pasul 0: pregătire, o singură dată

1. **Licență și DLP.**
   - Conectorul **HTTP with Microsoft Entra ID (preauthorized)** e premium.
   - Pentru un flow pornit din Power Apps, regula Microsoft e simplă: fie fiecare utilizator al
     aplicației are licență premium (Power Apps Premium / per app), fie flow-ul are o licență
     **Power Automate Process**.
   - Verifică și că politica DLP a tenantului permite conectorul. Multe firme îl blochează.
2. **Folderul `ExportFiles`.** Creează-l în **Documents** pe site-ul `simri`. Apoi: Manage access →
   Advanced → **Stop inheriting permissions** și scoate pe toată lumea, în afară de proprietarul
   flow-ului și administratori. Fiecare utilizator primește acces doar la fișierul lui (pasul 13).
3. **Template-ul.** Încarcă `SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx` în
   `Documents/ExportTemplate`.
4. **Conexiunea HTTP with Microsoft Entra ID.** La prima acțiune îți cere:
   - Base Resource URL: `https://graph.microsoft.com`
   - Microsoft Entra ID Resource URI: `https://graph.microsoft.com`

   Te loghezi cu contul proprietarului flow-ului.

> Mai jos, „**HTTP Graph**” înseamnă acțiunea **Invoke an HTTP request** a acestui conector.

---

## Pasul 1: Trigger

**When Power Apps calls a flow (V2)**, input **Text** `Country`.

## Pasul 2: `Respond to a PowerApp or flow`, imediat

Output **Text** `message`:

```
Export for @{triggerBody()?['text']} started - you will receive an email with the link in a few minutes.
```

Răspunsul vine primul pentru că Power Apps așteaptă cel mult ~2 minute. Scrierea unei țări mari
durează mai mult, iar flow-ul continuă după răspuns.

## Pasul 3: Compose `File name`

```
concat('SIM_Inventory_', replace(trim(triggerBody()?['text']), ' ', '_'), '_', formatDateTime(convertFromUtc(utcNow(), 'GTB Standard Time'), 'yyyy-MM-dd_HHmmss'), '.xlsx')
```

## Pasul 4: SharePoint `Get file content` → `Get template`

`/Shared Documents/ExportTemplate/SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx`

## Pasul 5: SharePoint `Create file` → `Create export file`

- Folder Path: `/Shared Documents/ExportFiles`
- File Name: `outputs('File_name')`
- File Content: din `Get template`

## Pasul 6: HTTP Graph → `Get export item`

- Method: `GET`
- Url:

```
https://graph.microsoft.com/v1.0/sites/deutschebank.sharepoint.com:/sites/simri:/drive/root:/ExportFiles/@{encodeUriComponent(outputs('File_name'))}
```

Apoi un **Compose `Workbook url`**:

```
concat('https://graph.microsoft.com/v1.0/drives/', body('Get_export_item')?['parentReference']?['driveId'], '/items/', body('Get_export_item')?['id'], '/workbook')
```

> Dacă `body('Get_export_item')?['id']` iese gol, pune un **Parse JSON** pe body-ul acțiunii,
> generat dintr-un răspuns de test, și folosește câmpurile de acolo.

## Pasul 7: SharePoint `Get items` → `Get inventory`

| Câmp | Valoare |
|---|---|
| Site / List | `simri` / Global SIM Inventory |
| Filter Query | `SIM_Country eq '@{replace(triggerBody()?['text'],'''','''''')}'` |
| Order By | `ID` |
| Top Count | `5000` |

**Settings → Pagination: On, Threshold `100000`.** Fără asta primești doar primele 5000 de rânduri.

## Pasul 8: `Select` → `Shape rows`

- From: `body('Get_inventory')?['value']`
- Map: apasă butonul **Switch to text mode** (iconița din dreapta câmpului Map), ca să poți pune
  o expresie în loc de perechi cheie/valoare. Expresia:

```
createArray(
  coalesce(item()?['FirstName'], ''),
  coalesce(item()?['LastName'], ''),
  coalesce(item()?['PhoneNr'], ''),
  coalesce(item()?['Plan'], ''),
  coalesce(item()?['ICC_ID'], ''),
  coalesce(item()?['GD_ID'], ''),
  coalesce(item()?['StartDate'], ''),
  coalesce(item()?['SIM_Country'], ''),
  coalesce(item()?['Provider'], ''),
  coalesce(item()?['CostCenter'], ''),
  coalesce(item()?['UBR_Code'], ''),
  coalesce(item()?['Legalentity'], ''),
  if(equals(item()?['IsVR'], null), '', item()?['IsVR']),
  coalesce(item()?['BAN'], ''),
  coalesce(item()?['SOC'], ''),
  coalesce(item()?['Status']?['Value'], ''),
  coalesce(item()?['SIMType'], ''),
  coalesce(item()?['Email'], ''),
  item()?['ID'],
  coalesce(item()?['IMEI'], ''),
  null, null, null, null, null, null, null, null
)
```

- Cele **20 de valori** urmează ordinea coloanelor A–T din `Table_query`.
- Cele **8 `null`** de la final sunt coloanele de verificare U–AB. `null` înseamnă „nu atinge
  celula”, așa că formulele rămân.
- `Status` e Choice, deci vine ca obiect: `?['Value']`. Dacă alte coloane sunt tot Choice (de exemplu
  `SIMType` sau `Provider`), adaugă și la ele `?['Value']`.
- Dacă `StartDate` e coloană de tip **Date** în listă, înlocuiește linia ei cu:
  `if(empty(item()?['StartDate']), '', formatDateTime(convertFromUtc(item()?['StartDate'], 'GTB Standard Time'), 'dd-MM-yyyy'))`
- PhoneNr, ICC_ID, StartDate și IMEI ajung ca text, pentru că acele coloane sunt Text în template,
  inclusiv în rândurile noi. Nicio cifră nu se pierde.

## Pasul 9: HTTP Graph → `Create session`

- Method `POST`, Url: `@{outputs('Workbook_url')}/createSession`
- Body: `{ "persistChanges": true }`

ID-ul sesiunii este `body('Create_session')?['id']`.

## Pasul 10: `Apply to each` → `Write chunks`

- Input: `chunk(body('Shape_rows'), 1000)`
- **Settings → Concurrency control: On, Degree of parallelism `1`.** E obligatoriu: păstrează
  ordinea și sesiunea.

În interior, HTTP Graph → `Add rows`:

| Câmp | Valoare |
|---|---|
| Method | `POST` |
| Url | `@{outputs('Workbook_url')}/tables/Table_query/rows/add` |
| Headers | `workbook-session-id` : `@{body('Create_session')?['id']}` |
| Body | `{ "index": null, "values": @{items('Write_chunks')} }` |

**Settings → Retry policy: None.** Un retry după un timeout ar putea scrie același lot de două ori.

## Pasul 10b: HTTP Graph → `Remove placeholder row`

Template-ul are un rând gol pe rândul 2, care păstrează formulele. Fără el, un tabel complet gol
pierde regulile de culoare de pe primul rând. `rows/add` adaugă rândurile după el, așa că rândul
gol se șterge la sfârșit. Pune acțiunea **după** `Write chunks`, într-un `Condition`
`outputs('Total_rows')` > 0 (Compose `Total rows` = `length(body('Shape_rows'))`):

| Câmp | Valoare |
|---|---|
| Method | `POST` |
| Url | `@{outputs('Workbook_url')}/worksheets/query/range(address='A2:AB2')/delete` |
| Headers | `workbook-session-id` : `@{body('Create_session')?['id']}` |
| Body | `{ "shift": "Up" }` |

## Pasul 11: HTTP Graph → `Close session`

- Method `POST`, Url: `@{outputs('Workbook_url')}/closeSession`
- Headers: `workbook-session-id` : `@{body('Create_session')?['id']}`
- Body: `{}`
- **Configure run after:** bifează și *has failed*, ca sesiunea să se închidă și când scrierea
  eșuează.

## Pasul 12: `Condition` → a reușit scrierea?

Verifică dacă **Write chunks** a reușit. Cel mai simplu e să pui pașii 13–14 după `Close session`,
iar pe ramura de eșec un mail către tine cu `result('Write_chunks')`.

## Pasul 13: HTTP Graph → `Grant access`

- Method `POST`
- Url:

```
https://graph.microsoft.com/v1.0/drives/@{body('Get_export_item')?['parentReference']?['driveId']}/items/@{body('Get_export_item')?['id']}/invite
```

- Body:

```json
{
  "recipients": [ { "email": "@{triggerOutputs()?['headers']?['x-ms-user-email']}" } ],
  "requireSignIn": true,
  "sendInvitation": false,
  "roles": [ "write" ]
}
```

Utilizatorul primește acces **doar la fișierul lui**, nu la folder, la site sau la listă.

## Pasul 14: Office 365 Outlook `Send an email (V2)`

- To: `triggerOutputs()?['headers']?['x-ms-user-email']`
- Subject: `SIM inventory export – @{triggerBody()?['text']}`
- Body, cu linkul `body('Get_export_item')?['webUrl']`:

```
Your inventory export for <b>@{triggerBody()?['text']}</b> is ready:
<a href="@{body('Get_export_item')?['webUrl']}">open the file</a>

It contains @{length(body('Shape_rows'))} rows. Open it in Excel (desktop or browser), edit,
then upload it through the app as usual.
```

## Pasul 15: Power Apps

La fel ca în varianta Power Query, doar cu acest flow:

```
Notify('SIMInventory-ExportGraph'.Run(ddExportCountry.Selected.Value).message, NotificationType.Success)
```

## Pasul 16, opțional: curățenie

Un flow programat, zilnic:
1. SharePoint **Get files (properties only)** pe `ExportFiles`, cu Filter Query
   `Modified lt '@{addDays(utcNow(), -14)}'`.
2. **Apply to each** → **Delete file**.

---

## Primul test: ce verifici

1. Rulează flow-ul pe o țară mică.
   - Fiecare `Add rows` trebuie să întoarcă 201/200.
   - Mailul trebuie să vină cu linkul.
2. Deschide fișierul și verifică pe primul rând:
   - **Coloanele U–AB au formule.** Click pe o celulă din IsPhoneValid: în bara de formule trebuie
     să fie formula, nu un text.
     - Dacă sunt goale, înseamnă că Graph a scris `null` peste formule. Spune-mi și schimbăm
       metoda: rânduri goale mai întâi, apoi valorile doar în A–T.
   - **ICC_ID are toate cifrele** (19–20), iar PhoneNr păstrează `+`.
   - **Culorile apar**, iar **RowErrors** arată „OK” sau erorile.
3. Deschide linkul cu un cont **fără** acces la listă. Fișierul trebuie să se deschidă, iar lista
   nu trebuie să fie accesibilă.
4. Încarcă fișierul prin flow-ul de import, ca de obicei.

## Viteză: de testat cu țara cea mai mare

- Formulele de verificare folosesc `INDIRECT`, care e volatil. De aceea Excel pe web recalculează
  tot tabelul după fiecare lot adăugat, iar loturile devin tot mai lente pe măsură ce tabelul
  crește.
- Pornește cu loturi de **1000** și uită-te la durata fiecărui `Add rows` în istoricul rulării.
- Dacă un lot depășește ~60 de secunde sau dă 504, coboară la 500.
- Dacă țara cea mai mare e tot prea lentă, spune-mi. Pot rescrie formulele fără `INDIRECT`
  (non-volatile), ceea ce ajută și template-ul de import.

## Netestat de mine

N-am acces la tenant și n-am putut rula Graph. Template-ul l-am verificat în Excel:
- rândurile adăugate într-un tabel gol primesc formulele și formatul Text;
- un ICC_ID de 19 cifre rămâne text;
- un rând gol nu arată erori;
- validările și culorile sunt aceleași ca în template-ul de import.

Comportamentul Graph cu `null` în coloanele cu formule și viteza pe volume mari se confirmă la
primul test.
