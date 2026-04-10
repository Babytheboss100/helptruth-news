# HelpTruth — Claude Code Prosjektkontekst

## Hva er dette?
HelpTruth er en norsk Twitter/X-kopi bygget med Node.js backend og React frontend.
All kode er skrevet og fungerer. Du skal nå utvide den med nye features.

---

## Prosjektstruktur

```
helptruth/
├── backend/
│   ├── server.js                  ← Express-server, port 4000
│   ├── package.json
│   ├── Procfile                   ← Railway deploy
│   ├── railway.toml
│   ├── .env.example
│   ├── db/
│   │   ├── pool.js                ← PostgreSQL-kobling (støtter DATABASE_URL)
│   │   └── schema.sql             ← Alle tabeller + testdata
│   ├── middleware/
│   │   └── auth.js                ← JWT-verifisering
│   └── routes/
│       ├── auth.js                ← POST /register, POST /login, GET /me
│       ├── posts.js               ← Feed, opprett, slett, like, repost, bokmerk, svar
│       └── users.js               ← Profil, følg, søk, oppdater
└── frontend/
    ├── package.json
    ├── vercel.json
    ├── .env.local                 ← REACT_APP_API_URL=http://localhost:4000/api
    ├── .env.production
    ├── public/
    │   └── index.html
    └── src/
        ├── index.js
        ├── api.js                 ← Alle fetch-kall til backend
        └── App.jsx                ← HELE applikasjonen (én fil)
```

---

## Tech Stack

| Del | Teknologi |
|-----|-----------|
| Frontend | React 18, CSS-in-JS (inline styles) |
| Backend | Node.js 18+, Express 4 |
| Database | PostgreSQL 15 |
| Auth | JWT (jsonwebtoken) + bcryptjs |
| Deploy backend | Railway |
| Deploy frontend | Vercel |

---

## Database — eksisterende tabeller

```sql
users        (id, name, handle, email, password, bio, avatar, avatar_color, verified,
              followers_count, following_count, posts_count, created_at)

posts        (id, user_id, content, likes_count, reposts_count, replies_count,
              views_count, created_at)

likes        (id, user_id, post_id, created_at) UNIQUE(user_id, post_id)
reposts      (id, user_id, post_id, created_at) UNIQUE(user_id, post_id)
replies      (id, user_id, post_id, content, created_at)
follows      (id, follower_id, following_id, created_at)
bookmarks    (id, user_id, post_id, created_at) UNIQUE(user_id, post_id)
notifications(id, user_id, from_user_id, type, post_id, read, created_at)
```

---

## Eksisterende API-endepunkter

### Auth
- POST   /api/auth/register
- POST   /api/auth/login
- GET    /api/auth/me

### Posts
- GET    /api/posts                  ← feed
- GET    /api/posts/following        ← feed fra fulgte
- POST   /api/posts                  ← nytt innlegg
- DELETE /api/posts/:id
- POST   /api/posts/:id/like
- POST   /api/posts/:id/repost
- POST   /api/posts/:id/bookmark
- GET    /api/posts/bookmarks/all
- POST   /api/posts/:id/reply
- GET    /api/posts/:id/replies

### Users
- GET    /api/users/:handle
- GET    /api/users/:handle/posts
- POST   /api/users/:handle/follow
- GET    /api/users/search/query?q=
- PUT    /api/users/me/update

---

## Design-system (VIKTIG — bruk disse konsekvent)

```javascript
// Farger
const BLACK      = "#050e05"    // bakgrunn
const PANEL      = "#0a150a"    // kort/panel
const BORDER     = "#1e2a1e"    // border
const GREEN      = "#4ade80"    // primær aksent
const GREEN_DARK = "#1a6b4a"    // knapper
const TEXT       = "#e8f5e8"    // hovedtekst
const MUTED      = "#4a7a4a"    // dempet tekst

// Fonter
// 'DM Serif Display' → overskrifter, knapper, navn
// 'Crimson Pro'      → brødtekst, innlegg, input

// Knapp-stil (primær)
background: "linear-gradient(135deg, #1a6b4a, #0e4f3a)"
borderRadius: 20, color: "#fff", fontFamily: "'DM Serif Display', serif"

// Input-stil
background: "transparent", border: "1px solid #2d4a2d"
color: "#e8f5e8", borderRadius: 10, padding: "12px 16px"
```

