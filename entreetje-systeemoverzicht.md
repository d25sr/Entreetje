# Entreetje — systeemoverzicht en samenvoegplan

Stand: oktober 2026 · gebaseerd op een volledige lezing van `src/`, `supabase/functions` en `supabase/migrations`.

## 1. Wat Entreetje is

Een ticketingplatform met drie kanten:

| Wie             | Wat                                                                          | Account nodig?                                                                     |
| --------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Organisator** | Maakt events, tickettypes, team, scanners; ontvangt uitbetalingen            | Ja (rol `organizer` + organisatie)                                                 |
| **Fan**         | Zoekt events, koopt tickets, toont QR, draagt over of verkoopt door          | Nu: altijd ja. Bedoeling: kopen met alleen e-mail, account alleen voor doorverkoop |
| **Admin**       | Beheert events, imports, bestellingen, uitbetalingen, e-mail, social, fraude | Ja (rol `admin`)                                                                   |

Daaromheen: een **scanner** (deur), een **doorverkoop-marktplaats** (resale met servicekosten) en een **import-/contentmachine** (events van externe bronnen, social posts, e-mailcampagnes, pushmeldingen).

**Belangrijkste bevinding:** gastaankoop met alleen e-mail bestaat nog niet. `create-mollie-organizer-ticket-payment` en `claim-free-organizer-ticket` eisen een ingelogde gebruiker (`requireUser`), en `evenementen.$slug.tsx` toont "Log in om een ticket te kopen". Dit is de grootste afwijking tussen visie en code.

## 2. Techniek in één blik

