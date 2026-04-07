# Gemini API Integration

Complete setup for all 3 Gemini model tiers with streaming, structured output, function calling, content generation, and image analysis.

## Setup

```bash
npm install @google/generative-ai
```

```bash
# .env.local
GEMINI_API_KEY=AIzaSy...
```

## Model Configuration

```typescript
// lib/gemini.ts
import {
  GoogleGenerativeAI,
  HarmCategory,
  HarmBlockThreshold,
} from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) {
  throw new Error("GEMINI_API_KEY environment variable is not set");
}

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

const safetySettings = [
  {
    category: HarmCategory.HARM_CATEGORY_HARASSMENT,
    threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
  },
  {
    category: HarmCategory.HARM_CATEGORY_HATE_SPEECH,
    threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
  },
  {
    category: HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
    threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
  },
  {
    category: HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
    threshold: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
  },
];

// Heavy processing — PDF parsing, complex analysis, cron jobs
export const geminiPro = genAI.getGenerativeModel({
  model: "gemini-2.5-pro",
  safetySettings,
});

// User-facing features — chat, search, real-time responses
export const geminiFlash = genAI.getGenerativeModel({
  model: "gemini-2.0-flash",
  safetySettings,
});

// Classification, tagging, simple decisions
export const geminiLite = genAI.getGenerativeModel({
  model: "gemini-2.0-flash-lite",
  safetySettings,
});
```

## Streaming Responses (Chat)

```typescript
// app/api/chat/route.ts
import { NextRequest } from "next/server";
import { geminiFlash } from "@/lib/gemini";

export async function POST(request: NextRequest) {
  try {
    const { message, history } = await request.json();

    if (!message || typeof message !== "string") {
      return new Response(JSON.stringify({ error: "Message is required" }), {
        status: 400,
        headers: { "Content-Type": "application/json" },
      });
    }

    // Build chat with history
    const chat = geminiFlash.startChat({
      history: (history ?? []).map((msg: { role: string; text: string }) => ({
        role: msg.role === "user" ? "user" : "model",
        parts: [{ text: msg.text }],
      })),
    });

    const result = await chat.sendMessageStream(message);

    const encoder = new TextEncoder();
    const stream = new ReadableStream({
      async start(controller) {
        try {
          for await (const chunk of result.stream) {
            const text = chunk.text();
            if (text) {
              controller.enqueue(
                encoder.encode(`data: ${JSON.stringify({ text })}\n\n`)
              );
            }
          }
          controller.enqueue(encoder.encode("data: [DONE]\n\n"));
          controller.close();
        } catch (err) {
          const errorMsg = err instanceof Error ? err.message : "Stream error";
          controller.enqueue(
            encoder.encode(`data: ${JSON.stringify({ error: errorMsg })}\n\n`)
          );
          controller.close();
        }
      },
    });

    return new Response(stream, {
      headers: {
        "Content-Type": "text/event-stream",
        "Cache-Control": "no-cache",
        Connection: "keep-alive",
      },
    });
  } catch (err) {
    console.error("Chat API error:", err);
    return new Response(JSON.stringify({ error: "Chat failed" }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
}
```

## Client-Side Streaming Hook

```typescript
// hooks/use-chat.ts
"use client";
import { useState, useCallback } from "react";

interface Message {
  role: "user" | "assistant";
  text: string;
}

export function useChat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isStreaming, setIsStreaming] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const sendMessage = useCallback(async (text: string) => {
    setError(null);
    setIsStreaming(true);

    const userMessage: Message = { role: "user", text };
    setMessages((prev) => [...prev, userMessage]);

    try {
      const response = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          message: text,
          history: messages.map((m) => ({
            role: m.role === "user" ? "user" : "model",
            text: m.text,
          })),
        }),
      });

      if (!response.ok) throw new Error("Chat request failed");
      if (!response.body) throw new Error("No response body");

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let assistantText = "";

      setMessages((prev) => [...prev, { role: "assistant", text: "" }]);

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });
        const lines = chunk.split("\n");

        for (const line of lines) {
          if (line.startsWith("data: ")) {
            const data = line.slice(6);
            if (data === "[DONE]") break;

            try {
              const parsed = JSON.parse(data);
              if (parsed.error) throw new Error(parsed.error);
              if (parsed.text) {
                assistantText += parsed.text;
                setMessages((prev) => {
                  const updated = [...prev];
                  updated[updated.length - 1] = {
                    role: "assistant",
                    text: assistantText,
                  };
                  return updated;
                });
              }
            } catch {}
          }
        }
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : "Something went wrong");
    } finally {
      setIsStreaming(false);
    }
  }, [messages]);

  return { messages, sendMessage, isStreaming, error };
}
```

