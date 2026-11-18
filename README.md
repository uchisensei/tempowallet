# TempoWallet — PWA installabile con account privato

Webapp installabile come app nativa su iOS e Android, con autenticazione utente e
profilo privato sincronizzato sul cloud. **Nessun build step**: è una PWA statica
(HTML + manifest + service worker) che usa il client Supabase via CDN per auth e
database. Si deploya su Vercel come sito statico in un click.

- **Privata per utente**: ogni utente vede solo i propri dati (Row Level Security di Supabase).
- **Installabile**: aggiungi alla schermata Home su iPhone/Android, si avvia a schermo intero come app.
- **Offline**: il service worker memorizza la shell dell'app; funziona anche senza connessione.
- **Sincronizzata**: il profilo (nome, tariffa, attività, notifiche, tema) si salva sul tuo account.

---

## 1. Crea il progetto Supabase (gratis)

1. Vai su <https://supabase.com> e registrati (free tier, nessuna carta richiesta).
2. **New project** → scegli nome, regione (es. EU Central / Frankfurt), genera una password forte per il DB. Attendi 1-2 minuti.
3. Prendi le credenziali: **Project Settings → API**
   - `Project URL`  (es. `https://abcdexyz.supabase.co`)
   - `anon` public key (una stringa lunga che inizia con `eyJ...`)
4. Lascia abilitata l'autenticazione **Email** (lo è di default):
   *Authentication → Providers → Email → abilitato*.

## 2. Crea la tabella del profilo + le policy di sicurezza

Vai su **SQL Editor → New query**, incolla ed esegui:

```sql
-- Tabella profilo TempoWallet: una riga per utente
create table if not exists public.tempowallet_profiles (
  user_id    uuid primary key references auth.users(id) on delete cascade,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

-- Attiva la sicurezza per riga (RLS): obbligatoria
alter table public.tempowallet_profiles enable row level security;

-- Ogni utente può leggere / inserire / modificare / cancellare SOLO la propria riga
create policy "tw_select_own" on public.tempowallet_profiles
  for select using (auth.uid() = user_id);
create policy "tw_insert_own" on public.tempowallet_profiles
  for insert with check (auth.uid() = user_id);
create policy "tw_update_own" on public.tempowallet_profiles
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "tw_delete_own" on public.tempowallet_profiles
  for delete using (auth.uid() = user_id);
```

Con queste policy, anche se la chiave `anon` è pubblica, nessuno può leggere o
scrivere i dati di un altro utente: il database rifiuta ogni operazione che non
riguardi la riga dell'utente autenticato.

## 3. Configura l'app

Apri **`config.js`** e inserisci le due credenziali del punto 1:

```js
window.SUPABASE_URL = 'https://abcdexyz.supabase.co';
window.SUPABASE_ANON_KEY = 'eyJhbGciOi...la-tua-chiave-anon...';
```

> Se i due campi restano vuoti, l'app funziona comunque in **modalità locale**
> (dati solo su questo dispositivo, nessuna sincronizzazione cloud) — utile per
> provare subito. Appena metti le credenziali, compare la schermata di login.

## 4. Carica su GitHub

1. Crea un nuovo repository su <https://github.com> (pubblico o privato).
2. Copia in un'unica cartella **tutti i file** di `tempowallet-pwa/` (mantenendo i nomi):
   `index.html`, `manifest.webmanifest`, `sw.js`, `config.js`, `vercel.json`,
   `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `icon-32.png`, `README.md`.
3. Da quella cartella:
   ```bash
   git init
   git add .
   git commit -m "TempoWallet PWA"
   git branch -M main
   git remote add origin https://github.com/<tuoutente>/<tuorepo>.git
   git push -u origin main
   ```

> **Nota su `config.js`**: la chiave `anon` è sicura da pubblicare (è pubblica per
> design e protetta da RLS). Se preferisci non versionarla, puoi metterla in
> `.gitignore` e inserirla manualmente dopo il deploy, oppure definirla come
> variabile d'ambiente su Vercel e leggerla a runtime — ma per un uso semplice
> tenerla in `config.js` è il percorso consigliato.

## 5. Deploy su Vercel

1. Vai su <https://vercel.com> e accedi con GitHub.
2. **Add New → Project → Import** del repository appena creato.
3. Configurazione:
   - **Framework Preset**: Other
   - **Build Command**: *(vuoto — nessun build)*
   - **Output Directory**: `.` (oppure lascia vuoto)
   - **Install Command**: *(vuoto)*
4. **Deploy**. In pochi secondi hai un URL `https://<tuo-progetto>.vercel.app` con
   HTTPS (necessario per la PWA e per il service worker).

Aggiornamenti: basta fare `git push` e Vercel ridistribuisce automaticamente.

## 6. Installa come app sul telefono

Apri l'URL Vercel sul telefono:

- **iPhone / iPad (Safari)**: tocca l'icona Condividi → *Aggiungi alla schermata
  Home*. Si avvia a tutto schermo, con icona e senza barra del browser.
- **Android (Chrome / Edge)**: menu ⋮ → *Installa app* (o *Aggiungi alla schermata
  Home*). Compare nella lista app con la propria icona.

L'app resta aggiornata: il service worker scarica la nuova versione al primo
avvio successivo al deploy.

## 7. Come funziona la sincronizzazione

- Al primo accesso ti **registri** con email + password (o accedi se hai già un account).
- Dopo il login, l'app **carica** il tuo profilo da `tempowallet_profiles`.
- Ogni modifica (onboarding, nuova attività, notifica, cambio tema) viene salvata
  subito in locale e **sincronizzata sul cloud** (con un piccolo debounce) via
  `upsert` sulla tua riga.
- Il `user_id` è preso dalla sessione Supabase: non puoi toccare righe altrui.
- Il pulsante **Esci dall'account** (in Impostazioni) fa logout; i dati restano
  sul cloud e vengono ricaricati al prossimo login, anche da un altro telefono.

## Prova in locale

Apri `index.html` in un browser, oppure servi la cartella con un piccolo server
necessario per il service worker / manifest (il protocollo `file://` non registra
lo SW):

```bash
cd tempowallet-pwa
python3 -m http.server 8080
# apri http://localhost:8080
```

Senza `config.js` compilato vedi la modalità locale. Per testare il login devi
aver creato il progetto Supabase (passaggi 1-3).

## File del progetto

| File | Ruolo |
| --- | --- |
| `index.html` | L'app completa (UI, logica, auth, sync) |
| `manifest.webmanifest` | Definizione PWA (nome, icone, display standalone) |
| `sw.js` | Service worker: cache offline dell'app shell |
| `config.js` | Credenziali Supabase (URL + chiave anon) |
| `vercel.json` | Header hosting (Cache-Control, Service-Worker-Allowed) |
| `icon-*.png` | Icone app (192, 512 maskable, 180 apple-touch, 32) |
