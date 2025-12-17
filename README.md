# aichatbot
AI chatbot framework for messaging apps. Supports concurrent messages through queueing, as well as AI tool calls.

## Telegram bot example

Demo [here.](https://github.com/voscarmv/aibot2)

Using

```bash
npm install @voscarmv/aichatbot grammy drizzle-orm pg
npm install -D typescript drizzle-kit @types/pg tsx
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
import { messages } from './schema.js';
import { sql, eq, and, asc } from 'drizzle-orm';
import { drizzle } from 'drizzle-orm/node-postgres';
import { FunctionMessageStore } from '@voscarmv/aichatbot';

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL is not defined");
}
const db = drizzle(process.env.DATABASE_URL);

async function readUserMessages(user_id: string): Promise<string[]> {
    const response: { message: string }[] = await db
        .select({ message: messages.message })
        .from(messages)
        .where(eq(messages.user_id, user_id))
        .orderBy(asc(messages.updated_at), asc(messages.id));
    return response.map(item => item.message);
}

async function unqueueUserMessages(user_id: string): Promise<string[]> {
    const response: { message: string }[] = await db
        .update(messages)
        .set({ queued: false, updated_at: sql`now()` })
        .where(eq(messages.user_id, user_id))
        .returning({ message: messages.message });
    return response.map(item => item.message);
}

async function insertMessages(user_id: string, queued: boolean, msgs: string[]): Promise<string[]> {
    const response: { message: string }[] = await db
        .insert(messages)
        .values(msgs.map((msg) => (
            {
                user_id,
                queued,
                message: msg
            }
        )))
        .returning({ message: messages.message });
    return response.map(item => item.message);
}

async function queuedMessages(user_id: string): Promise<string[]> {
    const response: { message: string }[] = await db
        .select({ message: messages.message })
        .from(messages)
        .where(
            and(
                eq(messages.queued, true),
                eq(messages.user_id, user_id)
            ));
    return response.map(item => item.message);
}

export const messageStore = new FunctionMessageStore({
    readUserMessages,
    unqueueUserMessages,
    insertMessages,
    queuedMessages
});
```

### `schema.ts`

```typescript
import { boolean, bigint, timestamp, text, pgTable, varchar } from "drizzle-orm/pg-core";

export const messages = pgTable("messages", {
  id: bigint({ mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  user_id: varchar({ length: 255 }).notNull(),
  message: text().notNull(),
  queued: boolean().notNull(),
  created_at: timestamp('created_at').notNull().defaultNow(),
  updated_at: timestamp('updated_at').notNull().defaultNow()
});
```

## About `api.ts`

You can use a different backend for `messageStore()` in `api.ts` instead of direct database queries. For example `axios` calls to an external server such as `express`.

Just follow the same storage and retrieval logic as in `schema.ts` and `api.ts`.