---

## Features som skal bygges (i prioritert rekkefølge)

### 1. BILDEOPPLASTING
**Backend:**
- Installer: `multer`, `cloudinary`
- Ny fil: `routes/upload.js` — POST /api/upload/image
- Oppdater `schema.sql`: legg til `image_url VARCHAR(500)` i posts-tabellen
- Oppdater `routes/posts.js`: inkluder image_url i alle SELECT-spørringer

**Frontend:**
- Oppdater `api.js`: legg til `uploadImage(file)` funksjon (multipart/form-data)
- Oppdater `App.jsx` compose-boks: bildeopplastingsknapp (🖼️ er allerede der men ikke koblet)
- Vis forhåndsvisning av valgt bilde før posting
- Vis bilde i PostCard hvis image_url finnes
- Ny env-variabel: CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET

### 2. DIREKTEMELDINGER
**Backend:**
- Ny tabell i `schema.sql`:
  ```sql
  CREATE TABLE messages (
    id           SERIAL PRIMARY KEY,
    sender_id    INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    receiver_id  INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content      TEXT NOT NULL CHECK (char_length(content) <= 1000),
    read         BOOLEAN DEFAULT FALSE,
    created_at   TIMESTAMPTZ DEFAULT NOW()
  );
  CREATE TABLE conversations (
    id           SERIAL PRIMARY KEY,
    user1_id     INT NOT NULL REFERENCES users(id),
    user2_id     INT NOT NULL REFERENCES users(id),
    last_message TEXT,
    updated_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(LEAST(user1_id,user2_id), GREATEST(user1_id,user2_id))
  );
  ```
- Ny fil: `routes/messages.js`
  - GET  /api/messages              ← hent alle samtaler
  - GET  /api/messages/:handle      ← hent meldinger med én bruker
  - POST /api/messages/:handle      ← send melding
  - PUT  /api/messages/:handle/read ← marker som lest

**Frontend:**
- Oppdater meldingssiden i `App.jsx` fra placeholder til fullt fungerende:
  - Venstre panel: liste over samtaler med siste melding og tidspunkt
  - Høyre panel: chat-vindu med meldingsbobler
  - Send med Enter-tasten
  - Meldingsbobler: egne høyre (grønn), andres venstre (mørk)

### 3. SANNTIDS-FEED (Socket.io)
**Backend:**
- Installer: `socket.io`
- Oppdater `server.js`: initialiser Socket.io på HTTP-serveren
- Emit events:
  - `new_post`        → når noen poster
  - `post_liked`      → når noen liker
  - `post_reposted`   → når noen reposter
  - `new_message`     → når ny melding ankommer
  - `new_notification`→ når ny varsling oppstår

**Frontend:**
- Installer: `socket.io-client`
- Oppdater `App.jsx`:
  - Koble til WebSocket ved innlogging
  - Nytt innlegg dukker opp øverst i feed automatisk
  - Likes/reposts oppdateres live uten refresh
  - Meldings-badge i navbar oppdateres live
  - Varsel-badge oppdateres live

### 4. VARSLER FRA DATABASE
**Backend:**
- Ny fil: `routes/notifications.js`
  - GET  /api/notifications          ← hent varsler (nyeste først)
  - GET  /api/notifications/unread   ← antall uleste
  - PUT  /api/notifications/read-all ← marker alle som lest
- Trigger varsel-INSERT i:
  - `posts.js` → like: notify innleggets eier
  - `posts.js` → repost: notify innleggets eier
  - `posts.js` → reply: notify innleggets eier
  - `users.js` → follow: notify den som blir fulgt

**Frontend:**
- Oppdater varsel-siden i `App.jsx` til å hente fra `/api/notifications`
- Vis ikon for type (❤️ like, 🔁 repost, 💬 svar, 👤 følger)
- Vis "ulest"-indikator (blå prikk) på uleste
- Oppdater navbar-badge dynamisk

