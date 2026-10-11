# osurankreg

# osu! Rank → Hours Played Estimator (Indonesia Leaderboard Data Sample)

A Google Colab notebook that estimates **how many hours a player has likely spent playing osu!** (standard mode) from their **global rank**. It scrapes the osu! API v2 performance leaderboard, fits several regression models of hours vs. rank, and compares them with cross-validation.

The data is focused on the **Indonesian (ID) leaderboard**, optionally supplemented with the global top 50 pages for better accuracy at very high ranks.

---

## How it works

1. **Authenticate** with the osu! API v2 using the `client_credentials` grant.
2. **Scrape** `/rankings/osu/performance` page by page (50 players/page), collecting `user_id`, `username`, `global_rank`, `pp`, and `play_time` (converted to hours). Rate limits (HTTP 429) are handled with a retry, and requests are throttled to ~1.1 s each.
3. **Fit models** of hours as a function of global rank.
4. **Predict** hours for any rank you type in.
5. **Evaluate** all models with 5-fold cross-validation (the final cell).

### Datasets

| Section | Pages | Rough rank coverage | Purpose |
|---|---|---|---|
| Global ("all digit") dataset | ID pages 1–200 (+ global pages 1–49, optional) | All ranks | One model across everything |
| 5 digits | ID pages 5–45 | ~5-digit ranks | Model tuned for that range |
| 6 digits | ID pages 45–200 | ~6-digit ranks | Model tuned for that range |

> The osu! API caps rankings at 200 pages, so each country query returns at most 10,000 players.

---

## Models compared

- **Linear** – `hours = a + b·rank`
- **Log** – `log(hours) = a + b·rank`
- **Log-log (power law)** – `log(hours) = a + b·log(rank)`
- **Saturating** – `log(hours) = logH − log(1 + exp(k·(log(rank) − log(r0))))`, a sigmoid-style curve that levels off at a maximum `H`

The **saturating** and **log-log spline** models perform best and are nearly tied. All models slightly underestimate (negative bias). Expect roughly **±30% typical error** — this is a rough estimate, not an exact figure.

---

## Setup

### Requirements

- Python 3 with: `requests`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `matplotlib`
- A [Google Colab](https://colab.research.google.com/) environment (the notebook uses `google.colab.userdata` for secrets)

### osu! API credentials

1. Go to your osu! account settings → **OAuth** → **New OAuth Application**.
2. Copy the **Client ID** and **Client Secret**.
3. In Colab, open the **Secrets** panel (key icon) and add:
   - `OSU_CLIENT_ID`
   - `OSU_CLIENT_SECRET`
4. Enable notebook access for both secrets.

> Never hard-code or commit your client secret.

---

## Usage

Run the cells in order:

1. **Authentication cell** – gets an access token.
2. **Helper cell** – defines `progress()` and `process(START, END)`.
3. **Pick a dataset section** (global, 5-digit, or 6-digit) and run the scrape + fit cells. Scraping 200 pages takes ~4 minutes due to rate limiting.
4. **Prediction cell** – enter a rank (commas allowed, e.g. `109,500`) and get estimates from every model:

   ```
   Rank 109,500 → lin: 644 | log: 485 | log-log: 377 | saturating: 432 h
   ```

5. **Evaluation cell** – runs 5-fold cross-validation and prints metrics.

### Customizing

- **Different country:** change `"country": "ID"` in `process()`.
- **Different page range:** change `START, END` before calling `process()`.
- **Different game mode:** replace `osu` in the rankings URL with `taiko`, `fruits`, or `mania`.

---

## Notes and limitations

- **Pages vs. ranks:** ID page *n* is the *n*-th page of Indonesian players, so its `global_rank` values are not contiguous. Ranks outside the sampled range are extrapolated.
- **Play time is self-reported by the game** and includes time spent on any play, not strictly "efficient" practice. Players of the same rank can differ greatly (e.g. returning players, alt accounts, prior game experience).
- The prediction cell uses whichever dataset/fit was run **most recently** — re-run the relevant section before predicting.
- The evaluation cell reuses the current `df`, so it evaluates whichever dataset was last loaded.
- The optional global top-50-pages cell prints `/10` in its progress message but actually loops over 49 pages (cosmetic only).
- `time` and `pandas` are imported in a later cell than the helper that uses them; run the dataset cell once (or move the imports to the top) if you hit a `NameError`.

---

## Credits

Data from the [osu! API v2](https://osu.ppy.sh/docs/index.html). Not affiliated with or endorsed by ppy Pty Ltd.
