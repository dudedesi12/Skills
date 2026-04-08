# Alert Routing and Escalation

## Slack Webhook Setup

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create New App
2. Add "Incoming Webhooks" feature
3. Create a webhook for your alerts channel
4. Add the URL as `SLACK_WEBHOOK_URL` env var

## Alert Service

```typescript
// lib/monitoring/alert-service.ts
type AlertSeverity = "info" | "warning" | "critical";

interface AlertConfig {
  info: { channels: string[] };
  warning: { channels: string[] };
  critical: { channels: string[] };
}

const ALERT_CONFIG: AlertConfig = {
  info: { channels: ["slack"] },
  warning: { channels: ["slack"] },
  critical: { channels: ["slack", "email"] },
};

export async function sendAlert(
  severity: AlertSeverity,
  title: string,
  details?: Record<string, unknown>
) {
  const config = ALERT_CONFIG[severity];

  // Dedup: don't send same alert within 5 minutes
  const dedupKey = `alert:${title}`;
  const redis = getRedis();
  const recent = await redis.get(dedupKey);
  if (recent) return; // Skip duplicate
  await redis.set(dedupKey, "1", { ex: 300 }); // 5 min dedup window

  for (const channel of config.channels) {
    switch (channel) {
      case "slack":
        await sendSlackAlert(severity, title, details);
        break;
      case "email":
        await sendEmailAlert(severity, title, details);
        break;
    }
  }
}

async function sendSlackAlert(severity: AlertSeverity, title: string, details?: Record<string, unknown>) {
  const color = { info: "#3b82f6", warning: "#f59e0b", critical: "#dc2626" }[severity];

  await fetch(process.env.SLACK_WEBHOOK_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      attachments: [{
        color,
        title: `[${severity.toUpperCase()}] ${title}`,
        text: details ? `\`\`\`${JSON.stringify(details, null, 2)}\`\`\`` : undefined,
        footer: `${process.env.VERCEL_ENV ?? "development"} | ${new Date().toISOString()}`,
      }],
    }),
  });
}

async function sendEmailAlert(severity: AlertSeverity, title: string, details?: Record<string, unknown>) {
  // Use Resend for email alerts
  const { Resend } = await import("resend");
  const resend = new Resend(process.env.RESEND_API_KEY);

  await resend.emails.send({
    from: "alerts@yourdomain.com",
    to: process.env.ADMIN_EMAIL!,
    subject: `[${severity.toUpperCase()}] ${title}`,
    text: JSON.stringify(details, null, 2),
  });
}
```

## Common Alert Triggers

```typescript
// Budget exceeded
if (budget.percentUsed > 80) {
  await sendAlert("warning", "Budget threshold reached", {
    service: "gemini",
    used: `$${budget.usage.toFixed(2)}`,
    limit: `$${budget.limit.toFixed(2)}`,
    percentUsed: budget.percentUsed,
  });
}

// Cron job failed
await sendAlert("critical", `Cron job failed: ${jobName}`, {
  error: errorMessage,
  duration: `${duration}ms`,
  jobId: heartbeatId,
});

// Slow API response
if (duration > 10000) {
  await sendAlert("warning", "Extremely slow API response", {
    route,
    duration: `${duration}ms`,
    requestId,
  });
}

// Error rate spike
if (errorRate > 0.05) { // > 5% error rate
  await sendAlert("critical", "Error rate spike detected", {
    errorRate: `${(errorRate * 100).toFixed(1)}%`,
    window: "5 minutes",
  });
}
```

## Alert Fatigue Prevention

1. **Dedup window** — Same alert title ignored for 5 minutes
2. **Severity routing** — Info goes to Slack only, Critical goes to Slack + email
3. **Throttle** — Max 10 alerts per hour per category
4. **Business hours** — Info alerts only during business hours
5. **Weekly review** — Tune thresholds based on false positive rate
