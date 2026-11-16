# 🏦 Banca 1°H — BANCA COMPLETAMENTE FINTA

> ## ⚠️ ATTENZIONE: È TUTTO FINTO
> **Banca 1°H non è una banca vera.** È un gioco / progetto scolastico di fantasia.
>
> - Gli **Alessandri (₳)** sono una valuta **inventata**: **non sono soldi veri** e **non hanno nessun valore**.
> - Non si può depositare, prelevare, pagare o ricevere **nulla di reale**.
> - Nessun conto, bonifico o truffa è reale: sono tutti **simulazioni**.
> - **Non inserire dati personali veri** (nome, cognome, email, telefono, indirizzo…).
> - **Non usare password che usi altrove.** Inventane una apposta per questo gioco.
> - Questo progetto non ha alcun legame con banche, enti o persone reali.

---

## Cos'è

Un sito web (un solo `index.html`) che simula una piccola banca per gioco, pubblicabile su **GitHub Pages** e collegato a **Firebase** (Authentication + Firestore).

### Regole del gioco

| Cosa | Come funziona |
|---|---|
| Registrazione | Nickname + password **rigida** |
| Regalo iniziale | **1.000 Alessandri** all'iscrizione |
| Bonus periodico | **+30 Alessandri ogni 3 giorni** (si riceve in automatico quando apri il sito; se manchi per più periodi, li recuperi) |
| Bonifici | Scrivi il nickname di un altro utente e l'importo: **numero intero, da 1 a 100.000 ₳**, e solo se hai abbastanza saldo |
| Segnalare una truffa | Dal tuo conto scrivi una segnalazione all'amministratore |
| Amministratore | **Uno solo**. Vede tutto, aggiunge/toglie soldi, esamina le segnalazioni |

### Password rigida
Almeno 12 caratteri, con minuscola, maiuscola, numero e simbolo; niente spazi, niente 3 caratteri uguali di fila, niente sequenze ovvie (`abcd`, `1234`, `qwerty`), niente parole comuni e non deve contenere il nickname. Il sito mostra la lista in tempo reale e non ti lascia registrare finché non è tutto a posto.

> 🔐 **Dove viene salvata la password?** Non viene scritta nel database (Firestore) in chiaro. La gestisce **Firebase Authentication**, che la conserva in forma protetta (hash). È il modo corretto di farlo: nemmeno l'amministratore può leggerla.

### Segnalazioni di truffa
1. Un utente invia una segnalazione: nickname di chi l'ha truffato, importo perso, descrizione (può partire anche dal bottone "Segnala truffa" accanto a un bonifico inviato).
2. L'amministratore la esamina (vede i bonifici tra i due account, il saldo del segnalato, ecc.) e può:
   - **Accettare**: sposta soldi dal truffatore al segnalante (fino al saldo disponibile del truffatore) + un **risarcimento in più** pagato dalla banca (importi modificabili).
   - **Rifiutare**: con **addebito di 30 ₳** per il disturbo, **oppure nessun addebito**, oppure un altro importo.