## Structured Output (JSON)

```typescript
// lib/ai/extract-data.ts
import { geminiFlash } from "@/lib/gemini";

interface ExtractedContact {
  name: string;
  email: string | null;
  phone: string | null;
  company: string | null;
}

export async function extractContactInfo(
  text: string
): Promise<ExtractedContact> {
  try {
    const result = await geminiFlash.generateContent([
      {
        text: `Extract contact information from this text. Return ONLY valid JSON with these fields:
{
  "name": "string",
  "email": "string or null",
  "phone": "string or null",
  "company": "string or null"
}

Text: ${text}`,
      },
    ]);

    const responseText = result.response.text();
    const jsonMatch = responseText.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      throw new Error("No JSON found in Gemini response");
    }

    return JSON.parse(jsonMatch[0]) as ExtractedContact;
  } catch (err) {
    console.error("Contact extraction failed:", err);
    return { name: "Unknown", email: null, phone: null, company: null };
  }
}
```

## Function Calling

```typescript
// lib/ai/assistant.ts
import { geminiFlash } from "@/lib/gemini";
import {
  FunctionDeclarationSchemaType,
  type FunctionDeclaration,
} from "@google/generative-ai";

const tools: FunctionDeclaration[] = [
  {
    name: "get_weather",
    description: "Get current weather for a location",
    parameters: {
      type: FunctionDeclarationSchemaType.OBJECT,
      properties: {
        location: {
          type: FunctionDeclarationSchemaType.STRING,
          description: "City name, e.g., 'Sydney, Australia'",
        },
      },
      required: ["location"],
    },
  },
  {
    name: "search_products",
    description: "Search for products in the catalog",
    parameters: {
      type: FunctionDeclarationSchemaType.OBJECT,
      properties: {
        query: {
          type: FunctionDeclarationSchemaType.STRING,
          description: "Search query",
        },
        maxResults: {
          type: FunctionDeclarationSchemaType.NUMBER,
          description: "Maximum number of results",
        },
      },
      required: ["query"],
    },
  },
];

// Implement the actual functions
async function executeFunction(
  name: string,
  args: Record<string, unknown>
): Promise<string> {
  switch (name) {
    case "get_weather":
      // Call your weather API
      return JSON.stringify({
        location: args.location,
        temperature: 22,
        conditions: "Sunny",
      });
    case "search_products":
      // Query your database
      return JSON.stringify({
        results: [{ name: "Example Product", price: 29.99 }],
      });
    default:
      return JSON.stringify({ error: "Unknown function" });
  }
}

export async function chatWithTools(message: string): Promise<string> {
  try {
    const model = geminiFlash;
    const chat = model.startChat({
      tools: [{ functionDeclarations: tools }],
    });

    const result = await chat.sendMessage(message);
    const response = result.response;

    // Check if the model wants to call a function
    const functionCall = response.functionCalls()?.[0];
    if (functionCall) {
      const functionResult = await executeFunction(
        functionCall.name,
        functionCall.args as Record<string, unknown>
      );

      // Send the function result back to the model
      const followUp = await chat.sendMessage([
        {
          functionResponse: {
            name: functionCall.name,
            response: JSON.parse(functionResult),
          },
        },
      ]);

      return followUp.response.text();
    }

    return response.text();
  } catch (err) {
    console.error("Chat with tools failed:", err);
    throw new Error("AI assistant error");
  }
}
```

## Image Analysis

```typescript
// lib/ai/image-analysis.ts
import { geminiFlash } from "@/lib/gemini";

export async function analyzeImage(
  imageBuffer: Buffer,
  mimeType: string,
  prompt: string = "Describe this image in detail"
): Promise<string> {
  try {
    const result = await geminiFlash.generateContent([
      { text: prompt },
      {
        inlineData: {
          data: imageBuffer.toString("base64"),
          mimeType,
        },
      },
    ]);

    return result.response.text();
  } catch (err) {
    console.error("Image analysis failed:", err);
    throw new Error("Failed to analyze image");
  }
}
```

