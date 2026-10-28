# Lung Prevention API

Backend for an Augmented Reality serious game about **lung cancer prevention**, developed as part of a Master's thesis in Computer Engineering.

The game is a Unity + Vuforia Android app with three levels, each unlocked by scanning a printed image target:

| Level | Game | Theme |
|-------|------|-------|
| 1 | Match-3 molecule game | Healthy breathing (O₂ / CO₂) |
| 2 | Slicing game | Harmful vs. healthy habits |
| 3 | Quiz + treatments | Tumours and treatment (surgery, radiation, medication) |

This API sits between the app and the external services. It handles:

- **Image targets:** creates, lists and deletes the targets in the Vuforia Cloud Database, so the app's admin panel can replace them without opening the Vuforia developer portal.
- **Leaderboard:** stores and returns player scores.
- **Session metrics:** stores per-session gameplay data used in the thesis evaluation, and exports it to Excel for statistical analysis.

## Architecture

```
┌──────────────────┐        ┌───────────────────────┐        ┌──────────────────────────┐
│  Unity app (AR)  │──HTTP─▶│  Lung Prevention API  │──VWS──▶│ Vuforia Cloud Database   │
│  game + admin    │        │  Node.js / Express    │        └──────────────────────────┘
└──────────────────┘        │                       │──────▶ Firebase Storage  (target images)
         ▲                  │                       │──────▶ Cloud Firestore   (leaderboard, metricas)
         │                  └───────────────────────┘
         └──── image recognition: Vuforia Cloud Recognition (directly from the app)
```

- **Vuforia Web Services (VWS):** requests are signed with HMAC-SHA1 using the server access/secret keys. The keys never reach the app.
- **Firebase Storage:** keeps a copy of every target image so the admin panel can show a gallery. Vuforia does not return the original images.
- **Cloud Firestore:** stores the `leaderboard` and `metricas` collections in the database named `prevention-game`.

## Tech stack

