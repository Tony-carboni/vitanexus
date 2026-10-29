# Boekhouddashboard managementvennootschap: onderzoek en voorstel

Status: brainstormdocument, 29 oktober 2026. Niets is nog gebouwd.

Doel van de app: één lijst met elke zakelijke banktransactie én elke binnengekomen factuur (ook de nog onbetaalde), met per regel twee statussen: **"zit in de boekhouding (Billit)?"** en **"is betaald?"**. Daaruit volgen twee werklijsten: *nog te betalen* en *nog document aan te leveren*.

Onderzoeksmethode: webonderzoek via vier parallelle deelonderzoeken (Billit API, banktoegang KBC, Peppol in België, niet-Peppol facturen). De sessie kon de meeste leveranciersites niet rechtstreeks openen, dus een aantal details komt uit zoekresultaten en community-code in plaats van de officiële pagina's. Die punten staan gemarkeerd als **[te verifiëren]**.

---

## 1. Conclusies in het kort

1. **Billit is de spil, niet de boekhouder zijn software.** Het Billit "dossier" is van jouw vennootschap. De boekhouder werkt via een gratis accountantsportaal dat aan jouw dossier gekoppeld is en exporteert documenten naar zijn eigen boekhoudpakket. Jij kunt (of moet kunnen) zelf inloggen op dat dossier en daar je eigen API-sleutel aanmaken. Billit staat een API-sleutel expliciet toe voor niet-commerciële integraties "om je eigen administratie te automatiseren". Dat is precies dit project.
2. **Peppol-facturen komen automatisch in Billit** en zijn via de API uit te lezen als "Cost"-orders of als ruwe UBL via de Peppol-inbox. Een tweede ontvangend access point is niet mogelijk (één ontvanger per ondernemingsnummer), dus Billit blijft de ontvanger.
3. **Banktransacties zonder PDF-download kan op twee manieren.** (a) Gratis: KBC-CODA-koppeling in Billit zelf ("Billbank", activeren in KBC Touch), waarna de transacties via de Billit API beschikbaar zijn. (b) Eigen PSD2-koppeling via een vergunde aggregator: Enable Banking (gratis voor persoonlijk gebruik) of Ponto van Isabel (€4 per maand). KBC's eigen PSD2-API is enkel voor vergunde partijen en screen scraping van KBC Touch is niet haalbaar (altijd sterke authenticatie) en in strijd met de voorwaarden.
4. **itsme kan nooit geautomatiseerd worden.** Elke PSD2-toestemming vraagt een menselijke bevestiging in KBC Mobile of itsme, en vervalt na 180 dagen. Twee keer per jaar handmatig hertoestemmen is de prijs; verder loopt alles onbemand (maximaal 4 ophalingen per dag).
5. **Niet-Peppol facturen** (bol.com, Amazon, SaaS, e-mail) los je op met een eigen facturenmailbox die doorgestuurd wordt naar Billit én naar je app, met LLM-extractie (Claude met PDF-invoer en gestructureerde output, circa een tiende eurocent tot een cent per factuur). bol.com heeft geen koper-API; Amazon Business wel, maar die is praktisch onbereikbaar voor een eenmanszaak. Daar blijft een maandelijkse handmatige veegronde nodig, die het dashboard wel gericht maakt: het toont precies welke betaling nog geen document heeft.
6. **Matching** factuur-transactie is in België relatief dankbaar: gestructureerde mededeling (OGM +++…+++), IBAN van de leverancier, bedrag en datum. Bij Peppol-facturen zitten IBAN en OGM gestructureerd in de UBL.

Aanbeveling: bouw een **eigen kleine hub** (optie B hieronder), maar start met de **Billit-centrische MVP** (optie A) omdat die in één weekend werkt en 80% van de pijn wegneemt.

---

## 2. De bouwstenen in detail

### 2.1 Billit API

