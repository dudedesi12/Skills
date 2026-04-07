# React Email Templates Reference

This file contains 8 production-ready React Email templates. Copy and adjust them for your app.

## Shared Styles

All templates use these shared styles. You can put them in a separate file or copy them into each template:

```tsx
// emails/styles.ts
export const main = { backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" };
export const container = { margin: "0 auto", padding: "40px 20px", maxWidth: "560px" };
export const heading = { fontSize: "24px", color: "#1a1a1a", marginBottom: "16px" };
export const text = { fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" };
export const buttonContainer = { textAlign: "center" as const, margin: "24px 0" };
export const primaryButton = {
  backgroundColor: "#2563eb",
  borderRadius: "6px",
  color: "#fff",
  fontSize: "16px",
  padding: "12px 24px",
  textDecoration: "none",
};
export const hr = { borderColor: "#e5e7eb", margin: "24px 0" };
export const footer = { fontSize: "12px", color: "#9ca3af" };
export const smallText = { fontSize: "14px", color: "#6b7280" };
```

## 1. Welcome Email

Sent after a user verifies their email and their account is fully active.

```tsx
// emails/welcome.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr, Preview, Section } from "@react-email/components";

interface Props {
  userName: string;
  dashboardUrl: string;
}

export default function WelcomeEmail({ userName, dashboardUrl }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to the platform, {userName}!</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Welcome, {userName}!</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Your account is ready. Here is what you can do next:
          </Text>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            1. Complete your profile{"\n"}
            2. Explore the dashboard{"\n"}
            3. Set up your first project
          </Text>
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={dashboardUrl} style={{ backgroundColor: "#2563eb", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              Go to Dashboard
            </Button>
          </Section>
        </Container>
      </Body>
    </Html>
  );
}
```

## 2. Verify Email

Sent immediately after signup to confirm the email address is real.

```tsx
// emails/verify-email.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr, Preview, Section } from "@react-email/components";

interface Props {
  verificationUrl: string;
  expiresInHours: number;
}

export default function VerifyEmail({ verificationUrl, expiresInHours }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Verify your email address</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Verify Your Email</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Click the button below to verify your email address. This link expires in {expiresInHours} hours.
          </Text>
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={verificationUrl} style={{ backgroundColor: "#2563eb", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              Verify Email
            </Button>
          </Section>
          <Text style={{ fontSize: "12px", color: "#9ca3af", wordBreak: "break-all" as const }}>
            Or copy this URL: {verificationUrl}
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

## 3. Password Reset

Sent when a user requests to reset their password.

```tsx
// emails/password-reset.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr, Preview, Section } from "@react-email/components";

interface Props {
  resetUrl: string;
  userName: string;
}

