# Synaptex — Log modifiche sito (Lovable + Brevo + Stripe)

> Documento di riferimento per il vault di cowork.
> Sessione di lavoro: giugno 2026. Stack: **Lovable** (front-end + Lovable Cloud/Supabase), **Brevo** (email marketing + transazionali), **Resend** (email modulo contatti), **Stripe** (pagamenti).

---

## 0. Architettura e stack (da sapere prima di toccare qualsiasi cosa)

| Pezzo | Cosa fa | Dove |
|---|---|---|
| **Lovable** | Editor del sito (front-end + funzioni server) | app Lovable |
| **Lovable Cloud** | È un **Supabase** sotto il cofano (database + auth) | Connectors → Lovable Cloud → View backend |
| **Brevo** | Newsletter (campagne) + email transazionali admin + email di sistema (signup, reset password) | app.brevo.com — lista "Newsletter Synaptex" = **list ID 11** |
| **Resend** | Solo email del **modulo contatti** | resend.com |
| **Stripe** | Pagamenti (Embedded Checkout) + coupon/codici promo | dashboard.stripe.com |

**Framework:** app TanStack (rotte in `src/routes/`, funzioni server con `createServerFn`).

**Pattern di sicurezza ricorrente (importante):**
- Le rotte sotto `/admin/*` sono protette dal layout `src/routes/admin.tsx` (redirect se non loggato o non admin).
- Ogni funzione server passa per `requireSupabaseAuth` (valida il JWT) + `ensureAdmin(userId)` (controlla `role = 'admin'`).
- Le tabelle sensibili hanno **RLS** (Row Level Security) con `is_admin(auth.uid())`.
- Le chiavi segrete (`BREVO_API_KEY`, service role Supabase) vivono **solo lato server**, mai nel browser.

---

## 1. Promo LAUNCH30 con scadenza automatica

**Obiettivo:** dare una scadenza alla promo (30% di sconto, codice `LAUNCH30`), con countdown e sparizione automatica.

### File chiave
- `src/lib/promo-constants.ts` → **unica fonte di verità**:
  - `PROMO_DEADLINE = "2026-06-08T00:00:00+02:00"` (mezzanotte tra dom 7 e lun 8 giugno, ora italiana CEST)
  - `POSTI_DISPONIBILI = 3`
  - `isPromoActive(now)` → confronta ora corrente con la deadline
- `src/components/LaunchPromoBanner.tsx` → banner sulla pagina **pricing**
- `src/components/PromoCountdown.tsx` → countdown sulla **landing page** (giorni/ore/minuti/secondi, aggiornamento ogni secondo, frase "Affrettati! La promo scade tra:")
- `src/components/PromoUrlGuard.tsx` → se arriva `?promo=...` dopo la scadenza, mostra toast "Promozione scaduta" e pulisce l'URL
- `src/lib/payments.functions.ts` → checkout

### Come funziona (3 livelli di protezione)
1. **Front-end:** dopo `PROMO_DEADLINE` i componenti fanno `return null` → banner, countdown, codice e "posti disponibili" spariscono da soli.
2. **URL / checkout:** `allow_promotion_codes: isPromoActive()` nasconde il campo codice in Stripe; guard server-side rifiuta `LAUNCH30` se `!isPromoActive()`.
3. **Stripe (manuale):** impostare la scadenza del coupon nella dashboard Stripe → Prodotti → Codici promozionali → `LAUNCH30` → Expires at = 8 giugno 2026 00:00 (ora IT) / 7 giugno 22:00 UTC.

### Per cambiare la promo in futuro
Basta modificare le due costanti in `promo-constants.ts` (`PROMO_DEADLINE` e `POSTI_DISPONIBILI`). Tutto il resto si aggiorna da solo.

---

## 2. Dove finiscono i dati raccolti (mappa marketing)