| Onderwerp | Bevinding |
|---|---|
| Docs | https://docs.billit.be (gidsen), https://docs.billit.be/llms.txt (machineleesbare index), Swagger https://api.billit.be/swagger/ui/index |
| Basis-URL | Productie `https://api.billit.be/v1/`, sandbox `https://api.sandbox.billit.be/v1/` (gratis, volledig gescheiden sleutels en PartyID's) |
| Authenticatie | Headers `apiKey` + `partyID` (numeriek bedrijfs-ID, te vinden in de URL van "Mijn bedrijf"). API-sleutel aanmaken onder Profiel → Gebruikers & API-sleutel. OAuth is enkel vereist voor commerciële multi-klant integraties. |
| Datamodel | Alles is een "Order": `OrderType` Invoice/CreditNote/…, `OrderDirection` **Cost** (aankoop) of **Income** (verkoop). Velden o.a. `OrderNumber` (factuurnummer leverancier), `TotalIncl`, `Paid`, `PaidDate`, `ToPay`, `OrderStatus` (Draft, ToPay, Paid, Sent, …), `OrderPDF`, `Attachments`, `LastModified`. |
| Lijsten | `GET /v1/orders?$filter=OrderDirection eq 'Cost' and LastModified ge DateTime'2026-01-01'` (OData, 120 per pagina, `NextPageLink`) |
| Bestanden | `GET /v1/files/{FileID}` geeft PDF of originele UBL (base64) |
| Peppol-inbox | `GET /v1/peppol/inbox` (max 10, oudste eerst, geen filter) → `GET /v1/files/{FileID}` voor de onbewerkte UBL; `POST /v1/peppol/inbox/{id}/confirm` |
| Banktransacties | `GET /v1/financialTransactions` (enkel gevuld als er een bankkoppeling actief is in het dossier), `POST /v1/financialTransactions/commands/import` **[te verifiëren]** |
| Aankoopfactuur toevoegen | `POST /v1/orders` met `OrderDirection: "Cost"` en PDF in `OrderPDF`; of ongestructureerd naar Snelle invoer (OCR) via `POST /v1/documents` **[request-body te verifiëren]**, of gewoon mailen naar het unieke Snelle-invoer-adres |
| Betaling registreren | `POST /v1/orders/{id}/payments` (uitlezen van betalingen per order wordt niet ondersteund; gebruik `Paid`/`PaidDate`) |
| Webhooks | `POST /v1/webhooks` met EntityType `Order` of `Message` en updatetypes I/U/D; HMAC-SHA256-handtekening; endpoint moet geldig TLS hebben |
| "Geboekt"-status | Het accountantsportaal exporteert documenten naar het boekhoudpakket; daarna krijgen ze in Billit een "geëxporteerd"-vlag en later een boekingsnummer. Of die vlag via de API zichtbaar is: **[te verifiëren in Swagger]**. Fallback: "zit in Billit" = aanwezig als Cost-order. |
| Rate limits | Niet gepubliceerd **[te verifiëren]** |
| Kosten | Peppol ontvangen vereist een betaald plan (Starter ±€7,50/maand tot 25 documenten/maand; prijzen verschillen per regiopagina). Of de API een betaald plan vereist: **[te verifiëren bij Billit support]**. Extra gebruiker €5/maand. |
| Extra's | Officiële read-only MCP-server `https://mcp.my.billit.eu/` (handig om met Claude vragen over je Billit-data te stellen), Make.com-app, Postman-collectie, Billit Expenses-app voor bonnetjes, IMAP-scanning van je mailbox elke 60 minuten |

Toegangsmodel: één API-sleutel per **gebruiker**, geldig voor alle dossiers waar die gebruiker toegang toe heeft; `partyID` kiest het dossier. Je hebt dus geen sleutel van de boekhouder nodig. Als je nog geen login hebt, kan de boekhouder vanuit het dossier "logingegevens verzenden".

### 2.2 Peppol in België (stand 2026)

- Sinds 1 januari 2026 is gestructureerd e-factureren (Peppol BIS 3 / UBL) verplicht voor binnenlandse B2B tussen Belgisch gevestigde btw-plichtigen, zowel verzenden als ontvangen. Boetes €1.500 / €3.000 / €5.000 sinds 1 april 2026. Een managementvennootschap valt volledig in scope.
- Hermes (overheidsfallback naar PDF per e-mail) is eind 2025 stopgezet. Wie geen access point heeft, is onbereikbaar.
- E-reporting (near-real-time rapportering aan de fiscus) is gepland voor 1 januari 2028; voorontwerp goedgekeurd in juli 2026, nog niet gestemd.
- **Eén ontvangend access point per deelnemer-ID** (0208:ondernemingsnummer). Een kopie via een tweede provider kan niet. Het btw-nummer-ID (9925) als "tweede brievenbus" gebruiken wordt door de Belgische Peppol-autoriteit sterk afgeraden en leveranciers sturen er niet naartoe.
- Gevolg: Billit blijft ontvanger, de app leest uit Billit. Alternatieve API-first providers als je ooit zelf ontvanger wil worden: Recommand (Belgisch, gratis tot 25 documenten/maand, webhooks), e-invoice.be (€0,25 per factuur), Peppyrus. Dan moet je wel zelf de boekhouder voeden.
- Nuttige UBL-velden voor het dashboard: leverancier (naam, btw, KBO), `cbc:ID` (factuurnummer), `IssueDate`, `DueDate`, `PayableAmount`, btw-totalen, `PaymentMeans` met IBAN (`PayeeFinancialAccount/ID`) en betalingsreferentie (`PaymentID`, hier staat meestal de OGM), en `AdditionalDocumentReference` met eventueel ingesloten PDF (vaak aanwezig, niet gegarandeerd).
- Welke provider je bedrijf nu bedient, zie je niet in directory.peppol.eu maar wel via een SMP-lookup, bijvoorbeeld op https://peppol.tools of https://www.efacturatieontcijferd.be/en/peppol-checker/.

### 2.3 Banktransacties van de KBC-zakenrekening

| Optie | Kost per jaar (excl. btw) | Menselijke stap | Beoordeling |
|---|---|---|---|
| **KBC-CODA-koppeling in Billit ("Billbank")** | €0 (activatie in KBC Touch, eenmalig €0,10 IBAN-check) | eenmalig activeren | Simpelste weg als je dashboard toch op Billit leest. Dagelijkse import (niet realtime). **[Controleren of deze gratis koppeling nog bestaat en of de boekhouder niet al Codabox gebruikt in het dossier]** |
| **Enable Banking** (Fins, vergund AISP) | €0 in "restricted production" voor persoonlijk gebruik | hertoestemming elke 180 dagen via KBC Mobile | Beste gratis optie voor een eigen koppeling. Max 4 ophalingen/dag. Dekking van KBC *zakelijke* rekeningen in hun paneel nog te controleren. Python-SDK en Firefly III/Actual Budget gebruiken het al. |
| **Ponto** (Isabel Group, Belgisch) | €48 (€4/maand per rekening) | hertoestemming elke 180 dagen, mail als herinnering | Belgisch, expliciete KBC-ondersteuning, gebruikt door veel Belgische facturatietools. Betalingsinitiatie mogelijk (€0,10 per overschrijving). |
| Codabox Business (CODA) | ±€119 + €24 activatie | geen | Levert aan boekhoudsoftware, niet rechtstreeks aan jou; API enkel voor softwareleveranciers. |
| KBC CODA via Online for Business / Isabel 6 + Isabel Connect | €96 tot >€700 | geen | Overkill voor één rekening. |
| KBC's eigen PSD2-API | NBB-vergunning + eIDAS-certificaat | – | Niet haalbaar voor een particulier/kleine vennootschap. |
| GoCardless Bank Account Data | gratis maar **gesloten voor nieuwe gebruikers sinds juli 2025** | – | Valt af. |
| Screen scraping KBC Touch / Business Dashboard | €0 | faalt op SCA | Technisch onmogelijk zonder mens (elke login vraagt KBC Mobile, KBC Sign of kaartlezer) en in strijd met de voorwaarden. Valt af. |
| Handmatige CSV uit KBC Touch | €0 | maandelijks inloggen | Als fallback-import behouden (KBC kan afschriften automatisch als CSV klaarzetten, maar downloaden vereist login). |

Over itsme en SCA: de PSD2-regels (gedelegeerde verordening 2022/2360) verplichten hernieuwde sterke authenticatie elke 180 dagen en beperken onbemande toegang tot 4 keer per dag en 90 dagen historiek bij eerste toestemming. itsme-accounts zijn strikt persoonlijk; er bestaat geen API om een itsme-bevestiging te omzeilen. Dit is dus een harde grens voor elke oplossing: twee keer per jaar een tik op je telefoon.

### 2.4 Niet-Peppol facturen

- **bol.com Zakelijk**: facturen voor artikelen verkocht door bol zelf komen automatisch per e-mail (naar factuur-adres én besteller). Voor partnerverkopers moet je de btw-factuur aanvragen via de knop "aanvragen bij partner"; Belgische partners moeten sinds 2026 zelfs via Peppol factureren. Er is geen koper-API en geen bulkdownload. Verwachting: e-mailingest dekt het grootste deel, de rest komt op een maandelijkse handmatige lijst.
- **Amazon Business**: zet "facturen per e-mail" aan in de bedrijfsinstellingen **[exact menupad te verifiëren]**, registreer je Peppol-ID in je Amazon Business-account (vangt de Belgische verkopers in scope), en gebruik de bulkdownload (zip) in de facturenrapportage als backstop. De Amazon Business API bestaat maar is goedkeuringsgebonden en gericht op integrators; overslaan.
- **E-mail**: maak een apart adres (bv. `facturen@jouwbedrijf.be`) en gebruik dat overal als factuuradres. Stuur automatisch door naar (1) het unieke Snelle-invoer-adres van Billit en (2) je eigen app. Technisch: Gmail API met OAuth-consentscherm op "Intern" (Google Workspace; geen verificatie, tokens vervallen niet) of Microsoft Graph `Mail.Read`. Wachtwoord-IMAP is bij Google en Microsoft uitgefaseerd. Filter op PDF/XML-bijlagen, dedupliceer op SHA-256, markeer mails met enkel een "bekijk je factuur"-link voor handmatige actie.
- **Extractie uit PDF**: Claude API met PDF-invoer en gestructureerde output (schema: leverancier, btw-nummer, IBAN, factuurnummer, datums, netto/btw/bruto, OGM). Kost per factuur van één pagina: ruwweg €0,001 (Haiku) tot €0,02 (Opus); helft via Batches API. Voeg een rekencontrole toe (netto + btw = bruto) en een reviewwachtrij bij afwijking. ZUGFeRD/Factur-X-PDF's (Duitse leveranciers) bevatten ingesloten XML: eerst die lezen. Gehoste parsers (Google Document AI ±€0,10/document, Azure gratis tot 500 pagina's/maand) zijn een fallback; Veryfi/Rossum/Klippa zijn enterprise-geprijsd.
- **Kant-en-klare verzamelaars** (GetMyInvoices €19+/maand, invoicefetcher €4,59+/maand, Tailride): niemand ondersteunt bol.com aantoonbaar; voor Amazon wel. Niet nodig als de mailbox-route werkt.

