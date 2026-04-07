# Supabase Auth Patterns with Next.js

## Complete Auth Setup

### Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # Server-only, NEVER expose
```

### Auth Callback Route

Every auth flow (OAuth, magic link, email confirmation) redirects back to this route.

```ts
// app/auth/callback/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const next = searchParams.get("next") ?? "/dashboard";

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  // If code exchange fails, redirect to login with error
  return NextResponse.redirect(`${origin}/login?error=Could+not+authenticate`);
}
```

### Auth Confirm Route (Email Verification)

```ts
// app/auth/confirm/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const token_hash = searchParams.get("token_hash");
  const type = searchParams.get("type");
  const next = searchParams.get("next") ?? "/dashboard";

  if (token_hash && type) {
    const supabase = await createClient();
    const { error } = await supabase.auth.verifyOtp({
      type: type as "signup" | "email",
      token_hash,
    });

    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  return NextResponse.redirect(`${origin}/login?error=Verification+failed`);
}
```

### Middleware for Session Management

```ts
// middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // IMPORTANT: Use getUser() not getSession()
  // getUser() validates the JWT with the server
  // getSession() only reads the cookie (can be tampered with)
  const { data: { user } } = await supabase.auth.getUser();

  // Protected routes
  const protectedPaths = ["/dashboard", "/settings", "/account", "/billing"];
  const isProtected = protectedPaths.some((p) =>
    request.nextUrl.pathname.startsWith(p)
  );

  if (!user && isProtected) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("redirect", request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  // Redirect authenticated users away from auth pages
  const authPaths = ["/login", "/signup"];
  const isAuthPage = authPaths.includes(request.nextUrl.pathname);

  if (user && isAuthPage) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return response;
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

## Complete Login Page

```tsx
// app/login/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";
import { GoogleLoginButton } from "./google-button";

export default function LoginPage({
  searchParams,
}: {
  searchParams: Promise<{ error?: string; redirect?: string }>;
}) {
  async function login(formData: FormData) {
    "use server";

    const email = formData.get("email") as string;
    const password = formData.get("password") as string;

    if (!email || !password) {
      redirect("/login?error=Email+and+password+required");
    }

    const supabase = await createClient();
    const { error } = await supabase.auth.signInWithPassword({ email, password });

    if (error) {
      redirect("/login?error=Invalid+email+or+password");
    }

    redirect("/dashboard");
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="w-full max-w-sm space-y-6 p-8">
        <h1 className="text-center text-2xl font-bold">Log In</h1>

        <GoogleLoginButton />

        <div className="relative">
          <div className="absolute inset-0 flex items-center">
            <div className="w-full border-t" />
          </div>
          <div className="relative flex justify-center text-sm">
            <span className="bg-white px-2 text-gray-500">or</span>
          </div>
        </div>

        <form action={login} className="space-y-4">
          <input
            name="email"
            type="email"
            required
            placeholder="Email"
            className="w-full rounded border p-3"
          />
          <input
            name="password"
            type="password"
            required
            placeholder="Password"
            className="w-full rounded border p-3"
          />
          <button
            type="submit"
            className="w-full rounded bg-blue-500 py-3 text-white hover:bg-blue-600"
          >
            Log In
          </button>
        </form>

        <p className="text-center text-sm text-gray-500">
          No account?{" "}
          <a href="/signup" className="text-blue-500 underline">
            Sign up
          </a>
        </p>
      </div>
    </div>
  );
}
```

```tsx
// app/login/google-button.tsx
"use client";

import { createClient } from "@/lib/supabase/client";

export function GoogleLoginButton() {
  async function handleClick() {
    const supabase = createClient();
    const { error } = await supabase.auth.signInWithOAuth({
      provider: "google",
      options: {
        redirectTo: `${window.location.origin}/auth/callback`,
      },
    });
    if (error) {
      alert("Failed to start Google login");
    }
  }

  return (
    <button
      onClick={handleClick}
      className="flex w-full items-center justify-center gap-2 rounded border px-4 py-3 hover:bg-gray-50"
    >
      <svg className="h-5 w-5" viewBox="0 0 24 24">
        <path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92a5.06 5.06 0 0 1-2.2 3.32v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.1z" />
        <path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" />
        <path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z" />
        <path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" />
      </svg>
      Continue with Google
    </button>
  );
}
```

## Password Reset Flow

### Request Reset

```tsx
// app/forgot-password/page.tsx
import { createClient } from "@/lib/supabase/server";

export default function ForgotPasswordPage() {
  async function requestReset(formData: FormData) {
    "use server";

    const email = formData.get("email") as string;
    if (!email) throw new Error("Email is required");

    const supabase = await createClient();
    const { error } = await supabase.auth.resetPasswordForEmail(email, {
      redirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/callback?next=/reset-password`,
    });

    if (error) {
      throw new Error("Failed to send reset email");
    }
  }

  return (
    <form action={requestReset} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-2xl font-bold">Reset Password</h1>
      <p className="text-gray-600">Enter your email to receive a reset link.</p>
      <input name="email" type="email" required placeholder="Email" className="w-full rounded border p-3" />
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">Send Reset Link</button>
    </form>
  );
}
```

### Update Password

```tsx
// app/reset-password/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export default function ResetPasswordPage() {
  async function updatePassword(formData: FormData) {
    "use server";

    const password = formData.get("password") as string;
    const confirm = formData.get("confirm") as string;

    if (!password || password.length < 8) {
      throw new Error("Password must be at least 8 characters");
    }

    if (password !== confirm) {
      throw new Error("Passwords do not match");
    }

    const supabase = await createClient();
    const { error } = await supabase.auth.updateUser({ password });

    if (error) {
      throw new Error("Failed to update password");
    }

    redirect("/dashboard");
  }

  return (
    <form action={updatePassword} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-2xl font-bold">Set New Password</h1>
      <input name="password" type="password" required minLength={8} placeholder="New password" className="w-full rounded border p-3" />
      <input name="confirm" type="password" required placeholder="Confirm password" className="w-full rounded border p-3" />
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">Update Password</button>
    </form>
  );
}
```

## Protecting Server Components

```tsx
// app/dashboard/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    redirect("/login");
  }

  const { data: profile } = await supabase
    .from("profiles")
    .select("*")
    .eq("id", user.id)
    .single();

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold">Welcome, {profile?.full_name ?? user.email}</h1>
    </div>
  );
}
```

## Auth State in Client Components

```tsx
// app/components/auth-status.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";
import type { User } from "@supabase/supabase-js";

export function AuthStatus() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const supabase = createClient();

    supabase.auth.getUser().then(({ data: { user } }) => {
      setUser(user);
      setLoading(false);
    });

    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (_event, session) => {
        setUser(session?.user ?? null);
      }
    );

    return () => subscription.unsubscribe();
  }, []);

  if (loading) return <span className="text-gray-400">Loading...</span>;

  if (!user) {
    return <a href="/login" className="text-blue-500">Log in</a>;
  }

  return <span>Logged in as {user.email}</span>;
}
```

## Logout Action

```ts
// app/actions/auth.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export async function logout() {
  const supabase = await createClient();
  await supabase.auth.signOut();
  redirect("/login");
}
```

```tsx
// Use in a Server Component
import { logout } from "@/app/actions/auth";

export function LogoutButton() {
  return (
    <form action={logout}>
      <button type="submit" className="text-red-500">
        Log Out
      </button>
    </form>
  );
}
```