### Iscrizioni / Account → tabella `profiles`
Colonne: `id, email, full_name, university, role, subscription_status, trial_started_at, trial_ends_at, trial_course_id, trial_ai_requests, is_max_plan, is_founding_member, gdpr_consent_at, deletion_requested_at, created_at`.
- L'**università** viene scritta dal trigger DB `handle_new_user()` leggendo `raw_user_meta_data->>'university'` passato in `src/routes/register.tsx` (signUp metadata).

### Newsletter → tabella `newsletter_subscribers` **+ Brevo**
Colonne: `id, email, source, consent_at, created_at, university, topics (array)`.
- Salvataggio in `src/lib/newsletter.functions.ts` (server function `subscribeNewsletter`, upsert su `email`).
- Replica su Brevo via `upsertBrevoContact` (lista ID 11, attributi `UNIVERSITA`, `TOPICS`).
- `topics` = preferenze argomenti (es. `bandi`, `quiz`, `studio`).

### Privacy
- RLS su tutto: `profiles` leggibile solo dal proprietario o dagli admin; `newsletter_subscribers` solo admin.
- Consenso tracciato: `gdpr_consent_at` (profiles) e `consent_at` (newsletter).
- **Regola d'oro GDPR:** marketing solo a chi ha consenso newsletter. Agli utenti trial/registrati → solo comunicazioni **di servizio**.

---

## 3. Dashboard marketing — `/admin/marketing`

**File:** `src/lib/marketing-admin.functions.ts` (server function `getMarketingStats` + `syncNewsletterWithBrevo`), `src/routes/admin.marketing.tsx`, voce in `src/components/AdminSidebar.tsx`.

### Cosa contiene
- **Card riepilogo:** totale registrati, totale newsletter, nuovi ultimi 7/30 giorni.
- **Grafico nel tempo:** linee registrazioni vs newsletter, selettore 7/30/90 giorni (timeseries riempita di zeri per i giorni vuoti).
- **Classifica università** (da `profiles` e da `newsletter_subscribers`) con grafico a barre + tabella.
- **Interessi newsletter:** conteggio per topic (bandi/quiz/studio).
- **Tabella iscritti newsletter** con ricerca + ordinamento.
- **Filtri combinati** (università + topics multi + intervallo date) con conteggio risultati e **export CSV dei soli risultati filtrati** (UTF-8 BOM per Excel).
- **Sync Brevo:** pulsante che fa `upsertBrevoContact` per tutti gli iscritti (gestione errori per-contatto, toast riassuntivo `total/synced/errors`).

### Come la uso
Login admin → sidebar "Marketing". Sola lettura (tranne il pulsante Sync, che scrive solo su Brevo).

> Nota futura: sync e invii girano **in sequenza** (un contatto alla volta). Con liste molto grandi (1000+) valutare invio in batch per non toccare i rate limit di Brevo.

---

## 4. Sezione invio email — `/admin/email`

**File:** `src/lib/admin-email.functions.ts` (server functions protette da `ensureAdmin`), `src/routes/admin.email.tsx`, tabella `admin_email_sends` (RLS admin-only), `src/lib/brevo.server.ts` (aggiunte `sendBrevoTransactional` e `createAndSendBrevoCampaign`), voce "Comunicazioni" in sidebar.

### Due gruppi destinatari, due regole diverse
| Gruppo | Tipo invio Brevo | Regole |
|---|---|---|
| **Iscritti newsletter** | **Campagna** (`createAndSendBrevoCampaign`, lista 11) | Marketing OK. Brevo aggiunge footer di **disiscrizione** obbligatorio. |
| **Utenti trial/registrati** (`profiles`) | **Transazionale** (`sendBrevoTransactional`) | **Solo comunicazioni di servizio**, NON promozionali. Footer di servizio iniettato automaticamente. Skip degli indirizzi in `suppressed_emails`. |

