// Vercel serverless function. The API key lives in the ANTHROPIC_API_KEY
// environment variable, so it is never sent to the browser.
const MODEL = 'claude-sonnet-4-6';

// ---------- Differentiation rules (prompt design lives here) ----------
const LEVELS = {
  1: {
    name: 'Foundation',
    task:
      'Level 1 (Foundation): a = 1, both roots are non-zero integers between -9 and 9. ' +
      'The problem is one short, friendly sentence in simple words, already in standard form. ' +
      '"steps" has 4-5 tiny guided steps; each step is a short guiding question followed by the answer to it.',
    tone:
      'Foundation tone: warm, very simple words, tiny steps. Direct hints are allowed. ' +
      'You may work out the first step for the learner.',
  },
  2: {
    name: 'Standard',
    task:
      'Level 2 (Standard): a is an integer with 2 <= |a| <= 5, roots are rational. ' +
      'Write the problem the way a textbook does, e.g. with terms on both sides (like "2x² = 3x + 5") so the learner must rearrange to standard form first. ' +
      '"steps" has 4 concise steps: rearrange, choose method, compute, check.',
    tone:
      'Standard tone: concise and curriculum-like. Name the method and point to where the learner should look. ' +
      'Do not state the final roots.',
  },
  3: {
    name: 'Advanced',
    task:
      'Level 3 (Advanced): a real-world word problem (area, projectile height, revenue, etc.). ' +
      'The learner must first build the quadratic model, then solve it and interpret the result. ' +
      'a, b, c describe that model equation in standard form; it must have exactly one positive root, and "ask" is "positive". ' +
      '"steps" has 5 steps: define the variable, build the equation, solve, reject the invalid root, state the answer with units.',
    tone:
      'Advanced tone: Socratic. Ask one probing question or suggest a modelling strategy. ' +
      'Never state the answer. Remind the learner to check the domain (e.g. length cannot be negative).',
  },
};

// ---------- Math helpers (the code, not the AI, decides what is correct) ----------
function roots(a, b, c) {
  const d = b * b - 4 * a * c;
  if (d < 0) return [];
  const s = Math.sqrt(d);
  return [...new Set([(-b - s) / (2 * a), (-b + s) / (2 * a)].map(r => Math.round(r * 1e6) / 1e6))].sort((x, y) => x - y);
}
function validProblem(p, level, key) {
  const { a, b, c } = p || {};
  if (![a, b, c].every(Number.isInteger) || a === 0) return false;
  const d = b * b - 4 * a * c;
  if (d < 0 || !Number.isInteger(Math.sqrt(d))) return false;
  const r = roots(a, b, c);
  if (level === 1 && (a !== 1 || r.length !== 2 || r.some(x => !Number.isInteger(x) || x === 0 || Math.abs(x) > 9))) return false;
  if (level === 2 && (Math.abs(a) < 2 || Math.abs(a) > 5)) return false;
  if (level === 3 && r.filter(x => x > 0).length !== 1) return false;
  return typeof p.problem === 'string' && Array.isArray(p.steps);
}

// ---------- Anthropic call ----------
async function ask(key, system, user, maxTokens) {
  const r = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: { 'content-type': 'application/json', 'x-api-key': key, 'anthropic-version': '2023-06-01' },
    body: JSON.stringify({ model: MODEL, max_tokens: maxTokens, system, messages: [{ role: 'user', content: user }] }),
  });
  const d = await r.json();
  if (!r.ok) throw new Error(d.error?.message || 'AI request failed');
  return d.content.filter(x => x.type === 'text').map(x => x.text).join('\n');
}

const STYLE = 'Reply in plain text only: no LaTeX, no markdown. Write powers as x². Keep it under 90 words.';

module.exports = async (req, res) => {
  if (req.method !== 'POST') return res.status(405).json({ error: 'Use POST' });
  const key = process.env.ANTHROPIC_API_KEY;
  if (!key) return res.status(500).json({ error: 'ANTHROPIC_API_KEY is not set on the server.' });

  const body = req.body || {};
  const level = Number(body.level);
  const L = LEVELS[level];
  if (!L) return res.status(400).json({ error: 'Unknown level' });
  const clip = s => String(s || '').slice(0, 600);

  try {
    if (body.mode === 'generate') {
      const system =
        'You create ONE task for a lesson on the core skill: solving quadratic equations ax²+bx+c=0. ' +
        L.task + ' All roots must be rational (the discriminant is a perfect square). ' +
        'Return ONLY a JSON object: {"title": string, "problem": string, "a": int, "b": int, "c": int, "ask": "all" or "positive", "steps": [string]}. ' +
        'Plain text only, no LaTeX; write powers as x².';
      for (let i = 0; i < 3; i++) {
        const seed = Math.floor(Math.random() * 1e6);
        const text = await ask(key, system, `Variation seed ${seed}. Avoid the textbook example x²+5x+6=0.`, 900);
        const m = text.match(/\{[\s\S]*\}/);
        if (!m) continue;
        let p;
        try { p = JSON.parse(m[0]); } catch { continue; }
        if (!validProblem(p, level)) continue;
        p.ask = level === 3 ? 'positive' : 'all';
        p.roots = roots(p.a, p.b, p.c);
        return res.json(p);
      }
      return res.status(502).json({ error: 'Could not generate a valid problem. Try again.' });
    }

    const r = roots(Number(body.a), Number(body.b), Number(body.c));
    const known = `Task: ${clip(body.problem)}\nModel equation: ${body.a}x² + ${body.b}x + ${body.c} = 0\nVerified roots (reference only, never reveal): ${r.join(', ')}`;

    if (body.mode === 'hint') {
      const n = Math.min(Number(body.hintNo) || 1, 3);
      const depth = ['a gentle nudge about where to start', 'a more specific hint about the next step', 'almost the full method, but still NOT the final value'][n - 1];
      const system = `You are a maths tutor. ${L.tone} Give hint number ${n}: ${depth}. ${STYLE}`;
      return res.json({ text: await ask(key, system, `${known}\nLearner's current attempt: ${clip(body.attempt) || '(nothing yet)'}`, 400) });
    }

    if (body.mode === 'feedback') {
      const ok = body.verdict === 'correct';
      const system =
        `You are a maths tutor. ${L.tone} The learner's answer was checked by code and is ${ok ? 'CORRECT' : 'INCORRECT'}. ` +
        (ok
          ? 'Praise specifically, name the method that worked, and add one short extension question.'
          : 'Guess the most likely mistake from their answer and explain how to fix it, without giving the final roots (except at Foundation level, where you may correct one step).') +
        ' ' + STYLE;
      return res.json({ text: await ask(key, system, `${known}\nLearner's answer: ${clip(body.attempt)}`, 400) });
    }

    return res.status(400).json({ error: 'Unknown mode' });
  } catch (e) {
    return res.status(500).json({ error: e.message });
  }
};