### 2.5 Matching factuur ↔ transactie

Cascade, geleend van de gedocumenteerde Odoo-reconciliatie en Firefly III-regels:

1. Exacte OGM-match (+++123/4567/89012+++, 12 cijfers, controle = eerste 10 mod 97, 0 → 97).
2. Factuurnummer of betalingsreferentie in de mededeling (regex, spaties/slashes normaliseren).
3. Bekende tegenpartij (IBAN → leverancier) + bedrag onder de openstaande facturen van die leverancier.
4. Bedrag binnen tolerantie (±€0,02; ruimer voor kaartbetalingen in vreemde munt) + datumvenster (betaaldatum ≥ factuurdatum, ≤ vervaldag + 30 dagen).
5. Som van meerdere facturen voor één betaling, en meerdere kaartafschrijvingen voor één Amazon-bestelling.
6. Kaartbetalingen "BOL.COM" / "AMZN Mktp BE" zonder referentie: match op bedrag + datum + merchantlabel.

Praktische hefboom: betaal zelf altijd met de OGM van de leverancier (of het factuurnummer in de mededeling). Dan is je eigen matching voor 90% triviaal.

Belgische parsers als je ooit CODA/CAMT-bestanden krijgt: Python `pycoda` of `febelfin-coda`, `pycamt`; PHP `codelicious/php-belgianbankstatement-parser` (kent ook KBC-CSV).

