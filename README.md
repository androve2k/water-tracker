# 💧 Contatore Acqua

**🔗 Demo live: [roversia.it/apps/acqua.html](https://roversia.it/apps/acqua.html)**

Mini web app per tracciare l'acqua bevuta ogni giorno, con obiettivo personalizzabile, streak e grafico settimanale. Account basato su **username + PIN** (niente email), dati sincronizzati su Firebase Realtime Database.

Fa parte della suite di app di [roversia.it](https://roversia.it/apps.html), il mio sito personale — costruito interamente in **HTML/CSS/JS vanilla, zero framework, zero build step**.

---

## Stack tecnico

- **Frontend:** HTML + CSS + JavaScript puro (nessuna dipendenza, nessun bundler)
- **Backend:** Firebase Realtime Database via **REST API** (niente SDK Firebase)
- **Auth:** username + PIN, hash **PBKDF2-SHA256** (150.000 iterazioni) calcolato client-side con Web Crypto API — il PIN non lascia mai il browser in chiaro
- **Sessione:** `localStorage`, scadenza 30 giorni
- **Scritture atomiche:** pattern **ETag + `If-Match`** con retry fino a 4 tentativi su conflitto (HTTP 412), per evitare race condition quando lo stesso utente scrive da più tab/dispositivi
- **UI ottimistica:** l'interfaccia si aggiorna subito al tocco, poi conferma (o rollback) in base alla risposta del server

## Struttura dati (Realtime Database)

```
apps_users/
  {username}/ { s: saltB64, h: hashB64, c: timestamp }

apps_water/
  {username}/
    goal: 2000
    days/
      2026-07-02: 1750
```

## Regole di sicurezza RTDB

```json
{
  "rules": {
    "apps_users": {
      "$u": {
        ".read": true,
        ".write": "!data.exists()",
        ".validate": "$u.matches(/^[a-z0-9_]{3,20}$/) && newData.hasChildren(['s','h','c']) && newData.child('h').val().length === 44 && newData.child('s').val().length === 24"
      }
    },
    "apps_water": {
      "$u": {
        ".read": true,
        ".write": true,
        "goal": { ".validate": "newData.isNumber() && newData.val() >= 500 && newData.val() <= 6000" },
        "days": {
          "$d": { ".validate": "$d.matches(/^\\d{4}-\\d{2}-\\d{2}$/) && newData.isNumber() && newData.val() >= 0 && newData.val() <= 20000" }
        },
        "$other": { ".validate": false }
      }
    }
  }
}
```

## Limite noto (by design)

Senza un backend che verifichi il PIN server-side, le regole RTDB non possono legare la scrittura all'identità: chiunque conosca uno username potrebbe scrivere sui suoi dati via REST diretta. Accettabile per dati non sensibili come i millilitri d'acqua bevuti. Upgrade path valutato: una Netlify Function che verifica l'hash e firma un token HMAC (stesso pattern usato in un'altra app della suite per l'auth cross-app), con regole RTDB basate su custom auth.

Documentare esplicitamente i trade-off di sicurezza, invece di nasconderli, è una scelta voluta.

## Eseguirla in locale

1. Crea un progetto Firebase gratuito → attiva Realtime Database
2. Pubblica le regole di sicurezza qui sopra
3. In `index.html`, sostituisci `RTDB_URL` con l'URL del tuo progetto
4. Apri `index.html` in un browser (nessun server richiesto — è tutto statico)

> Nota: alcuni path assoluti (`/manifest.json`, `/lang.js`, `/main.js`, `/sw.js`, immagini) assumono la struttura di roversia.it. Per un deploy standalone (es. GitHub Pages) vanno adattati o rimossi.

## Perché questa repo

Questo codice è estratto dalla suite di app su [roversia.it/apps.html](https://roversia.it/apps.html), pubblicato come esempio di:
- autenticazione custom senza dipendere da provider esterni
- uso di Firebase via REST invece che via SDK (più controllo, meno peso)
- gestione di scritture concorrenti senza un vero backend

---

Andrea Roversi · Liguria, Italia · [roversia.it](https://roversia.it)
