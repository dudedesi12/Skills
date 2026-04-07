# Routing Logic — Task Decision Tree

## How the Router Decides

The master agent uses Gemini Flash to classify incoming tasks. Here's the decision tree it follows:

```
Is the task about data collection?
  └─ Yes → scraper-agent
      └─ Also needs parsing? → sequential: scraper → parser

Is the task about content?
  └─ Yes → content-agent
      └─ Needs SEO? → sequential: content → seo-agent

Is the task about analysis?
  └─ Yes → analysis-agent
      └─ Needs report? → sequential: analysis → report-agent

Is the task about multiple topics?
  └─ Yes → parallel workflow with relevant agents

Is the task unclear?
  └─ Yes → flash-lite classification first → then route
```

## Routing Configuration

```typescript
// file: lib/agents/routing-config.ts

export const ROUTING_RULES: RoutingRule[] = [
  {
    keywords: ['scrape', 'crawl', 'extract', 'website data', 'government portal'],
    agent: 'scraper-agent',
    followUp: 'parser-agent', // Often needed after scraping
  },
  {
    keywords: ['parse', 'structure', 'extract fields', 'PDF', 'normalize'],
    agent: 'parser-agent',
  },
  {
    keywords: ['write', 'blog', 'content', 'article', 'SEO content', 'landing page'],
    agent: 'content-agent',
  },
  {
    keywords: ['analyze', 'calculate', 'score', 'predict', 'compare', 'statistics'],
    agent: 'analysis-agent',
  },
  {
    keywords: ['notify', 'email', 'alert', 'push notification', 'remind'],
    agent: 'notification-agent',
  },
  {
    keywords: ['moderate', 'review', 'spam', 'abuse', 'quality check'],
    agent: 'moderation-agent',
  },
  {
    keywords: ['search', 'find', 'look up', 'query'],
    agent: 'search-agent',
  },
  {
    keywords: ['report', 'dashboard', 'summary', 'PDF report', 'export'],
    agent: 'report-agent',
  },
  {
    keywords: ['translate', 'Hindi', 'Hinglish', 'language'],
    agent: 'translation-agent',
  },
  {
    keywords: ['support', 'help', 'question', 'FAQ', 'customer'],
    agent: 'support-agent',
  },
];

interface RoutingRule {
  keywords: string[];
  agent: string;
  followUp?: string;
}
```

## Workflow Type Selection

```typescript
// file: lib/agents/workflow-selector.ts

export function selectWorkflowType(steps: { agent: string }[]): 'sequential' | 'parallel' | 'conditional' {
  if (steps.length === 1) return 'sequential'; // Single agent

  // Check for dependencies between agents
  const hasDependent = steps.some((step, i) => {
    const dependsOn = AGENT_DEPENDENCIES[step.agent];
    return dependsOn?.some(dep => steps.slice(0, i).some(s => s.agent === dep));
  });

  if (hasDependent) return 'sequential';
  return 'parallel'; // Independent agents run in parallel
}

const AGENT_DEPENDENCIES: Record<string, string[]> = {
  'parser-agent': ['scraper-agent'],     // Parser needs scraper output
  'report-agent': ['analysis-agent'],    // Report needs analysis output
  'notification-agent': ['moderation-agent'], // Notify after moderation
};
```
