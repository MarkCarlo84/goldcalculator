# Gold Calculator (Philippine Peso)

Vue.js gold calculator using **Philippine live gold price** in PHP (₱), with a **design fee** and the standard gold-value formula.

## Formula

**Gold value (₱):**

```
Gold Value = Weight (grams) × (Karat ÷ 24) × 24K price per gram (₱)
```

- **Purity factor** = Karat ÷ 24 (e.g. 18K = 18/24 = 0.75)
- **Total price** = Gold value + Design fee

## Features

- **24K gold price per gram** – Default matches [GoldPriceZ Philippines (gold per gram)](https://goldpricez.com/ph/gram). Edit the field after copying the current rate from that page for live accuracy.
- **Weight** – In grams.
- **Purity** – 24K, 22K, 21K, 18K, 14K, 10K.
- **Design fee** – Either a **fixed amount (₱)** or a **percentage** of the gold value.

## Run locally

```bash
npm install
npm run dev
```

Then open the URL shown in the terminal (usually `http://localhost:5173`).

## Build

```bash
npm run build
```

Output is in `dist/`.

## Deploy on Vercel

### Option A: Deploy with Vercel CLI

1. Install the Vercel CLI (one time):  
   `npm i -g vercel`

2. From the project folder, run:  
   `vercel`

3. Follow the prompts (log in if needed, link to a project or create new one).  
   Vercel will build and deploy; you’ll get a live URL.

4. To deploy to production:  
   `vercel --prod`

### Option B: Deploy with Git (GitHub / GitLab / Bitbucket)

1. Push this project to a Git repository (GitHub, GitLab, or Bitbucket).

2. Go to [vercel.com](https://vercel.com) and sign in.

3. Click **Add New…** → **Project**, then import your repository.

4. Vercel will detect the Vite app. Keep the defaults:
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`

5. Click **Deploy**. Your app will be built and published; each push to the main branch can auto-deploy if you enable that.