```typescript
// app/api/analyze-image/route.ts
import { NextRequest, NextResponse } from "next/server";
import { analyzeImage } from "@/lib/ai/image-analysis";

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get("image") as File | null;
    const prompt = formData.get("prompt") as string | null;

    if (!file) {
      return NextResponse.json({ error: "No image provided" }, { status: 400 });
    }

    const allowedTypes = ["image/jpeg", "image/png", "image/webp", "image/gif"];
    if (!allowedTypes.includes(file.type)) {
      return NextResponse.json({ error: "Invalid image type" }, { status: 400 });
    }

    if (file.size > 10 * 1024 * 1024) {
      return NextResponse.json({ error: "Image too large (max 10MB)" }, { status: 400 });
    }

    const buffer = Buffer.from(await file.arrayBuffer());
    const description = await analyzeImage(
      buffer,
      file.type,
      prompt ?? "Describe this image in detail"
    );

    return NextResponse.json({ description });
  } catch (err) {
    console.error("Image analysis API error:", err);
    return NextResponse.json({ error: "Analysis failed" }, { status: 500 });
  }
}
```

## Heavy Processing with Pro (Cron Jobs)

```typescript
// lib/ai/document-processor.ts
import { geminiPro } from "@/lib/gemini";

interface DocumentSummary {
  title: string;
  summary: string;
  keyTopics: string[];
  sentiment: "positive" | "neutral" | "negative";
  wordCount: number;
}

export async function processDocument(
  content: string
): Promise<DocumentSummary> {
  try {
    const result = await geminiPro.generateContent([
      {
        text: `Analyze this document thoroughly. Return ONLY valid JSON:
{
  "title": "inferred title",
  "summary": "2-3 sentence summary",
  "keyTopics": ["topic1", "topic2"],
  "sentiment": "positive" | "neutral" | "negative",
  "wordCount": number
}

Document:
${content}`,
      },
    ]);

    const text = result.response.text();
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) throw new Error("No JSON in response");

    return JSON.parse(jsonMatch[0]) as DocumentSummary;
  } catch (err) {
    console.error("Document processing failed:", err);
    throw new Error("Failed to process document");
  }
}
```

## Classification with Lite (Fast + Cheap)

```typescript
// lib/ai/classifier.ts
import { geminiLite } from "@/lib/gemini";

type TicketCategory = "billing" | "technical" | "general" | "urgent";
type Sentiment = "positive" | "neutral" | "negative";

export async function classifyTicket(text: string): Promise<TicketCategory> {
  try {
    const result = await geminiLite.generateContent(
      `Classify this support ticket into exactly one category: billing, technical, general, urgent. Reply with ONLY the category word, nothing else.\n\nTicket: ${text}`
    );

    const category = result.response.text().trim().toLowerCase();
    const valid: TicketCategory[] = ["billing", "technical", "general", "urgent"];
    return valid.includes(category as TicketCategory)
      ? (category as TicketCategory)
      : "general";
  } catch {
    return "general";
  }
}

export async function analyzeSentiment(text: string): Promise<Sentiment> {
  try {
    const result = await geminiLite.generateContent(
      `What is the sentiment of this text? Reply with ONLY one word: positive, neutral, or negative.\n\nText: ${text}`
    );

    const sentiment = result.response.text().trim().toLowerCase();
    const valid: Sentiment[] = ["positive", "neutral", "negative"];
    return valid.includes(sentiment as Sentiment)
      ? (sentiment as Sentiment)
      : "neutral";
  } catch {
    return "neutral";
  }
}

export async function tagContent(text: string): Promise<string[]> {
  try {
    const result = await geminiLite.generateContent(
      `Generate 3-5 relevant tags for this content. Return ONLY a JSON array of strings, nothing else.\n\nContent: ${text}`
    );

    const response = result.response.text().trim();
    const tags = JSON.parse(response);
    return Array.isArray(tags) ? tags.slice(0, 5) : [];
  } catch {
    return [];
  }
}
```