---

## 3. Het statusmodel van het dashboard

Elke regel is een **"financieel feit"** met drie mogelijke oorsprongen: banktransactie, factuur, of beide (gematcht).

| Regel | In boekhouding? | Betaald? | Actie |
|---|---|---|---|
| Factuur (Peppol of mail) zonder transactie | ja (Peppol komt in Billit) / nee (mail die nergens kwam) | nee | **Betalen** (toon IBAN + OGM + bedrag, eventueel EPC-QR-code om in KBC Mobile te scannen) |
| Transactie zonder factuur | nee | ja | **Document zoeken** (bol.com partner, Amazon-zip, SaaS-portaal) en uploaden naar Billit |
| Factuur + transactie gematcht, factuur in Billit | ja | ja | klaar |
| Factuur + transactie gematcht, factuur enkel in je app | nee | ja | **Doorsturen naar Billit** (één knop: `POST /v1/orders` of mail naar Snelle invoer) |
| Transactie die geen document hoort te hebben (bankkosten, btw-betaling, loon, dividend) | n.v.t. | ja | regel "geen document nodig" (handmatig of via regel op tegenpartij) |

"In boekhouding" betekent in eerste instantie **"aanwezig als Cost-order in Billit"**. Als de API de "geëxporteerd naar boekhouder"-vlag blootlegt, wordt dat een derde niveau: in Billit → geëxporteerd → geboekt.

