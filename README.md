# Manish AI — Single-file project bundle

This document contains a complete small web app (React frontend + Node/Express backend) you can run locally or deploy to Vercel/Render/GitHub. It implements a ChatGPT-like Q&A interface called **Manish AI** and uses the OpenAI API for answers (you provide your own API key).

---

## File: README.md

```
Manish AI
=========

A minimal ChatGPT-like web app (React + Node). Follow steps below to run locally or deploy.

Requirements
- Node.js 18+
- An OpenAI API key (store in OPENAI_API_KEY env var)

Run locally
1. Create a project folder and copy files from this repo structure.
2. npm install
3. export OPENAI_API_KEY="sk-..." (Windows: set OPENAI_API_KEY=...)
4. npm run dev
5. Open http://localhost:3000

Deploy to Vercel (recommended)
1. Create a new GitHub repo and push this code.
2. Import repo into Vercel.
3. In Vercel dashboard, set Environment Variable OPENAI_API_KEY to your key.
4. Deploy. Vercel will provide a public URL (e.g. https://manish-ai-xxxxx.vercel.app).

Notes
- Do NOT commit your OpenAI API key to the repo.
- This sample uses a lightweight server endpoint /api/chat to proxy requests to OpenAI.
```

---

## File: package.json

```
{
  "name": "manish-ai",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "node server.js",
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2",
    "openai": "^4.0.0"
  }
}
```

---

## File: server.js

```js
// Minimal Express server that serves the React app (production) and provides /api/chat
const express = require('express');
const path = require('path');
const cors = require('cors');
const bodyParser = require('body-parser');

const { Configuration, OpenAIApi } = require('openai');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const OPENAI_KEY = process.env.OPENAI_API_KEY;
if (!OPENAI_KEY) {
  console.warn('Warning: OPENAI_API_KEY not set. /api/chat will fail without it.');
}

const configuration = new Configuration({ apiKey: OPENAI_KEY });
const openai = new OpenAIApi(configuration);

app.post('/api/chat', async (req, res) => {
  try {
    const { messages } = req.body; // array of {role, content}
    if (!messages) return res.status(400).json({ error: 'Missing messages' });

    // Basic guard: limit message length
    if (messages.length > 30) return res.status(400).json({ error: 'Too many messages' });

    // Call OpenAI chat completion
    const completion = await openai.createChatCompletion({
      model: 'gpt-4o-mini', // replace with a model you have access to
      messages,
      max_tokens: 800,
      temperature: 0.2,
    });

    const reply = completion.data.choices[0].message;
    res.json({ reply });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: err.message || err.toString() });
  }
});

// Serve static build if present (for simple deploys)
const buildPath = path.join(__dirname, 'build');
if (require('fs').existsSync(buildPath)) {
  app.use(express.static(buildPath));
  app.get('*', (req, res) => {
    res.sendFile(path.join(buildPath, 'index.html'));
  });
}

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Manish AI server running on port ${port}`));
```

---

## File: frontend/src/App.jsx

```jsx
import React, { useState, useEffect, useRef } from 'react';

export default function App() {
  const [input, setInput] = useState('');
  const [messages, setMessages] = useState([
    { role: 'system', content: 'You are Manish AI — a helpful assistant.' }
  ]);
  const [loading, setLoading] = useState(false);
  const messagesEndRef = useRef(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, loading]);

  const sendMessage = async () => {
    if (!input.trim()) return;
    const userMsg = { role: 'user', content: input };
    const newMessages = [...messages, userMsg];
    setMessages(newMessages);
    setInput('');
    setLoading(true);

    try {
      const resp = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ messages: newMessages })
      });
      const data = await resp.json();
      if (data.error) throw new Error(data.error);
      setMessages(prev => [...prev, data.reply]);
    } catch (err) {
      setMessages(prev => [...prev, { role: 'assistant', content: 'Error: ' + err.message }]);
    } finally {
      setLoading(false);
    }
  };

  const handleKey = (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <div className="min-h-screen p-6 bg-gray-50 flex flex-col items-center">
      <div className="w-full max-w-3xl bg-white rounded-2xl shadow p-6">
        <header className="flex items-center justify-between mb-4">
          <h1 className="text-2xl font-bold">Manish AI</h1>
          <span className="text-sm text-gray-500">Ask anything — powered by OpenAI</span>
        </header>

        <div className="h-96 overflow-auto border rounded p-4 mb-4 bg-gray-50">
          {messages.filter(m => m.role !== 'system').map((m, i) => (
            <div key={i} className={`mb-3 ${m.role === 'user' ? 'text-right' : 'text-left'}`}>
              <div className={`inline-block p-3 rounded-lg ${m.role === 'user' ? 'bg-blue-600 text-white' : 'bg-white text-gray-800 shadow-sm'}`}>
                {m.content}
              </div>
            </div>
          ))}
          {loading && <div className="text-left text-gray-500">Manish AI is typing...</div>}
          <div ref={messagesEndRef} />
        </div>

        <div>
          <textarea
            className="w-full p-3 border rounded mb-2"
            rows={3}
            placeholder="Type a question and press Enter"
            value={input}
            onChange={e => setInput(e.target.value)}
            onKeyDown={handleKey}
          />
          <div className="flex justify-end">
            <button
              className="px-4 py-2 rounded bg-blue-600 text-white"
              onClick={sendMessage}
              disabled={loading}
            >
              {loading ? 'Waiting...' : 'Send'}
            </button>
          </div>
        </div>

      </div>
      <footer className="mt-4 text-sm text-gray-500">Tip: Press Enter to send. Don't share sensitive info.</footer>
    </div>
  );
}
```

---

## File: frontend/package.json (optional)

```
{
  "name": "manish-ai-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
```

---

## Deployment notes

- Vercel: Create a project and point to the GitHub repo; set `OPENAI_API_KEY` in Environment Variables; Vercel will give you a public URL after deploy.
- Render/Heroku: similarly set env var and deploy the Node server.
- GitHub Pages: only useful for static frontends; you'd need to host the backend separately.

---

## Security & usage tips
- Rate-limit requests in production.
- Add basic authentication or quotas if the site will be public (to avoid abuse and cost).
- Never commit your API keys to a public repository.

---

## Branding
- Project name: **Manish AI**
- Change the site title in the React header if you prefer a different display.


Enjoy! Deploying the repo to Vercel will produce the live link (Vercel assigns the URL after deployment).