- **Frontend:** TanStack Start/Router (bestandsroutes in `src/routes`), React 19, Tailwind v4 + eigen CSS per gebied, React Query, shadcn/ui.
- **Backend:** Supabase (Postgres + RLS, 19 RPC's, ~35 edge functions, storage-buckets) en TanStack serverfuncties (`*.functions.ts`, met `requireSupabaseAuth` en de service-role client).
- **Extern:** Mollie (betaling, refunds), Resend (e-mail, inbound webhook), Web Push, social API's (Meta, TikTok), ticketproviders/festivalbronnen voor import.
- **Rollen:** `app_role` = admin / organizer / user. Organisatierollen = owner, admin, eventmanager, finance, marketing, scanner_lead, scanner, guestlist, support.
- **Omvang:** 64 tabellen, 102 migraties.

## 3. Hoe alles gekoppeld is

```
Browser (routes)
  ├─ Publiek: evenementen*, festivals, locaties, organisatoren, SEO-/uitleg-pagina's
  ├─ Account: mijn-tickets, profiel, meldingen, verkoop-flow, onboarding
  ├─ Organisator: /organisator/* (OrganizerControlCenter), /scan, /noodscan
  └─ Admin: /admin/* (AdminShell + hubs)
        │
        ▼
  Supabase client (RLS)  ──►  tabellen / RPC's
  Serverfuncties (.functions.ts)  ──►  service-role: uploads, payouts, listings, admin-status
  Edge functions  ──►  Mollie / Resend / Push / Social
        │
        ▼
  Mollie-webhook ──► order-fulfillment ──► issue_native_ticket ──► ticket_versions (QR)
                                      └─► notificaties + ledger + bevestigingsmail
```

### 3.1 Koop-flow organisator-ticket (primair verdienmodel)

1. Fan opent `/evenementen/$slug` → tickettypes uit `organizer_ticket_types` (event in `organizer_events` + `events`).
2. Knop → edge function `create-mollie-organizer-ticket-payment` → RPC `reserve_organizer_ticket_for_checkout` (voorraad vasthouden, `orders` aanmaken, fees berekenen) → Mollie betaling.
3. Mollie → `mollie-webhook` (haalt betaling opnieuw op bij Mollie) → `order-fulfillment.ts`: RPC `issue_native_ticket` (idempotent) → `tickets` + `ticket_versions` → notificaties, ledger, bevestigingsmail via `email/service.ts`.
4. Vangnet: `reconcile-mollie-payments` en `reconcile-paid-orders` herstellen gemiste webhooks.
5. Gratis tickets: `claim-free-organizer-ticket` (ook account-only).
6. Terugkeerpagina: `/betaling/resultaat?order=…`.

### 3.2 Doorverkoop (resale)

- Verkoper maakt `listings` (+ `listing_sensitive` voor privé-data) via `verkoop-flow` / `/verkopen` en `listings.functions.ts`.
- Koper → `create-mollie-marketplace-payment` → RPC `reserve_listing_for_checkout` → Mollie → zelfde webhook en fulfillment → `transfer_ticket_secure` zet eigenaar om, nieuwe QR-versie (oude QR vervalt).
- Fees: `payments/fee-engine.ts` (koper 5 % + €0,99, verkoper 5 %), uitbetaling na 5 dagen (`payout-policy.ts`).
- Uitbetaling: IBAN-model (`seller_payout_accounts`, `payout_requests`), verwerkt door admin op `/admin/payouts-verwerken`.
- Wachtlijsten (`waitlists`), favorieten (`favorites`).

### 3.3 Scannen

- Scannerapparaten en uitnodigingen: `create-scanner-invite` / `verify-scanner-invite` → `scanner_devices` (token in localStorage).
- Online: `validate-ticket-scan` → RPC `validate_ticket_scan_version` → `ticket_scans`.
- Offline/nood: `/noodscan/$eventId` (cache + later synchroniseren).
- Volledige scanner: `/scan/$eventId`; portaal `/scan`; organisator-tab "Scannen" in het control center.

### 3.4 Organisator-omgeving

`OrganizerControlCenter` rendert werkruimtes (`workspace=…`): overzicht, tickets, bestellingen, bezoekers, resale, scan, financiën, uitbetalingen, marketing, team, branding, instellingen, support, analytics. Dezelfde werkruimtes bestaan zowel globaal (`/organisator/x`) als per event (`/organisator/evenementen/$eventId/x`). Event aanmaken: wizard in `organisator.evenementen.nieuw.tsx`. Team: `organizer-team.ts` + `create-team-invite`. Branding/uploads: `uploads.functions.ts` + `organizer-branding.ts`.

### 3.5 Admin

Hubs in `admin-hubs.ts`: Dashboard · Mensen (gebruikers, organisatoren, rollen) · Ticketing (events, imports, bestellingen, tickets & resale) · Financieel (overzicht, uitbetalingen, geschillen) · Marketing (content, campagnes, inbox, meldingen, templates) · Systeem (integraties, beveiliging, audit, analyse, instellingen). `EventWorkspaceView` wordt hergebruikt voor admin-event-bewerking.

### 3.6 Content- en importmachine

- Import: `import-live-events`, `import-festivals`, `import-event-url`, `event_sources` → `event_candidates` → `publish-event-candidate` → `events`; `lib/event-import` doet normaliseren en ontdubbelen.
- Social: `generate-social-drafts` → `publish-social-post` / `process-social-publish-queue`, `manage-social-settings`, TikTok OAuth.
- E-mail: `manage-email-campaigns`, `process-email-campaigns`, `process-email-queue`, `manage-email-inbox`, `resend-inbound-webhook`, templates in `_shared/email`.
- Push/meldingen: `subscribe-web-push`, `send-web-push-notification`, `process-web-push-queue`, tabel `notifications`.

## 4. Wat dubbel of overbodig is — samenvoegvoorstel

Volgorde = impact. Per punt: _bevinding → voorstel_.

### A. Functioneel (grootste winst)

1. **Gastaankoop ontbreekt** → één `checkout` met e-mail + naam; ticket wordt aan e-mail gekoppeld (magic link / mijn-tickets via e-mail-verificatie). Account pas verplicht bij doorverkoop. Raakt: beide Mollie-checkout-functies, `claim-free-organizer-ticket`, `orders.buyer_id` (nullable maken + `buyer_email`), `issue_native_ticket`, `mijn-tickets`.
2. **Twee checkoutfuncties** (`…organizer-ticket-payment` en `…marketplace-payment`) delen ~80 % code → één `create-checkout` met `source: organizer | resale`. De webhook en fulfillment zijn al gedeeld.
3. **Twee gratis/betaald-paden** (`claim-free-…` en betaald) → in `create-checkout`: bedrag 0 = direct uitgeven.

### B. Scanner (vier ingangen)

`/scan`, `/scan/$eventId`, `/noodscan/$eventId`, organisator-tab "Scannen" + `OrganizerScanner.tsx` → **één scanner** (`/scan/$eventId`) met een offline-modus-schakelaar; `/scan` wordt de eventkeuze; de tab in het control center linkt alleen door. `/noodscan` blijft als redirect.

### C. Routes per event versus globaal

Alle organisator-werkruimtes bestaan twee keer (`/organisator/x` en `/organisator/evenementen/$eventId/x`, 13 + 13 bestanden van ~10 regels). → Eén route met optionele `eventId` search-param of `$eventId?`-segment; scheelt ~20 bestanden en verwarring in navigatie.
Zelfde voor `organisator.tickets`/`marketing`/`resale`/`financien`.

### D. Database — dubbele of ongebruikte tabellen

| Tabellen                                                                             | Bevinding                                                                                     | Voorstel                                                                                                                          |
| ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `listings` ↔ `resale_listings` (+ `resale_transactions`)                             | `resale_listings` wordt in `src/` 1×, in functions 0× gebruikt; `listings` is het levende pad | Kies `listings`; `resale_*` archiveren of droppen na datacheck                                                                    |
| `ticket_scans` ↔ `ticket_validations`                                                | `ticket_validations` niet gebruikt in `src/`                                                  | Samenvoegen in `ticket_scans`                                                                                                     |
| `marketplace_payouts` ↔ `payout_requests` (+ `payout_request_orders`)                | Twee uitbetalingsmodellen (route-model en IBAN-model)                                         | IBAN-model behouden; `marketplace_payouts` en `process-seller-payout-routes` afbouwen; `payout_request_orders` ongebruikt in code |
| `provider_webhook_events`, `fraud_cases`                                             | Geen enkele codeverwijzing                                                                    | Gebruiken (beveiligingspagina) of verwijderen                                                                                     |
| `tickets` ↔ `ticket_versions` ↔ `ticket_ownership_history` ↔ `ticket_status_history` | Bewust gescheiden (historie), maar `tickets` bevat ook oude kolommen van vóór native tickets  | Kolommen opschonen; één bron voor "huidige eigenaar"                                                                              |
| `events` ↔ `organizer_events`                                                        | Twee tabellen voor "een event"; admin- en publieke code moeten beide kennen                   | Op termijn: `organizer_events` als extensie van `events` (1-op-1) met één view                                                    |

### E. Mollie — oude Connect-flow

`mollie-connect-callback`, `start-mollie-onboarding`, `sync-mollie-onboarding`, `_shared/mollie-connect.ts`, `_shared/mollie-eligibility.ts`, `process-seller-payout-routes` en de velden `seller_mollie_not_ready` in `payout-policy.ts` horen bij het oude model; de twee checkoutfuncties zeggen zelf "No Mollie Connect check any more". → Verwijderen na bevestiging dat er geen actieve Connect-accounts meer zijn; `canRouteSellerPayout` vervangen door een IBAN-gate.

### F. Admin

- `admin.payouts` en `admin.payouts-verwerken`: twee pagina's voor uitbetalingen → één.
- `admin.orders`, `admin.tickets`, `admin.support` (geschillen), `admin.security` overlappen rond refunds/fraude → hub "Bestellingen" met tabs.
- `admin.email-campaigns`, `email-inbox`, `email-templates`, `notifications`: vier ingangen voor communicatie → één hub "Communicatie".
- `admin.providers` en `admin.imports` → één "Bronnen".

### G. Publieke pagina's

`verkopen`, `tickets-verkopen`, `voor-organisatoren`, `gids.ticketmaster-verkopen`, `hoe-werkt-het`, `vergelijking`, `over-entreetje`, `support`, `contact` zijn deels verkoop-/uitlegpagina's met overlappende teksten. → Twee landingen houden: **voor fans** (`tickets-kopen` + `tickets-verkopen`) en **voor organisatoren** (`voor-organisatoren`); `hoe-werkt-het`, `vergelijking` en de gids als secties/SEO-pagina's eronder. `/verkopen` (publiek) en `/verkoop-flow` (account) zijn twee stappen van dezelfde flow.

### H. Evenementen-zoek-routes

12 bijna identieke routes (`vandaag`, `morgen`, `dit-weekend`, `volgend-weekend`, `deze-week`, `deze-maand`, `stad/…`, `genre/…`, `categorie/…`). → Eén `evenementen` met filters in de URL; SEO-pagina's genereren vanuit dezelfde component (is al deels zo via `events-discovery.ts`).

### I. Meldingen

`notifications` (in-app), web push, e-mail (`email_deliveries`), organisatorsupport: drie kanalen met eigen logica. → Eén `notify(event, user, kanalen)` met voorkeuren (`email-preferences.ts`, `notification-taxonomy.ts` bestaan al) zodat nieuwe meldingen niet drie keer worden gebouwd.

## 5. Voorgestelde volgorde

1. **Gastaankoop** (A1–A3): grootste productwinst en raakt de checkout toch.
2. **Mollie Connect opruimen + payout-tabellen** (D, E): minder risico en minder verwarring bij uitbetalingen.
3. **Scanner naar één ingang** (B).
4. **Organisator-routes ontdubbelen** (C) en **admin-hubs** (F).
5. **Publieke pagina's en zoekroutes** (G, H).
6. **Eventmodel `events`/`organizer_events`** (D, laatste, grootste migratie).

Elke stap apart uitrollen met een redirect van de oude route, zodat links, SEO en bookmarks blijven werken.

## 6. Openstaande infrastructuur (niet code)

- Migraties toepassen: `20261003120000_organizer_team_access`, `20261004120000_organizer_branding_bucket`, `20261004130000_organizer_publish_requires_iban`, `20261005120000_user_avatars_bucket`.
- Edge functions deployen: `create-team-invite`, `create-mollie-organizer-ticket-payment`.
- Supabase Auth magic-link-template met `{{ .Token }}`; `SUPABASE_SERVICE_ROLE_KEY` in Vercel.

## 7. Kanttekening bij dit document

Het is gebaseerd op code, niet op productiedata. Of een tabel écht ongebruikt is (`resale_listings`, `ticket_validations`, `marketplace_payouts`, `fraud_cases`) moet eerst met een rijtelling in de database bevestigd worden voordat iets wordt verwijderd.
