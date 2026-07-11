// ═══════════════════════════════════════════════════════════
// HACKVENTURE AI LESSON WORKER
// Generates lessons with Workers AI, caches them FOREVER in KV.
// First student to reach a lesson triggers generation;
// everyone after gets the exact same cached lesson instantly.
// ═══════════════════════════════════════════════════════════
//
// Required bindings (Worker → Settings → Bindings):
//   1. Workers AI  → variable name: AI
//   2. KV namespace → variable name: LESSONS
//      → namespace: hackventure-lessons (id: 71f1b152cb2f48148fad42222c7ee1f4)
//
// ═══════════════════════════════════════════════════════════

const ALLOWED_ORIGINS = [
  'https://hackventure.dev',
  'https://www.hackventure.dev',
  'https://codequest-iaro.netlify.app',
  'http://localhost:8888',
];

const MODELS = [
  '@cf/meta/llama-3.3-70b-instruct-fp8-fast',
  '@cf/meta/llama-3.1-8b-instruct',
  '@cf/meta/llama-3.1-8b-instruct-fast',
  '@cf/meta/llama-4-scout-17b-16e-instruct',
];
const DAILY_LIMIT_PER_IP = 60; // generations per visitor per day (cached hits are free)

export default {
  async fetch(request, env) {
    const origin = request.headers.get('Origin') || '';
    const cors = {
      'Access-Control-Allow-Origin': ALLOWED_ORIGINS.includes(origin) ? origin : ALLOWED_ORIGINS[0],
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: cors });
    }
    if (request.method !== 'POST') {
      return json({ error: 'POST only' }, 405, cors);
    }
    if (origin && !ALLOWED_ORIGINS.includes(origin)) {
      return json({ error: 'Origin not allowed' }, 403, cors);
    }

    let body;
    try { body = await request.json(); }
    catch { return json({ error: 'Bad JSON' }, 400, cors); }

    const { kind, lang, langName } = body;
    if (!lang || !langName || !/^[a-z]+$/.test(lang)) {
      return json({ error: 'Missing lang' }, 400, cors);
    }

    // ── Build the permanent cache key ──
    let key;
    if (kind === 'extra') {
      const n = Math.max(1, Math.min(30, parseInt(body.n) || 1));
      key = `v2_extra_${lang}_${n}`;
    } else {
      const si = int(body.si, 0, 9), ui = int(body.ui, 0, 4), li = int(body.li, 0, 9);
      key = `v2_lesson_${lang}_${si}_${ui}_${li}`;
    }

    // ── Cached? Serve instantly, costs nothing ──
    if (!env.LESSONS) {
      return json({ error: 'CONFIG: LESSONS binding is missing — add KV binding named LESSONS' }, 500, cors);
    }
    const cached = await env.LESSONS.get(key);
    if (cached) {
      return new Response(cached, { headers: { ...cors, 'Content-Type': 'application/json', 'X-Cache': 'HIT' } });
    }

    // ── Rate limit actual generations ──
    const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
    const day = new Date().toISOString().slice(0, 10);
    const rlKey = `rl_${ip}_${day}`;
    const used = parseInt(await env.LESSONS.get(rlKey) || '0');
    if (used >= DAILY_LIMIT_PER_IP) {
      return json({ error: 'Daily limit reached — try tomorrow!' }, 429, cors);
    }

    // ── Build the prompt ──
    const prompt = kind === 'extra'
      ? extraPrompt(langName, body.n)
      : lessonPrompt(langName, body);

    // ── Check bindings are wired up ──
    if (!env.AI) {
      return json({ error: 'CONFIG: AI binding is missing — add Workers AI binding named AI' }, 500, cors);
    }
    if (!env.LESSONS) {
      return json({ error: 'CONFIG: LESSONS binding is missing — add KV binding named LESSONS' }, 500, cors);
    }

    // ── Generate: try each model until one works ──
    let lesson = null;
    let usedModel = '';
    const errors = [];
    for (const model of MODELS) {
      try {
        const ai = await env.AI.run(model, {
          messages: [
            { role: 'system', content: 'You are a coding teacher writing lessons for a learning app. You reply with ONLY a raw JSON object. No markdown, no backticks, no explanations — just JSON starting with { and ending with }.' },
            { role: 'user', content: prompt }
          ],
          max_tokens: 1400,
        });
        const raw = ai?.response ?? ai?.result ?? ai?.choices?.[0]?.message?.content ?? ai;
        lesson = parseLesson(raw);
        if (lesson) { usedModel = model; break; }
        errors.push(`${model}: bad JSON output`);
      } catch (e) {
        errors.push(`${model}: ${e.message}`);
      }
    }

    if (!lesson) {
      return json({ error: 'Generation failed', details: errors }, 502, cors);
    }

    // ── Cache FOREVER + count the generation ──
    const out = JSON.stringify(lesson);
    await env.LESSONS.put(key, out);
    await env.LESSONS.put(rlKey, String(used + 1), { expirationTtl: 86400 });

    return new Response(out, { headers: { ...cors, 'Content-Type': 'application/json', 'X-Cache': 'MISS' } });
  }
};

// ═══════════════ helpers ═══════════════

function int(v, min, max) {
  const n = parseInt(v);
  return isNaN(n) ? min : Math.max(min, Math.min(max, n));
}

function json(obj, status, cors) {
  return new Response(JSON.stringify(obj), {
    status,
    headers: { ...cors, 'Content-Type': 'application/json' }
  });
}