### Funzioni di sicurezza
- Selezione **un gruppo alla volta** (per non mischiare consensi).
- **Email di test** a un indirizzo a scelta prima dell'invio reale.
- **Anteprima + conferma** ("Confermo l'invio a N destinatari").
- **Storico invii** nella tabella `admin_email_sends` (status `sent / partial / failed`).

### ⚠️ Bug noto (NON ancora risolto)
Il campo **"Corpo (HTML semplice consentito)"** non interpreta l'HTML grezzo incollato: invia una **mail in testo semplice**. 
- **Workaround usato:** scrivere la mail in **testo semplice** (no tag HTML) e incollarla direttamente.
- **Fix futuro:** far passare il corpo a Brevo come `htmlContent` senza escaping, oppure aggiungere una modalità "Codice HTML" nell'editor + anteprima renderizzata.

---

## 5. Fix mittente email (noreply → info)

**Problema:** le mail non arrivavano. Causa: il mittente `noreply@synaptex.it` **non era verificato in Brevo** → Brevo le scartava (l'app diceva "inviato" ma non arrivava nulla).

**Soluzione:** sostituito ovunque `noreply@synaptex.it` → `info@synaptex.it` (mittente verificato).

File toccati:
- `src/lib/admin-email.functions.ts` (costante `SENDER.email`, riga ~23; `REPLY_TO_EMAIL = process.env.CONTACT_EMAIL || SENDER.email`)
- `src/lib/contact.functions.ts` (from per Resend, riga ~32)
- `src/lib/email/templates.server.ts` (footer)
- `src/routes/lovable/email/auth/webhook.ts` (email di sistema: **signup, reset password, magic link**, riga ~180)

**Scoperta collaterale importante:** il vecchio `noreply@` era usato anche per le **email di sistema** → conferma registrazione e reset password **probabilmente non arrivavano** agli utenti finché non l'abbiamo corretto.

### Come diagnosticare problemi email in futuro
1. App dice "inviato" ma non arriva → **non è il codice**, è Brevo/consegna.
2. Controlla **spam**.
3. Brevo → **Transactional → Logs**: stato `Delivered` (→ è spam), `Blocked/Invalid sender` (→ mittente non verificato), o **non compare** (→ API key / chiamata).
4. Brevo → Impostazioni → **Mittenti**: l'indirizzo `from` deve essere **verde "Verificato"**.

---

## 6. Deliverability — TODO consigliato

- **Autenticare il dominio `synaptex.it`** in Brevo (record **SPF/DKIM** verdi) e in **Resend** → le mail arrivano in posta in arrivo invece che spam.
- Finché non fatto, le mail (soprattutto campagne) rischiano lo spam.
- **Mai** usare un mittente `@gmail.com` per invii massivi (policy DMARC di Gmail → spam/rifiuto).

---

## 7. Stato / cose in sospeso

- [ ] **Confermare scadenza coupon LAUNCH30 su Stripe** (manuale in dashboard) — il sito sparisce da solo, ma il coupon va chiuso anche lato Stripe.
- [ ] **Fix bug HTML** nel campo Corpo di `/admin/email`.
- [ ] **Autenticazione dominio** SPF/DKIM (Brevo + Resend).
- [ ] (Opzionale) Invii email in **batch** se le liste crescono molto.
- [ ] (Opzionale) Allegato immagine / programmazione invio nella sezione email.

---

## 8. Reference rapida

- **Backend dati:** Lovable → Connectors → Lovable Cloud → View backend (Table Editor su `profiles`, `newsletter_subscribers`, `admin_email_sends`).
- **Newsletter / campagne:** app.brevo.com → lista "Newsletter Synaptex" (ID 11).
- **Pagamenti / coupon:** dashboard.stripe.com (modalità **Live**).
- **Pannelli sito (solo admin):** `/admin`, `/admin/marketing`, `/admin/email`.
- **Costanti promo:** `src/lib/promo-constants.ts`.
- **Mittente email ufficiale:** `info@synaptex.it`.
