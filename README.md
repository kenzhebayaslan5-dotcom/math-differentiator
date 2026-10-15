# Quadratic Differentiator

Web app: one core task (solving quadratic equations) is adapted by an AI model into three levels — Foundation, Standard, Advanced — with level-specific hints and feedback.

## Structure
- `index.html` — interface (level selector, answer check, hints, worked steps)
- `api/ai.js` — Vercel serverless function: system prompts, API call, math verification

## How it works
1. The browser asks `/api/ai` for a problem at the chosen level.
2. The server prompts the AI with level-specific rules and returns JSON (a, b, c, steps).
3. The server rejects any problem whose coefficients break the level rules (discriminant, integer roots, etc.) and asks again.
4. Answers are checked by JavaScript, not by the AI. The AI only explains.

## Run / deploy
1. Push this folder to a public GitHub repo.
2. On vercel.com: Add New Project → import the repo.
3. Settings → Environment Variables: add `ANTHROPIC_API_KEY`.
4. Deploy. The live URL is your submission link.

Local test: `npm i -g vercel`, then `vercel dev` (set the same environment variable).
