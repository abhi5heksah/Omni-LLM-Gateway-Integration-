#  AI Assistant (OmniRoute + Free-Tier LLM Gateway)

A production-grade, zero-downtime AI chatbot arbrachitecture designed for beauty parlors, hair salons, and spa websites.

By leveraging an **OmniRoute / LiteLLM Proxy Gateway**, this setup pools multiple free-tier LLM endpoints (Google Gemini, Groq, Mistral, DeepSeek) behind a single OpenAI-compatible URL. If one provider hits its Rate Limit (`HTTP 429` / Quota Exceeded), traffic automatically fails over to the next provider within milliseconds.

## 📑 Table of Contents

1. [Architecture Overview](#-architecture-overview)

2. [Prerequisites & Free API Keys](#-prerequisites--free-api-keys)

3. [Quick Start (Docker Gateway)](#-quick-start-docker-gateway)

4. [Project Directory Layout](#-project-directory-layout)

5. [Configuration Files](#-configuration-files)

6. [Backend Integration (Node.js & Express)](#-backend-integration-nodejs--express)

7. [Automated Fallback & Stress Testing](#-automated-fallback--stress-testing)

8. [Frontend Embed Widget](#-frontend-embed-widget)

9. [FAQ & Production Tips](#-faq--production-tips)

## 🏛 Architecture Overview

```
[ Customer on Salon Website / Mobile ]
                   │
                   ▼
       [ Salon Express.js Server ]
                   │ (OpenAI SDK / HTTP)
                   ▼
┌───────────────────────────────────────────────────────────────┐
│               OmniRoute / LiteLLM Proxy Gateway               │
│               Exposed at: http://localhost:4000/v1            │
│  - Load Balancing & Health Checks                             │
│  - Automatic Fallback & Cooldown on 429 (Too Many Requests)   │
└───────────────────────────────┬───────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
 [ Primary Model ]      [ Fallback 1 ]          [ Fallback 2+ ]
  Gemini 1.5 Flash        Groq (Llama 3.3)         Mistral 7B
  (15 RPM Free Tier)     (30 RPM Free Tier)     (Free Experiment)

```

## 🔑 Prerequisites & Free API Keys

Sign up and obtain free API keys from these supported providers:

* **Google AI Studio (Gemini)**: [ai.google.dev](https://aistudio.google.com/)

* **Groq Cloud (Llama 3.3 / Mixtral)**: [console.groq.com](https://console.groq.com/)

* **Mistral AI**: [console.mistral.ai](https://console.mistral.ai/)

* **Docker & Docker Compose**: Installed on your server or local machine.

* **Node.js**: v18+ (for running the backend and test suite).

## 🚀 Quick Start (Docker Gateway)

### Step 1: Clone Repository & Create Environment

```
git clone https://github.com/your-username/beauty-parlor-ai.git
cd beauty-parlor-ai
cp .env.example .env

```

Fill in your `.env` file:

```
PROXY_MASTER_KEY=sk-beauty-parlor-secret-key-12345
GEMINI_API_KEY=AIzaSyYourGeminiApiKeyHere
GROQ_API_KEY=gsk_YourGroqApiKeyHere
MISTRAL_API_KEY=your_mistral_api_key_here
PORT=5000

```

### Step 2: Start the Gateway

```
docker compose up -d

```

Verify that the gateway is running:

```
curl http://localhost:4000/health
# Output: {"status": "healthy"}

```

## 📂 Project Directory Layout

```
beauty-parlor-ai/
├── config.yaml             # Gateway routing & failover rules
├── docker-compose.yml      # Containerized proxy runner
├── .env.example            # Environment template
├── promptTemplate.js       # Salon knowledge base & system prompt
├── server.js               # Node.js backend chat endpoint
├── test-suite.js           # Automated concurrency & failover test
├── package.json            # Project dependencies
└── README.md

```

## ⚙️ Configuration Files

### `config.yaml` (Routing Engine)

```
model_list:
  # Primary: Gemini 1.5 Flash
  - model_name: salon-assistant
    litellm_params:
      model: gemini/gemini-1.5-flash
      api_key: os.environ/GEMINI_API_KEY
      rpm: 15

  # Fallback 1: Groq Llama 3.3 70B (Fast inference)
  - model_name: salon-assistant
    litellm_params:
      model: groq/llama-3.3-70b-versatile
      api_key: os.environ/GROQ_API_KEY
      rpm: 30

  # Fallback 2: Mistral Small
  - model_name: salon-assistant
    litellm_params:
      model: mistral/mistral-small-latest
      api_key: os.environ/MISTRAL_API_KEY

router_settings:
  routing_strategy: "latency-based-routing"
  num_retries: 3
  retry_after: 1
  timeout: 10
  allowed_fails: 2
  cooldown_time: 60
  fallbacks:
    - salon-assistant: ["salon-assistant"]

general_settings:
  master_key: os.environ/PROXY_MASTER_KEY

```

### `docker-compose.yml`

```
version: '3.8'

services:
  omniroute-proxy:
    image: ghcr.io/berriai/litellm:main-latest
    container_name: parlor_ai_gateway
    ports:
      - "4000:4000"
    volumes:
      - ./config.yaml:/app/config.yaml
    environment:
      - PROXY_MASTER_KEY=${PROXY_MASTER_KEY}
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - GROQ_API_KEY=${GROQ_API_KEY}
      - MISTRAL_API_KEY=${MISTRAL_API_KEY}
    command: ["--config", "/app/config.yaml", "--port", "4000", "--num_workers", "4"]
    restart: always

```

## 💻 Backend Integration (Node.js & Express)

### 1. `promptTemplate.js`

```
export const SALON_SYSTEM_PROMPT = `
You are "Bella", the friendly, elegant AI Assistant for "Glow & Glamour Beauty Studio & Spa".

Your Goals:
1. Greet guests warmly and answer queries regarding beauty packages, skincare, hair styling, and nails.
2. Direct guests toward booking appointments.
3. Offer helpful recommendations based on client requirements.

Salon Menu & Base Rates:
- Bridal Makeup Package: $350 (Includes trial, HD makeup, draping, and styling)
- Signature HydraFacial: $120 (60-minute deep pore hydration & LED therapy)
- Keratin Smoothing Treatment: $180 - $250
- Balayage / Global Color: $150+
- Luxury Manicure & Pedicure: $75
- Express Brow Shaping & Tint: $25

Studio Details:
- Address: 124 Blossom Boulevard, Suite 4B
- Operating Hours: Tuesday – Sunday, 10:00 AM – 8:00 PM (Mondays Closed).
- Cancellation Policy: Full refund if cancelled 24 hours in advance.

Rules:
- Never break character. Always stay warm, polite, and professional.
- For appointments, gather: (1) Name, (2) Phone Number, (3) Desired Service, (4) Date & Preferred Time.
`;

```

### 2. `server.js`

```
import express from 'express';
import cors from 'cors';
import OpenAI from 'openai';
import dotenv from 'dotenv';
import { SALON_SYSTEM_PROMPT } from './promptTemplate.js';

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

// Point standard OpenAI SDK to your OmniRoute Gateway
const aiClient = new OpenAI({
  baseURL: 'http://localhost:4000/v1',
  apiKey: process.env.PROXY_MASTER_KEY,
});

app.post('/api/chat', async (req, res) => {
  try {
    const { message, conversationHistory = [] } = req.body;

    if (!message) {
      return res.status(400).json({ error: "Message content cannot be empty." });
    }

    const messages = [
      { role: "system", content: SALON_SYSTEM_PROMPT },
      ...conversationHistory,
      { role: "user", content: message }
    ];

    const completion = await aiClient.chat.completions.create({
      model: "salon-assistant",
      messages: messages,
      temperature: 0.7,
      max_tokens: 350,
    });

    return res.json({
      reply: completion.choices[0].message.content,
      modelUsed: completion.model
    });
  } catch (err) {
    console.error("Gateway / Server Error:", err.message);
    return res.status(500).json({
      error: "Our assistant is temporarily occupied. Please contact our reception directly at +1-800-GLAMOUR."
    });
  }
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Salon API running on http://localhost:${PORT}`));

```

## 🧪 Automated Fallback & Stress Testing

Run this script to simulate high user activity and verify that failover works under rate limiting.

### `test-suite.js`

```
import axios from 'axios';

const ENDPOINT = 'http://localhost:4000/v1/chat/completions';
const API_KEY = process.env.PROXY_MASTER_KEY || 'sk-beauty-parlor-secret-key-12345';

async function sendQuery(index) {
  try {
    const start = Date.now();
    const res = await axios.post(
      ENDPOINT,
      {
        model: 'salon-assistant',
        messages: [{ role: 'user', content: 'What is included in the bridal makeup package?' }],
        max_tokens: 40,
      },
      {
        headers: {
          Authorization: `Bearer ${API_KEY}`,
          'Content-Type': 'application/json',
        },
        timeout: 8000,
      }
    );
    const latency = Date.now() - start;
    console.log(`[Req #${index}] PASS (${latency}ms) -> Model: ${res.data.model}`);
    return true;
  } catch (err) {
    console.error(`[Req #${index}] FAILED:`, err.response?.data?.error?.message || err.message);
    return false;
  }
}

async function runTest(concurrentRequests = 10) {
  console.log(`Firing ${concurrentRequests} parallel requests to test rate-limit failovers...\n`);
  const promises = Array.from({ length: concurrentRequests }, (_, i) => sendQuery(i + 1));
  const results = await Promise.all(promises);
  const successCount = results.filter(Boolean).length;
  console.log(`\nResults: ${successCount}/${concurrentRequests} passed.`);
}

runTest(10);

```

Execute the test:

```
node test-suite.js

```

## 🌐 Frontend Embed Widget

Add this snippet to your website to connect directly to your backend endpoint:

```
<!-- Floating Chat Widget -->
<div id="salon-chat" style="position: fixed; bottom: 25px; right: 25px; z-index: 10000; font-family: sans-serif;">
  <button id="chat-open-btn" onclick="toggleChat()" style="background: #e76f51; color: #fff; padding: 12px 22px; border-radius: 30px; border: none; cursor: pointer; font-size: 14px; font-weight: bold; box-shadow: 0 4px 10px rgba(0,0,0,0.2);">
    💬 Chat with Bella
  </button>

  <div id="chat-box" style="display: none; width: 340px; height: 440px; background: #fff; border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,0.2); overflow: hidden; flex-direction: column; margin-bottom: 12px;">
    <div style="background: #e76f51; color: #fff; padding: 14px; font-weight: bold;">Glow & Glamour Concierge</div>
    <div id="chat-messages" style="flex: 1; padding: 12px; overflow-y: auto; font-size: 13px; line-height: 1.4;"></div>
    <div style="display: flex; border-top: 1px solid #eee;">
      <input id="chat-input" type="text" placeholder="Ask about services or booking..." style="flex: 1; border: none; padding: 10px; font-size: 13px; outline: none;" onkeydown="if(event.key==='Enter') sendMessage()" />
      <button onclick="sendMessage()" style="border: none; background: #e76f51; color: #fff; padding: 0 16px; cursor: pointer;">Send</button>
    </div>
  </div>
</div>

<script>
  let conversationHistory = [];

  function toggleChat() {
    const box = document.getElementById('chat-box');
    box.style.display = box.style.display === 'none' ? 'flex' : 'none';
  }

  async function sendMessage() {
    const input = document.getElementById('chat-input');
    const msg = input.value.trim();
    if (!msg) return;

    const display = document.getElementById('chat-messages');
    display.innerHTML += `<div style="margin-bottom: 8px;"><b>You:</b> ${msg}</div>`;
    input.value = '';

    try {
      const res = await fetch('http://localhost:5000/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: msg, conversationHistory })
      });
      const data = await res.json();
      
      const reply = data.reply || data.error;
      display.innerHTML += `<div style="margin-bottom: 8px; color: #444;"><b>Bella:</b> ${reply}</div>`;
      display.scrollTop = display.scrollHeight;

      conversationHistory.push({ role: 'user', content: msg });
      conversationHistory.push({ role: 'assistant', content: reply });
    } catch {
      display.innerHTML += `<div style="margin-bottom: 8px; color: red;"><b>Error:</b> Unable to reach server.</div>`;
    }
  }
</script>

```

## ❓ FAQ & Production Tips

1. **Do I need a GPU server?**

   No. The proxy simply translates HTTP calls between your backend and the cloud APIs. A minimal \$4 VPS or local computer is more than sufficient.

2. **How does rate-limiting recovery work?**

   If one model hits a `429` error, the proxy temporarily pauses requests to that specific provider for 60 seconds (defined by `cooldown_time`) and transparently shifts requests to the next model in the pool.

3. **Are client API keys exposed in the browser?**

   No. All API keys remain isolated inside the backend and Docker environment. The frontend only communicates with your secure `/api/chat` endpoint.

## 📄 License

This project is open-source and available under the [MIT License](LICENSE).