function lessonPrompt(langName, b) {
  const lessonNum = int(b.li, 0, 9) + 1;
  return `Write lesson ${lessonNum} of 10 for the unit "${b.unitTitle}" (section "${b.sectionTitle}") in a ${langName} programming course.
Unit topics: ${b.topics || 'core concepts'}.
The lesson should teach ONE focused concept from those topics, appropriate for lesson ${lessonNum} of 10 (later lessons = more advanced).

Reply with ONLY this JSON structure:
{
  "icon": "one relevant emoji",
  "title": "short title, 2-4 words",
  "desc": "one line description",
  "content": "HTML string: one <p> explanation (1-3 sentences, may use <strong>) followed by one <div class=\\"code-box\\">code example with real ${langName} code</div>. Use \\n for line breaks inside the code-box. Wrap keywords in <span class=\\"kw\\">, strings in <span class=\\"str\\">, numbers in <span class=\\"num\\">, function names in <span class=\\"fn\\">, comments in <span class=\\"cm\\">.",
  "challenges": [
    { "type": "mcq", "q": "clear question about the concept", "opts": ["A", "B", "C", "D"], "ans": 0, "hint": "short hint", "ok": "✅ why correct", "bad": "❌ what the right answer is" },
    { "type": "fill", "q": "fill in the blank task", "pre": "code before blank", "suf": "code after blank", "ans": ["answer"], "hint": "short hint", "ok": "✅ explanation", "bad": "❌ explanation" }
  ]
}
Rules: exactly 2 challenges (one mcq, one fill). "ans" in mcq is the INDEX of the correct option. "ans" in fill is an array of acceptable answers. All code must be valid ${langName}. Everything must be accurate.`;
}

function extraPrompt(langName, n) {
  const themes = ['loops', 'conditionals', 'functions', 'strings', 'lists and collections', 'error handling', 'classes and objects', 'algorithms', 'math operations', 'input and output', 'debugging', 'best practices', 'data types', 'operators', 'recursion', 'sorting', 'searching', 'dictionaries and maps', 'file handling', 'clean code'];
  const theme = themes[(int(n, 1, 30) - 1) % themes.length];
  return `Write a fun EXTRA PRACTICE lesson about "${theme}" in ${langName} for a student who finished the whole course. Make it a creative challenge that reviews the concept in a fresh way.

Reply with ONLY this JSON structure:
{
  "icon": "one relevant emoji",
  "title": "short title, 2-4 words",
  "desc": "one line description",
  "content": "HTML string: one <p> explanation (1-3 sentences, may use <strong>) followed by one <div class=\\"code-box\\">code example with real ${langName} code</div>. Use \\n for line breaks inside the code-box. Wrap keywords in <span class=\\"kw\\">, strings in <span class=\\"str\\">, numbers in <span class=\\"num\\">, function names in <span class=\\"fn\\">, comments in <span class=\\"cm\\">.",
  "challenges": [
    { "type": "mcq", "q": "clear question", "opts": ["A", "B", "C", "D"], "ans": 0, "hint": "short hint", "ok": "✅ why correct", "bad": "❌ what the right answer is" },
    { "type": "fill", "q": "fill in the blank task", "pre": "code before blank", "suf": "code after blank", "ans": ["answer"], "hint": "short hint", "ok": "✅ explanation", "bad": "❌ explanation" }
  ]
}
Rules: exactly 2 challenges. "ans" in mcq is the INDEX of the correct option. All code must be valid ${langName}.`;
}

function parseLesson(raw) {
  if (!raw) return null;
  let lesson;
  if (typeof raw === 'object') {
    // Newer models return the parsed JSON object directly
    lesson = raw;
  } else if (typeof raw === 'string') {
    const start = raw.indexOf('{');
    const end = raw.lastIndexOf('}');
    if (start === -1 || end <= start) return null;
    try { lesson = JSON.parse(raw.slice(start, end + 1)); }
    catch { return null; }
  } else {
    return null;
  }

  // Validate structure
  if (typeof lesson.title !== 'string' || typeof lesson.content !== 'string') return null;
  if (!Array.isArray(lesson.challenges)) return null;
  lesson.challenges = lesson.challenges.filter(ch =>
    (ch.type === 'mcq' && Array.isArray(ch.opts) && ch.opts.length >= 2 && typeof ch.ans === 'number') ||
    (ch.type === 'fill' && typeof ch.pre === 'string' && Array.isArray(ch.ans) && ch.ans.length > 0)
  );
  if (lesson.challenges.length === 0) return null;
  lesson.icon = lesson.icon || '📚';
  lesson.desc = strip(lesson.desc || '');
  lesson.title = strip(lesson.title);
  // Safety defaults
  lesson.challenges.forEach(ch => {
    ch.hint = strip(ch.hint || 'Think about the lesson above...');
    ch.ok = strip(ch.ok || '✅ Correct!');
    ch.bad = strip(ch.bad || '❌ Not quite — check the lesson again.');
    ch.q = strip(ch.q || 'Complete the challenge:');
    if (ch.type === 'mcq') {
      ch.opts = ch.opts.map(strip);
    }
    if (ch.type === 'fill') {
      ch.pre = strip(ch.pre);
      ch.suf = strip(ch.suf || '');
      ch.ans = ch.ans.map(strip);
    }
  });
  return lesson;
}

// Remove HTML tags — quiz fields must be plain text (HTML lives only in content)
function strip(s) {
  return typeof s === 'string' ? s.replace(/<[^>]*>/g, '') : s;
}