3. L'utente vede l'esito (e la nota dell'admin) nella sua area.

### Cosa può fare l'amministratore
- Vedere **tutti i movimenti** di **tutti** gli utenti (con filtri per nickname e tipo)
- **Accreditare**, **togliere** soldi o fare un **addebito** a qualsiasi utente (mai sotto zero; ogni operazione compare nei movimenti)
- Gestire le segnalazioni
- Panoramica con utenti, Alessandri in circolazione e segnalazioni in attesa

---

## File del progetto

| File | A cosa serve |
|---|---|
| `index.html` | Tutto il sito (grafica + logica) |
| `README.md` | Questo file (contiene anche le regole di sicurezza da incollare in Firebase) |

---

## Configurazione (una volta sola)

Progetto Firebase già collegato: **`banchina`**.

### 1) Attiva l'accesso con Email/Password
Console Firebase → **Authentication** → **Metodo di accesso** → **Email/Password** → **Abilita** → Salva.
*(Il sito usa email finte tipo `nickname@banca1h.example` solo come "nome utente" tecnico: nessuna email vera viene inviata o richiesta.)*

### 2) Crea il database
Console Firebase → **Firestore Database** → **Crea database** → scegli una regione europea (es. `eur3`) → **modalità produzione**.

### 3) Pubblica le regole di sicurezza
Firestore → scheda **Regole** → cancella tutto, incolla il blocco qui sotto → **Pubblica**.

> ⚠️ **Questo passaggio è obbligatorio.** Senza queste regole il sito non funziona (o, se lasci le regole aperte, chiunque può cambiarsi il saldo e creare l'admin senza codice).
> Se la console segnala un errore su `hashing.sha256`, leggi la sezione "Problemi" più sotto.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ---------- funzioni di utilità ----------
    function signedIn() { return request.auth != null; }

    function isAdmin() {
      return signedIn()
        && exists(/databases/$(database)/documents/meta/admin)
        && get(/databases/$(database)/documents/meta/admin).data.uid == request.auth.uid;
    }

    function changed() {
      return request.resource.data.diff(resource.data).affectedKeys();
    }

    // la transazione "id" viene creata proprio in questa scrittura (anti-replay)
    function newTx(id) {
      return !exists(/databases/$(database)/documents/transactions/$(id))
        && existsAfter(/databases/$(database)/documents/transactions/$(id));
    }
    function txAfter(id) {
      return getAfter(/databases/$(database)/documents/transactions/$(id)).data;
    }

    // ---------- NICKNAME (serve a cercare chi riceve un bonifico) ----------
    match /nicknames/{nick} {
      allow get: if signedIn();
      allow list: if false;
      allow create: if signedIn()
        && nick.matches('^[a-z0-9_]{3,16}$')
        && request.resource.data.keys().hasOnly(['uid', 'role'])
        && request.resource.data.uid == request.auth.uid
        && (
          (request.resource.data.role == 'user'
            && existsAfter(/databases/$(database)/documents/users/$(request.auth.uid)))
          ||
          (request.resource.data.role == 'admin'
            && existsAfter(/databases/$(database)/documents/meta/admin)
            && getAfter(/databases/$(database)/documents/meta/admin).data.uid == request.auth.uid)
        );
      allow update, delete: if false;
    }

    // ---------- STATO PUBBLICO (esiste già un amministratore?) ----------
    match /meta/status {
      allow get: if true;
      allow list: if false;
      allow create: if signedIn()
        && request.resource.data == {'adminExists': true}
        && existsAfter(/databases/$(database)/documents/meta/admin)
        && getAfter(/databases/$(database)/documents/meta/admin).data.uid == request.auth.uid;
      allow update, delete: if false;
    }

    // ---------- ACCOUNT AMMINISTRATORE (UNO SOLO) ----------
    // Il documento si può creare una volta sola (se esiste già, la creazione fallisce).
    // Il codice segreto NON è scritto in chiaro da nessuna parte: qui c'è solo il suo hash SHA-256.
    // Se la console segnala un errore sulla riga "hashing.sha256", vedi il README (sezione "Problemi").
    match /meta/admin {
      allow get: if signedIn() && resource.data.uid == request.auth.uid;
      allow list: if false;
      allow create: if signedIn()
        && request.resource.data.keys().hasOnly(['uid', 'code', 'createdAt'])
        && request.resource.data.uid == request.auth.uid
        && request.resource.data.createdAt == request.time
        && request.resource.data.code is string
        && hashing.sha256(request.resource.data.code.toUtf8()).toHexString()
             == '245541d9d16b383489abf9947b3f6b18fc362c04f74889555e729964e25e75bc'
        && existsAfter(/databases/$(database)/documents/meta/status);
      allow update, delete: if false;
    }

    // ---------- UTENTI (saldo) ----------
    match /users/{uid} {
      allow get: if signedIn() && (request.auth.uid == uid || isAdmin());
      allow list: if isAdmin();

      // registrazione: 1000 Alessandri di partenza, niente di più
      allow create: if request.auth.uid == uid
        && request.resource.data.keys().hasOnly(['nickname', 'balance', 'bonusClaims', 'createdAt', 'lastBonusAt'])
        && request.resource.data.keys().hasAll(['nickname', 'balance', 'bonusClaims', 'createdAt', 'lastBonusAt'])
        && request.resource.data.balance == 1000
        && request.resource.data.bonusClaims == 0
        && request.resource.data.createdAt == request.time
        && request.resource.data.lastBonusAt == request.time
        && request.resource.data.nickname is string
        && request.resource.data.nickname.matches('^[a-z0-9_]{3,16}$')
        && getAfter(/databases/$(database)/documents/nicknames/$(request.resource.data.nickname)).data.uid == uid
        && getAfter(/databases/$(database)/documents/nicknames/$(request.resource.data.nickname)).data.role == 'user'
        && !isAdmin();

      allow update: if debitOk(uid) || creditOk(uid) || claimOk(uid) || adminAdjustOk();
      allow delete: if false;
    }

    // chi invia un bonifico: il saldo scende solo dell'importo del bonifico
    function debitOk(uid) {
      let n = request.resource.data;
      return request.auth.uid == uid
        && changed().hasOnly(['balance', 'lastDebit'])
        && n.lastDebit is string
        && n.balance >= 0
        && newTx(n.lastDebit)
        && txAfter(n.lastDebit).type == 'transfer'
        && txAfter(n.lastDebit).from == uid
        && resource.data.balance - n.balance == txAfter(n.lastDebit).amount;
    }

    // chi riceve: il saldo sale solo dell'importo di un bonifico appena creato dal mittente
    function creditOk(uid) {
      let n = request.resource.data;
      return request.auth.uid != uid
        && changed().hasOnly(['balance', 'lastCredit'])
        && n.lastCredit is string
        && newTx(n.lastCredit)
        && txAfter(n.lastCredit).type == 'transfer'
        && txAfter(n.lastCredit).to == uid
        && txAfter(n.lastCredit).from == request.auth.uid
        && n.balance - resource.data.balance == txAfter(n.lastCredit).amount
        && getAfter(/databases/$(database)/documents/users/$(request.auth.uid)).data.lastDebit == n.lastCredit;
    }

    // bonus: 30 Alessandri per ogni periodo di 3 giorni trascorso
    function claimOk(uid) {
      let n = request.resource.data.bonusClaims - resource.data.bonusClaims;
      return request.auth.uid == uid
        && changed().hasOnly(['balance', 'lastBonusAt', 'bonusClaims'])
        && n >= 1 && n <= 200
        && request.resource.data.balance == resource.data.balance + 30 * n
        && request.resource.data.lastBonusAt == resource.data.lastBonusAt + duration.value(3 * n, 'd')
        && request.resource.data.lastBonusAt <= request.time;
    }

    // amministratore: può cambiare solo il saldo (mai sotto zero)
    function adminAdjustOk() {
      return isAdmin()
        && changed().hasOnly(['balance'])
        && request.resource.data.balance is int
        && request.resource.data.balance >= 0;
    }

    // ---------- MOVIMENTI ----------
    match /transactions/{txId} {
      allow read: if signedIn() && (isAdmin() || request.auth.uid in resource.data.participants);
      allow create: if signedIn() && (transferOk(txId) || welcomeOk() || bonusTxOk() || adminTxOk());
      allow update, delete: if false;
    }

    function baseShape() {
      let d = request.resource.data;
      return d.keys().hasOnly(['type', 'from', 'fromNick', 'to', 'toNick', 'amount', 'note', 'participants', 'createdAt'])
        && d.keys().hasAll(['type', 'amount', 'note', 'participants', 'createdAt'])
        && d.amount is int && d.amount >= 1 && d.amount <= 1000000
        && d.note is string && d.note.size() <= 140
        && d.createdAt == request.time
        && d.participants is list && d.participants.size() >= 1 && d.participants.size() <= 2;
    }

    // bonifico tra due utenti: importo intero, massimo 100.000 ₳
    function transferOk(txId) {
      let d = request.resource.data;
      return baseShape()
        && d.type == 'transfer'
        && d.from == request.auth.uid
        && d.to is string && d.to != d.from
        && d.amount <= 100000
        && d.participants == [d.from, d.to]
        && getAfter(/databases/$(database)/documents/users/$(d.from)).data.lastDebit == txId
        && getAfter(/databases/$(database)/documents/users/$(d.to)).data.lastCredit == txId
        && get(/databases/$(database)/documents/users/$(d.from)).data.nickname == d.fromNick
        && get(/databases/$(database)/documents/users/$(d.to)).data.nickname == d.toNick;
    }

    function welcomeOk() {
      let d = request.resource.data;
      return baseShape()
        && d.type == 'welcome'
        && d.to == request.auth.uid
        && d.amount == 1000
        && d.participants == [d.to]
        && !exists(/databases/$(database)/documents/users/$(d.to))
        && getAfter(/databases/$(database)/documents/users/$(d.to)).data.balance == 1000;
    }

    function bonusTxOk() {
      let d = request.resource.data;
      return baseShape()
        && d.type == 'bonus'
        && d.to == request.auth.uid
        && d.participants == [d.to]
        && d.amount == getAfter(/databases/$(database)/documents/users/$(request.auth.uid)).data.balance
                       - get(/databases/$(database)/documents/users/$(request.auth.uid)).data.balance
        && getAfter(/databases/$(database)/documents/users/$(request.auth.uid)).data.bonusClaims
             > get(/databases/$(database)/documents/users/$(request.auth.uid)).data.bonusClaims;
    }

    function adminTxOk() {
      return isAdmin()
        && baseShape()
        && request.resource.data.type in ['admin_add', 'admin_remove', 'fee', 'refund', 'compensation'];
    }

    // ---------- SEGNALAZIONI DI TRUFFA ----------
    match /reports/{rid} {
      allow read: if signedIn() && (resource.data.reporterUid == request.auth.uid || isAdmin());

      allow create: if signedIn()
        && request.resource.data.keys().hasOnly(['reporterUid', 'reporterNick', 'scammerNick', 'scammerUid', 'amount', 'description', 'status', 'createdAt'])
        && request.resource.data.keys().hasAll(['reporterUid', 'reporterNick', 'scammerNick', 'scammerUid', 'amount', 'description', 'status', 'createdAt'])
        && request.resource.data.reporterUid == request.auth.uid
        && request.resource.data.status == 'pending'
        && request.resource.data.createdAt == request.time
        && request.resource.data.amount is int
        && request.resource.data.amount >= 1 && request.resource.data.amount <= 1000000
        && request.resource.data.description is string
        && request.resource.data.description.size() >= 10 && request.resource.data.description.size() <= 500
        && request.resource.data.scammerNick is string
        && request.resource.data.scammerNick.matches('^[a-z0-9_]{3,16}$')
        && request.resource.data.scammerNick != request.resource.data.reporterNick
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.nickname == request.resource.data.reporterNick
        && get(/databases/$(database)/documents/nicknames/$(request.resource.data.scammerNick)).data.uid == request.resource.data.scammerUid;

      allow update: if isAdmin()
        && resource.data.status == 'pending'
        && changed().hasOnly(['status', 'resolution', 'resolvedAt'])
        && request.resource.data.status in ['accepted', 'rejected'];

      allow delete: if false;
    }

    // ---------- tutto il resto è vietato ----------
  }
}
```

### 4) Autorizza il dominio di GitHub Pages
Authentication → **Impostazioni** → **Domini autorizzati** → **Aggiungi dominio** → `TUOUSERNAME.github.io`
*(`localhost` e `banchina.firebaseapp.com` ci sono già.)*

### 5) Pubblica su GitHub Pages
1. Crea un repository su GitHub e carica `index.html` e `README.md`.
2. **Settings → Pages → Build and deployment → Deploy from a branch → `main` / `(root)` → Save**.
3. Dopo un minuto il sito è su `https://TUOUSERNAME.github.io/NOME-REPO/`.

