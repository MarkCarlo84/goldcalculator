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
