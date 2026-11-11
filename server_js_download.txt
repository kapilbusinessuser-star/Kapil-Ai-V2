import express from "express";
import helmet from "helmet";
import rateLimit from "express-rate-limit";
import dotenv from "dotenv";
import path from "path";
import { fileURLToPath } from "url";

dotenv.config();
const app = express();
const __dirname = path.dirname(fileURLToPath(import.meta.url));
const PORT = Number(process.env.PORT || 3000);
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;
const OPENAI_MODEL = process.env.OPENAI_MODEL || "gpt-4o-mini";

app.use(helmet({ contentSecurityPolicy: false }));
app.use(express.json({ limit: "12mb" }));
app.use("/api/", rateLimit({ windowMs: 60_000, limit: 30, standardHeaders: "draft-7", legacyHeaders: false }));

// The HTML file you already uploaded to the repository root is served here.
app.get("/", (_req, res) => res.sendFile(path.join(__dirname, "kapil_ai_secure-1.html")));
app.get("/api/health", (_req, res) => res.json({ ok: true, service: "KAPIL AI backend", configured: Boolean(OPENAI_API_KEY) }));

app.post("/api/chat", async (req, res) => {
  if (!OPENAI_API_KEY) return res.status(503).json({ error: "Server API key missing. Add OPENAI_API_KEY in hosting environment." });
  const { messages, temperature = 0.7, max_tokens = 2048, stream = true } = req.body || {};
  if (!Array.isArray(messages) || !messages.length) return res.status(400).json({ error: "A non-empty messages array is required." });
  if (messages.length > 40) return res.status(400).json({ error: "Too many messages." });
  try {
    const upstream = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: { Authorization: `Bearer ${OPENAI_API_KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify({
        model: OPENAI_MODEL, messages,
        temperature: Math.max(0, Math.min(2, Number(temperature) || 0.7)),
        max_tokens: Math.max(64, Math.min(8192, Number(max_tokens) || 2048)),
        stream: Boolean(stream)
      })
    });
    if (!upstream.ok) return res.status(upstream.status).json({ error: "OpenAI request failed", detail: (await upstream.text()).slice(0,1200) });
    if (!stream) return res.json(await upstream.json());
    res.setHeader("Content-Type", "text/event-stream; charset=utf-8");
    res.setHeader("Cache-Control", "no-cache, no-transform");
    res.setHeader("Connection", "keep-alive");
    const reader = upstream.body.getReader();
    req.on("close", () => reader.cancel().catch(() => {}));
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      res.write(Buffer.from(value));
    }
    res.end();
  } catch (e) {
    if (!res.headersSent) res.status(500).json({ error: "Backend error", detail: e.message });
    else res.end();
  }
});
app.listen(PORT, "0.0.0.0", () => console.log(`KAPIL AI running on port ${PORT}`));
