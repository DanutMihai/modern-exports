# Varianta Graph cu conectorul **HTTP** simplu (fără HTTP with Microsoft Entra ID)

Folosește `Export_Flow_Graph_Guide.md` pas cu pas. Diferența e una singură: fiecare
**„HTTP Graph”** de acolo devine acțiunea **HTTP**, autentificată cu o aplicație din Entra ID.
Template-ul e același: `SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx`.

> **Înainte de orice:** și conectorul **HTTP** e premium, deci licențele sunt la fel ca la varianta
> Entra ID. Mai verifică în **Data policies** că HTTP e în același grup cu SharePoint și Outlook.
> Dacă e blocat și el, mergi direct pe `Export_Flow_OfficeScript_Guide.md`.

---

## Pasul A: aplicația în Entra ID (o face un admin)

1. **Creează aplicația.** Entra admin center → **App registrations** → **New registration**:
   - nume: `SIM Inventory Export (flow)`;
   - Single tenant;
   - fără Redirect URI.

   Notează **Application (client) ID** și **Directory (tenant) ID**.
2. **Creează secretul.** **Certificates & secrets** → **New client secret**, cu valabilitate de
   12–24 de luni. Copiază **Value** pe loc, pentru că nu se mai afișează a doua oară. Pune o
   reamintire în calendar înainte să expire: în ziua aceea flow-ul se oprește.
3. **Dă permisiunea.** **API permissions** → Add → **Microsoft Graph** → **Application
   permissions** → **`Sites.Selected`** → **Grant admin consent**.
   - `Sites.Selected` nu dă acces la nimic până nu e legat explicit de un site (pasul 4), așa că e
     varianta cea mai restrânsă.
   - Alternativa e `Files.ReadWrite.All`, care dă acces la toate fișierele din tenant. Echipa de
     securitate va prefera aproape sigur `Sites.Selected`.
4. **Leagă aplicația doar de site-ul `simri`, cu drept de scriere.** O face un admin SharePoint,
   de exemplu din Graph Explorer (are nevoie de `Sites.FullControl.All`):

   ```
   GET https://graph.microsoft.com/v1.0/sites/deutschebank.sharepoint.com:/sites/simri
   → copiezi "id"

   POST https://graph.microsoft.com/v1.0/sites/{id}/permissions
   {
     "roles": ["write"],
     "grantedToIdentities": [
       { "application": { "id": "<Application (client) ID>", "displayName": "SIM Inventory Export (flow)" } }
     ]
   }
   ```

   Același lucru din PnP PowerShell:
   `Grant-PnPAzureADAppSitePermission -AppId <id> -DisplayName "SIM Inventory Export (flow)" -Site https://deutschebank.sharepoint.com/sites/simri -Permissions Write`

## Pasul B: secretul, în flow

Secretul nu se scrie direct în acțiuni. Variante, de la cea mai bună:
1. o **variabilă de mediu de tip Secret** (Azure Key Vault), într-o soluție;
2. o variabilă de mediu text, într-o soluție;
3. cel puțin un **Compose** `Graph secret` la începutul flow-ului, cu **Settings → Secure inputs /
   Secure outputs: On**, ca să nu apară în istoricul rulărilor.

## Pasul C: fiecare acțiune HTTP

| Câmp | Valoare |
|---|---|
| Method / URI / Headers / Body | exact ca la pasul corespunzător din ghidul Graph |
| **Authentication** | **Active Directory OAuth** |
| Authority | `https://login.microsoftonline.com` |
| Tenant | Directory (tenant) ID |
| Audience | `https://graph.microsoft.com` |
| Client ID | Application (client) ID |
| Credential Type | Secret |
| Secret | variabila / Compose-ul de la pasul B |

- Pe fiecare acțiune HTTP: **Settings → Secure inputs: On**.
- La `Add rows`: **Retry policy: None**, ca în ghidul Graph.
- Pe acțiunile HTTP, header-ul `Content-Type: application/json` acolo unde ai body.

**Ce nu se schimbă:** `Get items` rămâne acțiunea SharePoint, cu conexiunea proprietarului flow-ului.
Tot standard rămân `Get template`, `Create file` și mailul.

**De știut:** cu aplicația, accesul la fișier îl dă **aplicația**, nu un om. În istoricul
fișierului, modificările apar ca făcute de `SIM Inventory Export (flow)`.

---

## Pasul D: testul de 5 minute, înainte să construiești tot flow-ul

Nu sunt sigur că API-ul Excel din Graph (`createSession`, `rows/add`) acceptă **token-uri de
aplicație** (app-only) pe fișiere din SharePoint. Asta trebuie verificat înainte de restul.
Fă un flow separat, cu **Manually trigger a flow**, și următoarele HTTP-uri (auth ca la pasul C):

1. `GET https://graph.microsoft.com/v1.0/sites/deutschebank.sharepoint.com:/sites/simri:/drive/root:/ExportTemplate/SIM_Inventory_Export_PLAIN_TEMPLATE.xlsx`
   - Trebuie să vină 200, cu `id` și `parentReference.driveId`.
   - 401 sau 403 aici înseamnă că pasul A.4 n-a fost făcut sau secretul e greșit.
2. Pe o **copie** a template-ului: `POST .../drives/{driveId}/items/{id}/workbook/createSession`
   cu body `{"persistChanges": true}`.
3. `POST .../workbook/tables/Table_query/rows/add` cu header `workbook-session-id` și body:

   ```json
   { "values": [[ "Test","Row","+886912345678","Plan A","8988601234567890120","GD1","15-03-2026","Taiwan","P","CC","U","LE",false,"B","S","Active","eSIM","t@x.test",1,"352099001761481", null,null,null,null,null,null,null,null ]] }
   ```

4. `POST .../workbook/closeSession`.
5. Deschide copia și verifică:
   - rândul există, cu formule în U–AB;
   - ICC_ID e întreg;
   - RowErrors arată „OK”.

| Rezultat | Ce urmează |
|---|---|
| Toate 200/201, iar rândul arată bine | Construiești flow-ul din ghidul Graph, cu autentificarea de la pasul C. |
| `createSession` sau `rows/add` dă 401/403/400 cu mesaj despre *application* / *not supported* | Graph nu acceptă app-only pentru Excel aici. Mergi pe varianta Office Script. |
| Rândul există, dar U–AB sunt goale | `null` a suprascris formulele. Trimite-mi răspunsul și schimb metoda de scriere. |