---

## 👑 Creare l'account amministratore

- Se ne può creare **uno solo**.
- Nella scheda **Registrati** spunta **"Voglio creare l'account amministratore"** e inserisci il **codice segreto** che ti ha consegnato chi ha creato il progetto.
- Il codice **non è scritto in chiaro in nessun file del sito**: nelle regole c'è solo la sua **impronta cifrata (SHA-256)**, e il controllo lo fa il server Firebase.
- **Non pubblicare mai il codice** su GitHub, nel README o in chat pubbliche.
- L'account amministratore **non ha un saldo** (non è un cliente): serve solo a gestire la banca.

### Cambiare il codice segreto
1. Scegli un nuovo codice lungo e casuale.
2. Calcola la sua impronta (su Mac/Linux): `printf '%s' 'NUOVO-CODICE' | sha256sum`
3. Nelle regole (passo 3) sostituisci l'impronta (la stringa di 64 caratteri) con quella nuova e ripubblica le regole.
4. Il codice va scritto **in MAIUSCOLO** (il sito lo converte da solo).

### Se perdi l'accesso all'amministratore
Console Firebase → Firestore → elimina i documenti `meta/admin`, `meta/status` e `nicknames/<nickname-admin>`; poi in Authentication elimina l'utente admin. Dopo potrai crearne uno nuovo.

