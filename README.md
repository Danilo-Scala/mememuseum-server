# Meme Museum — Server

API REST di **Meme Museum**, una community per condividere meme, sviluppata in Node.js con Express 5. Gestisce utenti, meme con immagini, tag, commenti e voti, e serve la Single Page Application del progetto.

![Node.js](https://img.shields.io/badge/Node.js-5FA04E?logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express%205-000000?logo=express&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma%20ORM-2D3748?logo=prisma&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?logo=jsonwebtokens&logoColor=white)

> Progetto universitario sviluppato per il corso di Tecnologie Web all'Università degli Studi di Napoli Federico II.
> Client: [mememuseum-client](https://github.com/Danilo-Scala/mememuseum-client)

---

## Panoramica

Chiunque può sfogliare la bacheca; per pubblicare e interagire serve un account. Il server distingue due livelli di accesso:

| Utente | Cosa può fare |
|---|---|
| **Visitatore** | Sfoglia i meme, li filtra per tag, li ordina per data o per voti, vede il meme del giorno, legge commenti e voti |
| **Utente registrato** | Carica meme con immagine, titolo, descrizione e tag, commenta, vota con upvote o downvote, modifica username, password e immagine del profilo, elimina i propri meme o l'intero account |

## Architettura

```mermaid
flowchart LR
    Client["Client<br/>React 19 · Vite"] -->|REST / JSON con JWT<br/>immagini su /uploads| API["Server<br/>Node.js · Express 5"]
    API -->|Prisma ORM| DB[("PostgreSQL")]
    API -->|Multer| FS[("Cartella uploads/<br/>immagini")]
```

Ogni richiesta segue lo stesso percorso: **route** (un router per risorsa, montati sotto `/api`) → **middleware** (verifica del JWT, upload con Multer) → **controller** (validazione, logica e query con Prisma) → **PostgreSQL**.

## Scelte tecniche principali

- **Autenticazione stateless con JWT.** Login e registrazione restituiscono un token firmato che contiene id e username dell'utente e scade dopo 24 ore. Un middleware lo verifica e rende disponibile l'utente ai controller: le letture sono pubbliche, le scritture richiedono il token.
- **Password con bcrypt.** Le password sono salvate come hash con salt. Il server ne verifica anche la robustezza (almeno 8 caratteri, una maiuscola, un numero e un carattere speciale), sia in registrazione sia nella modifica del profilo.
- **Upload delle immagini con Multer.** Ogni file viene salvato su disco con un nome univoco, accettato solo se è JPEG, PNG, GIF o WEBP e sotto i 5 MB. Le immagini sono poi servite da Express come file statici su `/uploads`.
- **Prisma ORM con migrazioni versionate.** Lo schema del database è definito in `schema.prisma` e le modifiche sono tracciate con Prisma Migrate. Il client Prisma usa il driver adapter per PostgreSQL su un pool di connessioni `pg`.
- **Un voto per utente, garantito dal database.** Il vincolo di unicità su utente e meme impedisce i voti doppi. La stessa chiamata aggiunge il voto, lo cambia se il valore è opposto o lo rimuove se è uguale a quello già espresso.
- **Paginazione e filtri.** Le liste di meme sono paginate (10 per pagina) e restituiscono i metadati per la navigazione (`currentPage`, `totalPages`, `totalItems`). Il filtro per tag non distingue maiuscole e minuscole, e i nuovi tag vengono normalizzati in minuscolo per evitare duplicati.
- **Meme del giorno senza stato.** Il meme viene scelto in base al giorno dell'anno: tutti gli utenti vedono lo stesso meme per l'intera giornata, senza salvare nulla nel database.
- **Controllo della proprietà.** Un utente può eliminare solo i propri meme; in caso contrario il server risponde `403`.

## Modello dei dati

```mermaid
erDiagram
    USER ||--o{ MEME : pubblica
    USER ||--o{ COMMENT : scrive
    USER ||--o{ VOTE : esprime
    MEME ||--o{ COMMENT : riceve
    MEME ||--o{ VOTE : riceve
    MEME }o--o{ TAG : etichettato
```

## API principali

Tutti gli endpoint sono sotto il prefisso `/api`.

| Area | Pubblici | Con autenticazione |
|---|---|---|
| Autenticazione | `POST /auth/register` · `POST /auth/login` | |
| Meme | `GET /memes` (paginata con `?page=`, filtro `?tag=`) · `GET /memes/search` (come sopra, più `?sortBy=` `date_desc`, `date_asc`, `most_upvoted`, `most_downvoted`) · `GET /memes/daily` · `GET /memes/{id}` | `POST /memes` (multipart, con immagine) · `DELETE /memes/{id}` |
| Tag | `GET /tags` (`?search=` per l'autocompletamento) | `POST /tags` |
| Commenti | `GET /comments/{memeId}` | `POST /comments/{memeId}` |
| Voti | `GET /votes/{memeId}` | `POST /votes/{memeId}` (`value`: `1` o `-1`) |
| Profilo | | `GET /users/profile` · `PUT /users/update-profile` (multipart, immagine facoltativa) · `DELETE /users/delete-account` |

Gli endpoint con autenticazione richiedono l'header `Authorization: Bearer <token>`. Le immagini caricate sono disponibili su `GET /uploads/{nomefile}`.

## Avvio in locale

**Prerequisiti:** Node.js 20.19 o superiore, Docker (per PostgreSQL).

1. Clona il repository e installa le dipendenze:

   ```bash
   git clone https://github.com/Danilo-Scala/mememuseum-server.git
   cd mememuseum-server
   npm install
   ```

2. Avvia il database:

   ```bash
   docker run -d --name mememuseum-postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16
   ```

3. Crea il file `.env` partendo dall'esempio (nessuna credenziale è salvata nel repository):

   ```bash
   cp .env.example .env
   ```

   | Variabile | Descrizione |
   |---|---|
   | `DATABASE_URL` | Stringa di connessione a PostgreSQL, ad esempio `postgresql://postgres:postgres@localhost:5432/postgres` |
   | `JWT_SECRET` | Chiave usata per firmare i token. Per generarne una: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
   | `PORT` | Porta del server (facoltativa, predefinita `5000`) |

4. Crea le tabelle e genera il client Prisma:

   ```bash
   npx prisma migrate deploy
   npx prisma generate
   ```

5. Avvia il server:

   ```bash
   npm run dev
   ```

Il server parte su `http://localhost:5000`. Per una verifica veloce, `GET /api/test` risponde con un messaggio di conferma. La cartella `uploads/` per le immagini viene creata automaticamente al primo avvio.

Su macOS la porta 5000 può essere già occupata dal ricevitore AirPlay: in quel caso imposta un'altra porta nel file `.env`, ad esempio `PORT=3000`.

## Struttura del progetto

```
mememuseum-server/
├── index.js               avvio di Express, CORS, file statici e router /api
├── prisma.config.ts       configurazione di Prisma (schema, migrazioni, DATABASE_URL)
├── prisma/
│   ├── schema.prisma      modello dei dati
│   └── migrations/        migrazioni SQL
└── src/
    ├── config/prisma.js   client Prisma con adapter PostgreSQL
    ├── routes/            un router per risorsa, aggregati in routes/index.js
    ├── middleware/        verifica del JWT e upload con Multer
    └── controllers/       logica degli endpoint e query Prisma
```

## Limiti noti e sviluppi futuri

- **Test automatici.** Il server non ha ancora test; il client ha una suite end-to-end con Cypress. Il passo successivo è aggiungere test di integrazione delle API, ad esempio con Jest e Supertest.
- **Ordinamento per voti.** Per gli ordinamenti `most_upvoted` e `most_downvoted` il server carica tutti i meme con i relativi voti, li ordina in JavaScript e poi applica la paginazione. Con pochi dati funziona; con volumi maggiori l'ordinamento andrebbe fatto nel database, con una query aggregata o un contatore dei voti sul meme.
- **Eliminazioni a cascata.** Quando si elimina un meme o un account, voti e commenti collegati vengono cancellati dai controller con query separate, fuori da una transazione. Più robusto definire `onDelete: Cascade` nello schema o usare `prisma.$transaction`.
- **Immagini.** Sono salvate sul disco del server, e quando si elimina un meme o si cambia immagine del profilo il vecchio file resta nella cartella `uploads/`. In produzione andrebbero spostate su uno storage dedicato.
- **Sicurezza.** CORS accetta richieste da qualsiasi origine e non c'è un limite ai tentativi di login. Il passo successivo è restringere CORS all'indirizzo del client e aggiungere un rate limiting sulle rotte di autenticazione.
- **Gestione errori.** Ogni controller gestisce i propri errori, ma gli errori di upload (file troppo grande o formato non supportato) non hanno un gestore dedicato e arrivano al client come errore generico. Un middleware di errore centralizzato renderebbe le risposte uniformi.
- **Containerizzazione.** Un `docker-compose.yml` permetterebbe di avviare server e database con un solo comando.

## Team

Progetto sviluppato da **Danilo Scala** ([@Danilo-Scala](https://github.com/Danilo-Scala)).