### 5. SØK MOT DATABASE
**Backend:**
- Oppdater `routes/posts.js`: GET /api/posts?q=søkeord
  ```sql
  WHERE content ILIKE '%søkeord%'
  OR user_name ILIKE '%søkeord%'
  ```
- Oppdater `routes/users.js`: GET /api/users/search/query?q= (finnes, men utvid)

**Frontend:**
- Oppdater Utforsk-siden i `App.jsx`:
  - Debounce søk (vent 300ms etter typing)
  - Vis to faner: "Innlegg" og "Brukere"
  - Vis søkeresultater med uthevet søkeord

### 6. TRENDING FRA DATABASE
**Backend:**
- Ny fil: `routes/trending.js`
  - GET /api/trending
  ```sql
  SELECT
    regexp_matches(content, '#[A-Za-z0-9_]+', 'g') AS tag,
    COUNT(*) AS count
  FROM posts
  WHERE created_at > NOW() - INTERVAL '24 hours'
  GROUP BY tag
  ORDER BY count DESC
  LIMIT 10
  ```

**Frontend:**
- Oppdater høyre sidebar: hent trending fra API hvert 5. minutt
- Klikk på hashtag → søk i Utforsk

### 7. RATE LIMITING
**Backend:**
- Installer: `express-rate-limit`
- Oppdater `server.js`:
  ```javascript
  // Max 5 posts per minutt per bruker
  // Max 100 likes per time per bruker
  // Max 10 innloggingsforsøk per 15 min per IP
  ```

### 8. INFINITE SCROLL
**Frontend:**
- Oppdater feed i `App.jsx`:
  - State: `page = 1`, `hasMore = true`
  - IntersectionObserver på siste innlegg
  - Last 20 innlegg om gangen
  - Vis spinner mens neste side lastes

### 9. SITER INNLEGG (Quote Tweet)
**Backend:**
- Legg til `quoted_post_id INT REFERENCES posts(id)` i posts-tabellen
- Oppdater POST /api/posts til å støtte quoted_post_id
- Inkluder sitert innlegg i SELECT-spørringer

**Frontend:**
- Oppdater PostCard: legg til "Siter"-knapp ved siden av repost
- Vis sitert innlegg som en innrammet boks inni det nye innlegget
- Oppdater compose-boksen: vis sitert innlegg under tekstfeltet

### 10. REDIGER PROFIL MED BILDE
**Backend:**
- Oppdater `users` tabell: legg til `profile_image_url VARCHAR(500)`
- Oppdater PUT /api/users/me/update til å støtte profilbilde

**Frontend:**
- Oppdater profil-siden i `App.jsx`:
  - Klikk på avatar → åpne filopplaster
  - Last opp via /api/upload/image
  - Oppdater profil med ny URL
  - Vis ekte profilbilde i stedet for initialer

---

## Viktige regler under utvikling

1. **Behold design-systemet** — alltid bruk fargene og fontene definert over
2. **Behold én App.jsx** — ikke splitt i mange filer (med mindre det blir over 2000 linjer)
3. **Alltid parameteriserte SQL-spørringer** — aldri string concatenation i SQL
4. **Alltid auth-middleware** på beskyttede ruter
5. **Oppdater .env.example** når nye miljøvariabler legges til
6. **Oppdater schema.sql** når nye tabeller eller kolonner legges til
7. **Oppdater api.js** når nye endepunkter legges til i backend

---

## Miljøvariabler som trengs

```bash
# Eksisterende
DATABASE_URL=
JWT_SECRET=
FRONTEND_URL=
NODE_ENV=

# Nye (legg til i .env.example og Railway)
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
```

---

## Kommandoer

```bash
# Start backend lokalt
cd backend && npm install && node server.js

# Start frontend lokalt
cd frontend && npm install && npm start

# Kjør nye SQL-migrasjoner
psql -U postgres -d helptruth -f backend/db/schema.sql
```

---

## Testbrukere (passord: password123)
- heljar@startfunder.no  → handle: heljar
- kontakt@startfunder.no → handle: startfunder
- post@cryptonorge.no    → handle: cryptonorge
