# Deploying PollFlow

This gets you from local `docker compose up` to a live, publicly reachable link, all on free tiers.

Pieces:
1. **MongoDB Atlas** — managed MongoDB
2. **Upstash Redis** (or Redis Cloud) — managed Redis
3. **Render** (or Fly.io / Railway) — hosts the Go API as a container
4. **Vercel** (or Netlify) — hosts the built React frontend

---

## 1. MongoDB Atlas

1. Create a free account at https://www.mongodb.com/cloud/atlas and create a free **M0** cluster.
2. Database Access → add a database user with a strong password.
3. Network Access → add `0.0.0.0/0` (allow from anywhere) so your deployed backend can reach it.
4. Get the connection string from **Connect → Drivers**, it looks like:
   `mongodb+srv://<user>:<password>@<cluster>.mongodb.net/?retryWrites=true&w=majority`
5. This is your `MONGO_URI`. Keep `MONGO_DATABASE=pollflow` (or whatever name you prefer — Atlas creates it on first write).

## 2. Redis (Upstash)

1. Create a free account at https://upstash.com and create a Redis database (choose a region close to your backend's region).
2. Copy the connection details. Upstash gives you a `redis://` or `rediss://` URL with a password baked in, e.g.
   `rediss://default:<password>@<host>:6379`
3. Paste that whole URL as `REDIS_ADDR`. The backend's `db/redis.go` accepts either a bare `host:port` (for local, unauthenticated Docker Redis) or a full `redis://`/`rediss://` URL with embedded credentials (for Upstash, Redis Cloud, etc.) — it detects which one you gave it automatically.

## 3. Backend on Render

1. Push this repo to a **public GitHub repo** (required for submission anyway).
2. On https://render.com → New → Web Service → connect your repo.
3. Settings:
   - **Root directory:** `backend`
   - **Runtime:** Docker (Render will pick up `backend/Dockerfile` automatically)
   - **Instance type:** Free is fine for a demo
4. Environment variables (Render dashboard → Environment):
   - `PORT` → `8080` (Render sets its own `$PORT`; if Render requires binding to its injected port, set `PORT` to match or update `main.go` to read `os.Getenv("PORT")`, which it already does)
   - `MONGO_URI` → your Atlas connection string
   - `MONGO_DATABASE` → `pollflow`
   - `REDIS_ADDR` → your Redis host:port (see note above)
   - `JWT_SECRET` → a long random string (e.g. `openssl rand -hex 32`)
   - `FRONTEND_URL` → your Vercel URL from step 4 (you can add this after step 4, then redeploy)
5. Deploy. Once live, note the backend's URL, e.g. `https://pollflow-backend.onrender.com`.
6. Check it: `https://pollflow-backend.onrender.com/api/health` should return `{"status":"ok"}`.

**Note on SSE + free tiers:** free instances on some platforms idle/sleep after inactivity, which will drop open SSE connections. For a submission demo this is fine — the frontend's `EventSource` reconnects automatically on the next request. If you need it always-warm, use a paid always-on instance or a platform with no idle timeout (Fly.io's free allowance behaves similarly; keep this in mind for the video demo).

## 4. Frontend on Vercel

1. On https://vercel.com → New Project → import the same repo.
2. Settings:
   - **Root directory:** `frontend`
   - **Framework preset:** Vite (auto-detected)
   - **Build command:** `npm run build` — **Output directory:** `dist`
3. Environment variable:
   - `VITE_API_URL` → your Render backend URL from step 3 (no trailing slash)
4. Deploy. You'll get a URL like `https://pollflow.vercel.app`.
5. Go back to Render and set `FRONTEND_URL` to this exact Vercel URL, then redeploy the backend so CORS allows it.

## 5. Verify end-to-end

1. Open the Vercel URL, sign up, and create a poll.
2. Open the poll's link in a second private/incognito window and vote.
3. Confirm the first window's results update without a refresh.
4. Confirm a poll owner can close their poll, and that a second vote attempt from the same browser is rejected.

## Alternative: containerized frontend

If you'd rather not use Vercel/Netlify, `frontend/Dockerfile` builds the app with Vite and serves it via nginx (`frontend/nginx.conf` handles client-side routing). Deploy it the same way as the backend, passing `VITE_API_URL` as a build arg:

```bash
docker build -t pollflow-frontend --build-arg VITE_API_URL=https://your-backend-url ./frontend
```
