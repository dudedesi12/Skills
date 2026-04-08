# Gemini API Cost Guide

## Pricing (as of 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Best For |
|-------|----------------------|----------------------|----------|
| gemini-2.0-flash-lite | $0.075 | $0.30 | Classification, extraction, formatting |
| gemini-2.0-flash | $0.10 | $0.40 | Summarization, Q&A, general tasks |
| gemini-2.5-pro | $1.25 | $10.00 | Complex analysis, reasoning, reports |

**Key insight:** gemini-2.5-pro output is **25x** more expensive than flash. Route carefully.

## Token Counting

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function countTokens(text: string, modelName = "gemini-2.0-flash"): Promise<number> {
  const model = genAI.getGenerativeModel({ model: modelName });
  const result = await model.countTokens(text);
  return result.totalTokens;
}

// Rough estimation without API call (1 token ≈ 4 characters in English)
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}
```

## Prompt Optimization

### Reduce Input Tokens

```typescript
// BAD: Verbose prompt (500+ tokens)
const verbosePrompt = `
  You are an expert immigration consultant specializing in Australian skilled migration.
  I would like you to carefully analyze the following user profile and determine their
  eligibility for the subclass 189 visa. Please consider all factors including age,
  English language ability, work experience, qualifications, and any other relevant
  criteria. Here is the profile:
  ${JSON.stringify(fullProfile, null, 2)}
`;

// GOOD: Concise prompt (150 tokens)
const concisePrompt = `
Assess 189 visa eligibility. Return JSON: {eligible: boolean, score: number, reasons: string[]}.
Profile: ${JSON.stringify({
  age: profile.age,
  english: profile.englishLevel,
  experience: profile.yearsExperience,
  qualification: profile.qualificationLevel,
  occupation: profile.occupation,
})}`;
```

### Reduce Output Tokens

```typescript
// BAD: Open-ended output
const prompt = "Analyze this user's visa options and explain each one in detail.";

// GOOD: Constrained output
const prompt = `List visa options as JSON array: [{subclass: string, eligible: boolean, score: number}]. Max 5 options.`;
```

## Cost Estimation Helper

```typescript
// lib/ai/cost-estimator.ts
const COSTS = {
  "gemini-2.0-flash-lite": { input: 0.075 / 1_000_000, output: 0.30 / 1_000_000 },
  "gemini-2.0-flash": { input: 0.10 / 1_000_000, output: 0.40 / 1_000_000 },
  "gemini-2.5-pro": { input: 1.25 / 1_000_000, output: 10.00 / 1_000_000 },
};

export function estimateCost(model: string, inputTokens: number, outputTokens: number): number {
  const cost = COSTS[model as keyof typeof COSTS];
  if (!cost) return 0;
  return inputTokens * cost.input + outputTokens * cost.output;
}

// Example: 1000 input + 500 output on flash = $0.0003
// Example: 1000 input + 500 output on pro = $0.00625 (20x more!)
```

## Free Tier Limits

| Model | Free RPM | Free TPM | Free RPD |
|-------|----------|----------|----------|
| gemini-2.0-flash-lite | 30 | 1M | 1,500 |
| gemini-2.0-flash | 15 | 1M | 1,500 |
| gemini-2.5-pro | 2 | 32K | 50 |

Stay within free tier by: routing simple tasks to flash-lite, caching responses, batching requests.
