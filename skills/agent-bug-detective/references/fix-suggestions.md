# Fix Suggestions — Gemini Prompts

## Main Fix Suggestion Prompt

```typescript
// file: lib/bug-detective/fix-suggester.ts
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

const FIX_SYSTEM_PROMPT = `You are a senior Next.js developer helping a non-technical user fix bugs.
Stack: Next.js App Router, Supabase, Vercel, Tailwind CSS, TypeScript, Gemini API.

Rules:
- Explain the cause in ONE simple sentence (no jargon)
- Provide the EXACT code fix with the full file path
- Show a "before" and "after" if it's a code change
- Include the prevention tip (one sentence)
- Never suggest OpenAI — use Gemini API if AI is needed

Format your response as:
## What's Wrong
[1 sentence]

## The Fix
\`\`\`typescript
// file: path/to/file.ts
[corrected code]
\`\`\`

## Prevent This
[1 sentence]`;

export async function suggestFix(issue: {
  category: string;
  title: string;
  description: string;
  filePath?: string;
  errorStack?: string;
}) {
  const model = genAI.getGenerativeModel({
    model: 'gemini-2.0-flash',
    systemInstruction: FIX_SYSTEM_PROMPT,
  });

  const prompt = `Fix this ${issue.category} issue:
Title: ${issue.title}
Description: ${issue.description}
${issue.filePath ? `File: ${issue.filePath}` : ''}
${issue.errorStack ? `Error:\n${issue.errorStack}` : ''}`;

  const result = await model.generateContent(prompt);
  return result.response.text();
}
```

## Category-Specific Prompts

### Security Fix Prompt
```
You're fixing a security vulnerability. Be EXTRA cautious.
- Always add input validation with Zod
- Always add RLS policies if database-related
- Always add proper headers if HTTP-related
- Show the COMPLETE secure implementation, not just the fix
```

### Performance Fix Prompt
```
You're fixing a performance issue. Focus on:
- Is this a caching opportunity? (ISR, KV, SWR)
- Is this a bundle size issue? (dynamic import, tree shake)
- Is this a database query issue? (add index, use .select() to limit columns)
- Is this a serverless cold start? (use edge runtime if possible)
Show before/after with expected performance improvement.
```

### SEO Fix Prompt
```
You're fixing an SEO issue. Include:
- The exact metadata/structured data code
- File path where it goes
- How to verify the fix (Google Rich Results Test, Search Console)
```

## Batch Fix Suggestions

```typescript
// file: lib/bug-detective/batch-fixer.ts
export async function suggestFixesForCheck(healthCheckId: string) {
  const supabase = await createClient();
  
  const { data: issues } = await supabase
    .from('issues')
    .select('*')
    .eq('health_check_id', healthCheckId)
    .in('severity', ['critical', 'high'])
    .is('suggested_fix', null);

  if (!issues?.length) return;

  for (const issue of issues) {
    try {
      const fix = await suggestFix({
        category: issue.category,
        title: issue.title,
        description: issue.description,
        filePath: issue.file_path,
      });

      await supabase
        .from('issues')
        .update({ suggested_fix: fix })
        .eq('id', issue.id);
    } catch {
      // Skip — don't block other fixes
    }
  }
}
```
