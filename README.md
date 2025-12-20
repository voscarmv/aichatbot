# aichatbot
AI chatbot framework for messaging apps. Supports concurrent messages through queueing, as well as AI tool calls.

## Telegram bot example

Demo [here.](https://github.com/voscarmv/aibot)

Using

```bash
npm install @voscarmv/aichatbot @voscarmv/aimessagestore grammy
npm install -D typescript @types/node
```

### `index.ts`
```typescript

import { aiClient } from "./bot.js";
import { messageStore } from "./api.js";
import { ChatService } from "@voscarmv/aichatbot";
import { Bot } from "grammy";
import "dotenv/config";

const chat = new ChatService({
    aiClient,
    messageStore
});
if (!process.env.TELEGRAM_KEY) {
    throw new Error("TELEGRAM_KEY is not defined");
}
const bot = new Bot(process.env.TELEGRAM_KEY);

bot.on("message:text", async (ctx) => {
    const from = ctx.message.from.id.toString();
    const content = ctx.message.text;
    console.log(new Date(), "from:", from, content);
    const reply = (content: string) => {
        if(content.length === 0) return;
        console.log(new Date(), "to:", from, content);
        ctx.reply(content);
    }
    chat.processMessages(from, content, reply);
});

bot.start();
```

### `bot.ts`

```typescript
import { OpenAiClient } from "@voscarmv/aichatbot";
import "dotenv/config";
if (!process.env.DEEPSEEK_KEY) {
    throw new Error("DEEPSEEK_KEY is not defined");
}
export const aiClient = new OpenAiClient({
    baseURL: 'https://api.deepseek.com',
    apiKey: process.env.DEEPSEEK_KEY,
    model: 'deepseek-chat',
    instructions: 'You are a helpful assistant.',
});
```

### `api.ts`

```typescript
import 'dotenv/config';
import { FunctionMessageStore } from '@voscarmv/aichatbot';
import axios from 'axios';

if (!process.env.MESSAGESTORE_URL) {
    throw new Error("MESSAGESTORE_URL is not defined");
}
const url = process.env.MESSAGESTORE_URL;

async function readUserMessages(user_id: string): Promise<string[]> {
    return (await axios.get<string[]>(`${url}/messages/${user_id}`)).data;
}

async function unqueueUserMessages(user_id: string): Promise<string[]> {
    return (await axios.put<string[]>(`${url}/messages/${user_id}/unqueue`)).data;
}

async function insertMessages(user_id: string, queued: boolean, msgs: string[]): Promise<string[]> {
    return (await axios.post<string[]>( `${url}/messages`, {user_id, queued, msgs})).data;
}

async function queuedMessages(user_id: string): Promise<string[]> {
    return (await axios.get<string[]>(`${url}/messages/${user_id}/queued`)).data;
}

export const messageStore = new FunctionMessageStore({
    readUserMessages,
    unqueueUserMessages,
    insertMessages,
    queuedMessages
});
```

### `server.ts`

```typescript
import { AiMessageStoreBackend } from "@voscarmv/aimessagestore";
import 'dotenv/config';

if (!process.env.DATABASE_URL) {
    throw Error('DATABASE_URL undefined');
}

const server = new AiMessageStoreBackend({
    dbUrl: process.env.DATABASE_URL,
    port: 3000
});

(async () => {
    console.log("Migrate DB");
    await server.migrate()
    console.log("Done Migrating DB. Start server...");
    server.listen();
})();

```

### `tools.ts`

```typescript
export const tools = [
    {
        type: 'function' as const,
        function: {
            name: 'getDate',
            description: 'Get the current date and time',
            parameters: {
                type: 'object',
                properties: {
                    language: {
                        type: 'string',
                        enum: ['en-US', 'es-MX'],
                        description: 'The language of the date and time',
                    }
                },
                required: ['language']
            }
        }
    },
    {
        type: 'function' as const,
        function: {
            name: 'throwDice',
            description: 'Throw N dice and get the results',
            parameters: {
                type: 'object',
                properties: {
                    nDice: {
                        type: 'integer',
                        description: 'Number N of dice to throw'
                    }
                },
                required: ['nDice']
            }
        }
    },
];

export const functions = {
    getDate: async (params: any, additionalArgs: any): Promise<string> => {
        const { language } = params;
        const myDate = new Date();
        return myDate.toLocaleString(language);
    },
    throwDice: async (params: any, additionalArgs: any): Promise<string> => {
        const { nDice } = params;
        let result = [];
        function sleep(ms: number) {
            return new Promise(resolve => setTimeout(resolve, ms));
        }
        await sleep(5000);
        for (let i = 0; i < nDice; i++) {
            result.push(Math.floor(Math.random() * 6) + 1);
        }
        return JSON.stringify(result);
    },
};
```

## About `api.ts`

You can use a different backend for `messageStore()` in `api.ts`.

Just follow the same storage and retrieval logic from [@voscarmv/aimessagestore](https://github.com/voscarmv/aimessagestore).