---

## Sicurezza e limiti (da sapere)

- Il sito funziona **senza server proprio** (solo Firebase). La sicurezza è affidata alle **regole Firestore**, scritte per impedire i trucchi più comuni: saldi modificabili a mano, bonifici senza addebito, bonus ripetuti, secondo amministratore, movimenti falsi.
- Per un gioco va benissimo, ma **non è un sistema bancario vero** e non va usato per nulla di reale.
- La chiave API nel codice (`apiKey`) **non è un segreto**: è fatta per stare nei siti web. A proteggere i dati sono le regole.
- Ogni persona potrebbe creare più account per prendere più volte i 1.000 ₳ iniziali: è un gioco, non c'è nulla da difendere.
- Il sito include Google Analytics (misurazione visite) perché presente nella configurazione Firebase: se non lo vuoi, togli le righe `getAnalytics` da `index.html`.

## Problemi comuni

| Problema | Soluzione |
|---|---|
| "L'accesso Email/Password non è attivo…" | Passo 1 non fatto |
| "Operazione non consentita dalle regole" | Regole non pubblicate (passo 3) o incollate in modo incompleto |
| Errore di dominio non autorizzato al login | Passo 4 non fatto |
| La console non accetta `hashing.sha256(...)` | Sostituisci quel confronto con `&& request.resource.data.code == 'IL-TUO-CODICE'` (in quel caso incollalo solo nella console Firebase: **non** scriverlo nel README su GitHub) |
| Bonus non arrivato | Si accredita quando apri il sito dopo 3 giorni dall'iscrizione (o dall'ultimo bonus) |

---

**Ricorda: Banca 1°H è finta. Gli Alessandri non valgono niente. Divertiti! 🎮**