---

## 4. Oplossingsopties

### Optie A: Billit-centrisch (MVP, kleinste bouw)

Alles gaat naar Billit, de app is een dunne leeslaag erbovenop.

- Bankkoppeling **in Billit** (Billbank KBC-CODA gratis, of Ponto €4/maand, of de Codabox van de boekhouder).
- Peppol komt automatisch in Billit. Mailfacturen via de facturenmailbox doorgestuurd naar het Snelle-invoer-adres (Billit doet OCR). Bonnetjes via de Billit Expenses-app.
- Je app haalt 4x per dag op: alle Cost-orders, alle `financialTransactions`, en bouwt zelf de matching en de twee werklijsten. Optioneel een Order-webhook voor near-realtime.
- Hosting: één container (Python FastAPI of Node) + SQLite op een kleine VPS, Fly.io of Railway; of zelfs een lokaal script dat een statische HTML-pagina genereert.

Voordelen: één koppeling, één login, geen PSD2-hertoestemming in je eigen app (Billit regelt de bankkoppeling), boekhouder ziet alles. Nadelen: je bent afhankelijk van wat Billit via de API toont (betaalstatus is Billit's eigen matching; transacties enkel als de bankkoppeling in het dossier staat), en Billit telt elk ontvangen document mee in je plan.

### Optie B: Eigen hub (aanbevolen eindbeeld)

Een eigen backend is de bron van waarheid voor het dashboard; Billit is één van de bronnen én de bestemming voor de boekhouder.

```
KBC ──PSD2 (Enable Banking / Ponto)──▶ ┐
Billit API (Cost-orders, Peppol-inbox, files) ──▶ ├─▶ eigen DB ──▶ matcher ──▶ dashboard
facturen@ mailbox ──Gmail API/Graph──▶ Claude-extractie ─▶ ┘            │
                                                                         └─▶ "naar Billit" (POST /v1/orders)
```

- Bank rechtstreeks via Enable Banking (gratis) of Ponto (€4/maand); transacties staan in je eigen DB, onafhankelijk van Billit.
- Peppol via Billit API (`orders` Cost + `peppol/inbox` voor de ruwe UBL, zodat IBAN en OGM gestructureerd beschikbaar zijn).
- Mailfacturen worden door je app geparst (Claude) én naar Billit gestuurd; status "in Billit" wordt gecontroleerd door te zoeken op factuurnummer + leverancier in de Cost-orders.
- Betaallijst met EPC-QR-code per factuur (scanbaar in KBC Mobile), of later betalingsinitiatie via Ponto (€0,10 per overschrijving; vraagt nog steeds SCA, maar vooringevuld).
- Hosting zoals A, plus een webhook-endpoint met TLS (Billit) en een redirect-URL (PSD2-consent).

Voordelen: volledige controle, eigen matching en regels, werkt ook als de boekhouder ooit van tool wisselt. Nadelen: meer bouwwerk, twee keer per jaar PSD2-hertoestemming, secrets beheren (API-sleutels, OAuth-tokens, Claude-sleutel).

### Optie C: No-code / Billit zelf + MCP

- Billit zelf met bankkoppeling toont al "betaald/onbetaald" per aankoopfactuur en een lijst van niet-gematchte transacties. Met de officiële Billit MCP-server kun je Claude vragen stellen als "welke betalingen van oktober hebben geen factuur?".
- Make.com heeft een officiële Billit-app (Watch Orders, List orders) waarmee je zonder code een Google Sheet of Notion-overzicht kunt vullen.

Voordelen: nul bouw. Nadelen: geen echte "eigen" dashboard, mailfacturen en bol.com blijven handwerk, matching is wat Billit doet.

### Optie D: Kant-en-klaar kopen

GetMyInvoices/invoicefetcher (portaalfetchers), Accountable, Dext. Geen enkele dekt KBC + Billit + bol.com + jouw statusmodel. Niet aanbevolen.

---

## 5. Voorgesteld traject

| Fase | Wat | Resultaat |
|---|---|---|
| 0. Voorbereiding (deze week, geen code) | Login op jouw Billit-dossier bemachtigen, API-sleutel + PartyID; sandbox-account; nagaan welke bankkoppeling het dossier heeft; facturen@-adres aanmaken en overal instellen; Peppol-ID in Amazon Business registreren; bol.com-factuuradres zetten. Aantal vragen aan Billit support (zie §6). | alle toegangen |
| 1. MVP (optie A) | Script/backend dat Cost-orders + financialTransactions ophaalt, matcht, en de twee werklijsten toont. | eerste bruikbare dashboard |
| 2. Eigen bankkoppeling | Enable Banking of Ponto toevoegen; vergelijken met Billit's transacties. | onafhankelijk van Billit-bankkoppeling |
| 3. Mail-ingest + extractie | Gmail/Graph-koppeling, Claude-extractie, "naar Billit"-knop, dedup. | mailfacturen automatisch |
| 4. Comfort | Webhooks, EPC-QR-betaalknop, regels voor "geen document nodig", maandafsluitingsrapport voor de boekhouder, bol.com/Amazon-herinneringen. | maandelijkse routine in 10 minuten |

Technische voorkeur (te bespreken): Python 3.12 + FastAPI + SQLite (later Postgres), HTMX of een lichte React-frontend, Docker, cron 4x/dag, secrets via omgevingsvariabelen, dashboard afgeschermd met één passkey/wachtwoord. Python heeft de rijkste bibliotheken voor UBL, CODA en de Enable Banking-SDK.

---

## 6. Open vragen

Voor jou:
1. Heb je vandaag een eigen login op het Billit-dossier van je vennootschap, of beheert de boekhouder het volledig?
2. Welke bankkoppeling staat er nu in dat dossier (geen, Codabox, Ponto, Billbank)? En gebruikt de boekhouder Billit alleen als brievenbus of ook als "boekhouding"?
3. Hoeveel documenten per maand ongeveer (bepaalt het Billit-plan en of extractiekosten relevant zijn)?
4. Mail: Google Workspace op eigen domein, gewone Gmail, of Microsoft 365?
5. Waar wil je het dashboard draaien: lokaal op je laptop, een VPS, of een cloudplatform? En wil je er vanaf je telefoon aan kunnen?
6. Wil je de betaalstap in de app (QR / Ponto-betaalinitiatie) of volstaat een lijst "te betalen"?

Te verifiëren bij Billit support of in Swagger (kon niet online bevestigd worden):
- Is de API beschikbaar op elk betaald plan? Rate limits?
- Toont de Order-API de "geëxporteerd naar boekhouder"- of "geboekt"-vlag?
- Werkt `$filter=Paid eq false` op `/v1/orders`?
- Exacte request-body van `POST /v1/documents` (Snelle invoer) en `POST /v1/financialTransactions/importFile`.
- Bestaat de gratis Billbank KBC-CODA-koppeling nog en wat zijn de voorwaarden?

Te verifiëren bij Enable Banking / Ponto:
- Staat de KBC *zakelijke* rekening in hun dekking en via welke KBC-app verloopt de toestemming (KBC Mobile vs. KBC Business)?
- Enable Banking: huidige voorwaarden van het gratis "personal use"-productieplan.

---

## 7. Bronnen (selectie)

Billit: https://docs.billit.be/docs/getting-started · https://docs.billit.be/docs/authentication · https://docs.billit.be/docs/oauth-faq · https://docs.billit.be/docs/odata · https://docs.billit.be/docs/get-list-of-incoming-invoices · https://docs.billit.be/docs/get-files-peppol-inbox · https://docs.billit.be/docs/webhooks · https://docs.billit.be/docs/mcp · https://www.billit.eu/nl-be/voor-wie/accountants/ · https://www.billit.eu/nl-be/helpartikelen/mijn-bedrijf/tabblad-bank/koppel-je-bankrekening-en-via-billbank-kbc-coda-gratis/ · https://www.billit.eu/nl-be/helpartikelen/snelle-invoer/bestanden-importerenontvangen/documenten-importeren-via-e-mail/

Bank/PSD2: https://developer.kbc.be/business-solution/psd2-account-information-services · https://developer.kbc.be/news/psd2-validity-refresh-token · https://myponto.com/en/pricing/customer-paying-model/ · https://documentation.isabel.eu/ponto-connect · https://enablebanking.com/terms/ · https://enablebanking.com/docs/markets/be/ · https://docs.firefly-iii.org/tutorials/data-importer/eb/ · https://actualbudget.org/docs/advanced/bank-sync/gocardless/ · https://codabox.com/nl/codabox-business/ · https://www.itsme-id.com/hubfs/Legal%20Information%20-%20B2B%20Website/itsme_apptermsandconditions_en_24-04-24.pdf

Peppol: https://kpmg.com/be/en/insights/my-tax-compass/indirect-tax-insights/e-invoicing-tolerance-of-three-months-announced-by-the-belgian-federal-public-service-finance.html · https://www.deloitte.com/be/en/services/tax/blogs/update-on-belgian-e-invoicing-mandate.html · https://econnect.eu/en/docs/knowledge/peppol/what-is-peppol/smp-and-participant-lookup · https://openpeppol.atlassian.net/wiki/spaces/Belgium/pages/4610949131 · https://recommand.eu/en/pricing · https://docs.peppol.eu/poacc/billing/3.0/syntax/ubl-invoice/cac-PaymentMeans/ · https://globaltaxnews.ey.com/news/2026-1652-belgium-announces-proposed-e-reporting-requirements-beginning-in-2028

Niet-Peppol: https://www.bol.com/be/nl/klantenservice/info/bltcdf36d04d753946c/zakelijk-account · https://partnerplatform.bol.com/nl/idp/factuur-voor-zakelijke-klanten · https://www.amazon.de/gp/help/customer/display.html?nodeId=GB2ABKD25MGSPE6Q · https://docs.business.amazon.com/docs/downloading-invoices-in-eu-and-jp-regions · https://support.google.com/cloud/answer/10311615 · https://platform.claude.com/docs/en/build-with-claude/pdf-support · https://cloud.google.com/document-ai/pricing · https://www.odoo.com/documentation/19.0/applications/finance/accounting/bank/reconciliation.html · https://www.getmyinvoices.com/en/prices/