- Node.js 18+ and Express 5
- `firebase-admin` (Storage + Firestore)
- `axios` (Vuforia Web Services)
- `exceljs` (metrics export)
- Deployed on [Render](https://render.com)

## Getting started

### 1. Install

```bash
npm install
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
VUFORIA_SERVER_ACCESS_KEY=your_vuforia_server_access_key
VUFORIA_SERVER_SECRET_KEY=your_vuforia_server_secret_key
PORT=3000
```

Use the **Server Access Keys** of the Cloud Database in the Vuforia developer portal, not the Client Access Keys.

### 3. Add the Firebase service account

Download a service account key from *Firebase Console → Project settings → Service accounts* and save it as `firebase-key.json` in the project root.

The server looks for the key in this order:

1. `/etc/secrets/firebase-key.json` (Render Secret File)
2. `./firebase-key.json` (local)

> Both `.env` and `firebase-key.json` are listed in `.gitignore`. Never commit them.

### 4. Run

```bash
npm run dev     # development, with auto-reload (nodemon)
npm start       # production
```

The server listens on `PORT` (default `3000`).

## API reference

All request and response bodies are JSON. Errors return HTTP `500` (or `400` for validation errors) with `{ "success": false, "error": ... }`.

### Image targets

#### `GET /gallery`

Lists the target images stored in Firebase Storage.

```json
{
  "marcadores": [
    {
      "id": "f7b4d4b991334342be8377a9f7007078",
      "nome": "target_nivel_1",
      "urlImagem": "https://firebasestorage.googleapis.com/v0/b/<bucket>/o/<file>?alt=media"
    }
  ]
}
```

#### `GET /targets/:id`

Returns the target's record from Vuforia, including its processing status and rating. This is a direct proxy of the VWS response.

#### `POST /targets`

Creates a new target in Vuforia and saves a copy of the image in Firebase Storage. **If a target with the same `name` already exists, it is deleted first**, from both Vuforia and Storage, so each level always has a single active target.

```json
{
  "name": "target_nivel_1",
  "width": 1,
  "imageBase64": "<PNG encoded in base64>",
  "metadata": { }
}
```

Returns the VWS response, including the new `target_id`.

> The app decides which level to open from the target name: it must contain `nivel_1`, `nivel_2` or `nivel_3`.

#### `DELETE /targets/:id/:name`

Deletes the target from Vuforia and removes its image from Firebase Storage.

### Leaderboard

#### `POST /leaderboard`

```json
{ "username": "player01", "score": 1250 }
```

Returns `{ "success": true, "id": "<firestore doc id>" }`.

#### `GET /leaderboard`

Returns every score, highest first.

```json
{
  "success": true,
  "leaderboard": [
    { "id": "...", "username": "player01", "score": 1250, "timestamp": { } }
  ]
}
```

### Metrics

#### `POST /metricas`

The app sends this once per play-through, after level 3 is completed. `sessao_id` and `username` are required. The document is stored in `metricas/{sessao_id}` and a server `timestamp` is added.

```json
{
  "sessao_id": "player01_20260630-143012_a1b2c3d4",
  "username": "player01",
  "data": "2026-06-30",
  "nivel1": {
    "pontuacao_final": 0,
    "moleculas_o2": 0,
    "moleculas_co2": 0,
    "moleculas_corretas": 0,
    "moleculas_erradas": 0
  },
  "nivel2": {
    "pontuacao_final": 0,
    "cortes_corretos": 0,
    "cortes_errados": 0,
    "miss_count": 0,
    "detalhes_cortes_errados": [ { "item_name": "agua", "quantidade": 1 } ]
  },
  "nivel3": {
    "pontuacao_final": 0,
    "respostas_certas": 0,
    "respostas_erradas": 0,
    "total_perguntas": 0,
    "bisturi_concluido": true,
    "radiacao_concluida": true,
    "comprimido_concluido": true,
    "tratamento_completo": true,
    "detalhes_respostas": [
      { "pergunta": "...", "resposta_dada": "...", "estava_correta": true }
    ]
  }
}
```

## Exporting metrics to Excel

```bash
node export-metricas.js
```

The script reads the whole `metricas` collection and writes `metricas_<timestamp>.xlsx` to the project root. It produces one row per session, with nested fields flattened into columns (for example `nivel1.pontuacao_final`). The two detail arrays are turned into their own columns:

| Field | Becomes |
|-------|---------|
| `detalhes_cortes_errados` | One column per item: `CORTE ERRADO > agua`, `CORTE ERRADO > maca`, ... (value = count) |
| `detalhes_respostas` | One column per question: `QUIZ > PERGUNTA 1`, `QUIZ > PERGUNTA 2`, ... (value = `ACERTOU` / `ERROU`) |

The spreadsheet can be loaded directly into jamovi or SPSS.

## Deployment (Render)

1. Create a **Web Service** from this repository. Use `npm install` as the build command and `npm start` as the start command.
2. Add `VUFORIA_SERVER_ACCESS_KEY` and `VUFORIA_SERVER_SECRET_KEY` as environment variables. Render sets `PORT` automatically.
3. Upload `firebase-key.json` as a **Secret File**. Render mounts it at `/etc/secrets/firebase-key.json`.

On the free plan the service sleeps when idle, so the first request after a pause can take up to a minute.

## Project structure

```
├── server.js            # Express app: targets, gallery, leaderboard, metrics
├── export-metricas.js   # Firestore → Excel export
├── uploads/             # Sample target images (levels 1–3)
├── .env                 # Vuforia keys (not committed)
└── firebase-key.json    # Firebase service account (not committed)
```

## Known limitations

- **No authentication.** All endpoints are public, so anyone with the URL can create or delete targets and write scores or metrics. This is acceptable for a controlled study, but an API key or token is required before any public release.
- `GET /leaderboard` returns every entry, with no pagination.
- Request bodies are limited to 50 MB to allow base64 images.

## Author

**Miguel Costa**, Master's student in Computer Engineering