export default function PasswordReset({ resetUrl, userName }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Reset your password</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Password Reset</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Hi {userName}, someone requested a password reset for your account.
          </Text>
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={resetUrl} style={{ backgroundColor: "#dc2626", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              Reset Password
            </Button>
          </Section>
          <Text style={{ fontSize: "14px", color: "#6b7280" }}>
            This link expires in 1 hour. If you did not request this, ignore this email.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

## 4. Order Confirmation

Sent after a successful purchase.

```tsx
// emails/order-confirmation.tsx
import { Html, Head, Body, Container, Text, Heading, Hr, Preview, Section, Row, Column } from "@react-email/components";

interface OrderItem {
  name: string;
  quantity: number;
  price: number;
}

interface Props {
  orderId: string;
  items: OrderItem[];
  total: number;
  customerName: string;
}

export default function OrderConfirmation({ orderId, items, total, customerName }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Order #{orderId} confirmed</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Order Confirmed</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Hi {customerName}, your order #{orderId} has been confirmed.
          </Text>
          <Hr style={{ borderColor: "#e5e7eb", margin: "24px 0" }} />
          {items.map((item, index) => (
            <Section key={index} style={{ marginBottom: "8px" }}>
              <Row>
                <Column style={{ width: "60%" }}>
                  <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "0" }}>
                    {item.name} x {item.quantity}
                  </Text>
                </Column>
                <Column style={{ width: "40%", textAlign: "right" as const }}>
                  <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "0" }}>
                    ${(item.price * item.quantity).toFixed(2)}
                  </Text>
                </Column>
              </Row>
            </Section>
          ))}
          <Hr style={{ borderColor: "#e5e7eb", margin: "16px 0" }} />
          <Section>
            <Row>
              <Column style={{ width: "60%" }}>
                <Text style={{ fontSize: "16px", fontWeight: "bold", color: "#1a1a1a", margin: "0" }}>Total</Text>
              </Column>
              <Column style={{ width: "40%", textAlign: "right" as const }}>
                <Text style={{ fontSize: "16px", fontWeight: "bold", color: "#1a1a1a", margin: "0" }}>
                  ${total.toFixed(2)}
                </Text>
              </Column>
            </Row>
          </Section>
        </Container>
      </Body>
    </Html>
  );
}
```

## 5. Subscription Changed

Sent when a user upgrades, downgrades, or cancels their subscription.

```tsx
// emails/subscription-changed.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr, Preview, Section } from "@react-email/components";

interface Props {
  userName: string;
  previousPlan: string;
  newPlan: string;
  changeType: "upgrade" | "downgrade" | "cancel";
  effectiveDate: string;
  manageUrl: string;
}

export default function SubscriptionChanged({ userName, previousPlan, newPlan, changeType, effectiveDate, manageUrl }: Props) {
  const titles = { upgrade: "Subscription Upgraded", downgrade: "Subscription Changed", cancel: "Subscription Cancelled" };
  const previews = { upgrade: `You've upgraded to ${newPlan}`, downgrade: `Your plan changed to ${newPlan}`, cancel: "Your subscription has been cancelled" };

  return (
    <Html>
      <Head />
      <Preview>{previews[changeType]}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>{titles[changeType]}</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Hi {userName},
          </Text>
          {changeType === "cancel" ? (
            <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
              Your {previousPlan} subscription has been cancelled. You will have access until {effectiveDate}.
            </Text>
          ) : (
            <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
              Your plan has been changed from {previousPlan} to {newPlan}, effective {effectiveDate}.
            </Text>
          )}
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={manageUrl} style={{ backgroundColor: "#2563eb", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              Manage Subscription
            </Button>
          </Section>
        </Container>
      </Body>
    </Html>
  );
}
```

## 6. Weekly Digest

A summary email sent on a schedule with key metrics or updates.

```tsx
// emails/weekly-digest.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr, Preview, Section } from "@react-email/components";

interface DigestItem {
  title: string;
  summary: string;
  url: string;
}

interface Props {
  userName: string;
  weekOf: string;
  items: DigestItem[];
  stats: { views: number; signups: number; revenue: number };
  dashboardUrl: string;
  unsubscribeUrl: string;
}

export default function WeeklyDigest({ userName, weekOf, items, stats, dashboardUrl, unsubscribeUrl }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Your weekly summary for {weekOf}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Weekly Summary</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a" }}>Hi {userName}, here is your week in review ({weekOf}):</Text>
          <Section style={{ backgroundColor: "#fff", padding: "16px", borderRadius: "8px", margin: "16px 0" }}>
            <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "4px 0" }}>Views: {stats.views.toLocaleString()}</Text>
            <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "4px 0" }}>New signups: {stats.signups}</Text>
            <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "4px 0" }}>Revenue: ${stats.revenue.toFixed(2)}</Text>
          </Section>
          <Hr style={{ borderColor: "#e5e7eb", margin: "24px 0" }} />
          {items.map((item, i) => (
            <Section key={i} style={{ marginBottom: "16px" }}>
              <Text style={{ fontSize: "16px", fontWeight: "bold", color: "#1a1a1a", margin: "0 0 4px" }}>{item.title}</Text>
              <Text style={{ fontSize: "14px", color: "#6b7280", margin: "0" }}>{item.summary}</Text>
            </Section>
          ))}
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={dashboardUrl} style={{ backgroundColor: "#2563eb", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              View Dashboard
            </Button>
          </Section>
          <Hr style={{ borderColor: "#e5e7eb", margin: "24px 0" }} />
          <Text style={{ fontSize: "12px", color: "#9ca3af", textAlign: "center" as const }}>
            <a href={unsubscribeUrl} style={{ color: "#6b7280" }}>Unsubscribe</a> from weekly digests.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

## 7. Alert / Notification

A general-purpose notification for things like security alerts, usage limits, or system events.

```tsx
// emails/alert-notification.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Preview, Section } from "@react-email/components";

interface Props {
  userName: string;
  alertType: "info" | "warning" | "error";
  title: string;
  message: string;
  actionUrl?: string;
  actionLabel?: string;
}

export default function AlertNotification({ userName, alertType, title, message, actionUrl, actionLabel }: Props) {
  const colors = { info: "#2563eb", warning: "#d97706", error: "#dc2626" };
  const borderColor = colors[alertType];

  return (
    <Html>
      <Head />
      <Preview>{title}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Section style={{ borderLeft: `4px solid ${borderColor}`, paddingLeft: "16px", marginBottom: "24px" }}>
            <Heading style={{ fontSize: "20px", color: "#1a1a1a", margin: "0 0 8px" }}>{title}</Heading>
            <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px", margin: "0" }}>
              Hi {userName}, {message}
            </Text>
          </Section>
          {actionUrl && actionLabel && (
            <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
              <Button href={actionUrl} style={{ backgroundColor: borderColor, borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
                {actionLabel}
              </Button>
            </Section>
          )}
        </Container>
      </Body>
    </Html>
  );
}
```

## 8. Invoice

Sent after a payment is processed, with a summary of charges.

```tsx
// emails/invoice.tsx
import { Html, Head, Body, Container, Text, Heading, Hr, Preview, Section, Row, Column } from "@react-email/components";

interface LineItem {
  description: string;
  amount: number;
}

interface Props {
  invoiceNumber: string;
  customerName: string;
  date: string;
  dueDate: string;
  lineItems: LineItem[];
  total: number;
  paidAt?: string;
}

export default function Invoice({ invoiceNumber, customerName, date, dueDate, lineItems, total, paidAt }: Props) {
  return (
    <Html>
      <Head />
      <Preview>Invoice #{invoiceNumber}{paidAt ? " — Paid" : ""}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Section style={{ marginBottom: "24px" }}>
            <Row>
              <Column>
                <Heading style={{ fontSize: "24px", color: "#1a1a1a", margin: "0" }}>Invoice</Heading>
              </Column>
              <Column style={{ textAlign: "right" as const }}>
                {paidAt ? (
                  <Text style={{ fontSize: "14px", color: "#16a34a", fontWeight: "bold", margin: "0" }}>PAID</Text>
                ) : (
                  <Text style={{ fontSize: "14px", color: "#d97706", fontWeight: "bold", margin: "0" }}>PENDING</Text>
                )}
              </Column>
            </Row>
          </Section>

          <Text style={{ fontSize: "14px", color: "#6b7280", margin: "4px 0" }}>Invoice: #{invoiceNumber}</Text>
          <Text style={{ fontSize: "14px", color: "#6b7280", margin: "4px 0" }}>Date: {date}</Text>
          <Text style={{ fontSize: "14px", color: "#6b7280", margin: "4px 0" }}>Due: {dueDate}</Text>
          <Text style={{ fontSize: "14px", color: "#6b7280", margin: "4px 0" }}>Bill to: {customerName}</Text>

          <Hr style={{ borderColor: "#e5e7eb", margin: "24px 0" }} />

          {lineItems.map((item, i) => (
            <Section key={i} style={{ marginBottom: "8px" }}>
              <Row>
                <Column style={{ width: "70%" }}>
                  <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "0" }}>{item.description}</Text>
                </Column>
                <Column style={{ width: "30%", textAlign: "right" as const }}>
                  <Text style={{ fontSize: "14px", color: "#4a4a4a", margin: "0" }}>${item.amount.toFixed(2)}</Text>
                </Column>
              </Row>
            </Section>
          ))}

          <Hr style={{ borderColor: "#e5e7eb", margin: "16px 0" }} />

          <Row>
            <Column style={{ width: "70%" }}>
              <Text style={{ fontSize: "16px", fontWeight: "bold", color: "#1a1a1a", margin: "0" }}>Total</Text>
            </Column>
            <Column style={{ width: "30%", textAlign: "right" as const }}>
              <Text style={{ fontSize: "16px", fontWeight: "bold", color: "#1a1a1a", margin: "0" }}>
                ${total.toFixed(2)}
              </Text>
            </Column>
          </Row>

          {paidAt && (
            <Text style={{ fontSize: "12px", color: "#16a34a", marginTop: "16px" }}>
              Paid on {paidAt}. Thank you for your payment.
            </Text>
          )}
        </Container>
      </Body>
    </Html>
  );
}
```

## Sending Any Template

```typescript
// Example: sending each template type
import { sendEmail } from "@/lib/email";
import WelcomeEmail from "@/emails/welcome";
import VerifyEmail from "@/emails/verify-email";
import PasswordReset from "@/emails/password-reset";
import OrderConfirmation from "@/emails/order-confirmation";
import AlertNotification from "@/emails/alert-notification";

// Welcome
await sendEmail({
  to: "user@example.com",
  subject: "Welcome!",
  react: WelcomeEmail({ userName: "Alex", dashboardUrl: "https://app.com/dashboard" }),
});

// Verify
await sendEmail({
  to: "user@example.com",
  subject: "Verify your email",
  react: VerifyEmail({ verificationUrl: "https://app.com/verify?token=abc", expiresInHours: 24 }),
});

// Password Reset
await sendEmail({
  to: "user@example.com",
  subject: "Reset your password",
  react: PasswordReset({ resetUrl: "https://app.com/reset?token=xyz", userName: "Alex" }),
});

// Order Confirmation
await sendEmail({
  to: "user@example.com",
  subject: "Order #1234 confirmed",
  react: OrderConfirmation({
    orderId: "1234",
    customerName: "Alex",
    items: [{ name: "Pro Plan", quantity: 1, price: 49 }],
    total: 49,
  }),
});

// Alert
await sendEmail({
  to: "user@example.com",
  subject: "Security Alert",
  react: AlertNotification({
    userName: "Alex",
    alertType: "warning",
    title: "New login detected",
    message: "A new login was detected from Melbourne, Australia.",
    actionUrl: "https://app.com/security",
    actionLabel: "Review Activity",
  }),
});
```
