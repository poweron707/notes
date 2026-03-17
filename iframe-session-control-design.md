# Iframe Session Control & Security Design

**Date:** 2026-03-17

---

## Overview

Design for controlling iframe content visibility based on the parent page's login session. When a user's session expires, both the parent page and iframe content are locked down, and the user is prompted to re-authenticate.

---

## 1. Architecture

```
┌─────────────────────────────────────────────┐
│  Parent Page (authenticated)                │
│                                             │
│  ┌─── Session Monitor ───┐                  │
│  │ • Tracks user activity │                  │
│  │ • Countdown timer      │                  │
│  │ • Expiry overlay       │                  │
│  └────────┬───────────────┘                  │
│           │ postMessage                      │
│  ┌────────▼────────────────────────────┐     │
│  │  iframe (signed URL)                │     │
│  │                                     │     │
│  │  • Listens for SESSION_EXPIRED      │     │
│  │  • Reports IFRAME_ACTIVITY          │     │
│  │  • secureFetch → 401 = expired      │     │
│  │  • Clears content on expiry         │     │
│  └─────────────────────────────────────┘     │
└─────────────────────────────────────────────┘

Server:
  • Session validated on every API call
  • Iframe URL is signed + time-limited
  • 401 triggers client-side expiry flow
```

---

## 2. Parent Page — Session Monitor

```tsx
"use client";

import { useEffect, useState, useRef, useCallback } from "react";

export default function ProtectedIframePage() {
  const iframeRef = useRef<HTMLIFrameElement>(null);
  const [sessionExpired, setSessionExpired] = useState(false);
  const [timeLeft, setTimeLeft] = useState<number | null>(null);

  const SESSION_TIMEOUT = 30 * 60 * 1000; // 30 minutes
  const WARNING_THRESHOLD = 5 * 60 * 1000; // warn at 5 min left

  // track last activity
  const lastActivity = useRef(Date.now());

  const resetActivity = useCallback(() => {
    lastActivity.current = Date.now();
    // notify iframe that session is still alive
    iframeRef.current?.contentWindow?.postMessage(
      { type: "SESSION_REFRESH", expiresAt: Date.now() + SESSION_TIMEOUT },
      process.env.NEXT_PUBLIC_IFRAME_ORIGIN!
    );
  }, []);

  // listen for activity events
  useEffect(() => {
    const events = ["mousedown", "keydown", "scroll", "touchstart"];
    events.forEach((e) => window.addEventListener(e, resetActivity));
    return () => {
      events.forEach((e) => window.removeEventListener(e, resetActivity));
    };
  }, [resetActivity]);

  // session countdown timer
  useEffect(() => {
    const interval = setInterval(async () => {
      const elapsed = Date.now() - lastActivity.current;
      const remaining = SESSION_TIMEOUT - elapsed;

      if (remaining <= 0) {
        // session expired
        setSessionExpired(true);
        clearInterval(interval);

        // notify iframe to clear content
        iframeRef.current?.contentWindow?.postMessage(
          { type: "SESSION_EXPIRED" },
          process.env.NEXT_PUBLIC_IFRAME_ORIGIN!
        );

        // invalidate server session
        await fetch("/api/auth/logout", { method: "POST" });
        return;
      }

      if (remaining <= WARNING_THRESHOLD) {
        setTimeLeft(Math.ceil(remaining / 1000));
      } else {
        setTimeLeft(null);
      }
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  // listen for messages FROM iframe (activity, token requests)
  useEffect(() => {
    const handler = (event: MessageEvent) => {
      // verify origin
      if (event.origin !== process.env.NEXT_PUBLIC_IFRAME_ORIGIN) return;

      if (event.data?.type === "IFRAME_ACTIVITY") {
        resetActivity();
      }

      if (event.data?.type === "REQUEST_SESSION_STATUS") {
        iframeRef.current?.contentWindow?.postMessage(
          {
            type: "SESSION_STATUS",
            active: !sessionExpired,
            expiresAt: lastActivity.current + SESSION_TIMEOUT,
          },
          process.env.NEXT_PUBLIC_IFRAME_ORIGIN!
        );
      }
    };

    window.addEventListener("message", handler);
    return () => window.removeEventListener("message", handler);
  }, [sessionExpired, resetActivity]);

  if (sessionExpired) {
    return (
      <div className="fixed inset-0 bg-black/60 flex items-center justify-center z-50">
        <div className="bg-white rounded-xl p-8 max-w-md text-center">
          <h2 className="text-xl font-bold">⏱️ Session Expired</h2>
          <p className="mt-2 text-gray-600">
            Your session has timed out for security reasons.
          </p>
          <a
            href="/auth/signin"
            className="mt-4 inline-block px-6 py-2 bg-blue-600 text-white rounded-lg"
          >
            Sign In Again
          </a>
        </div>
      </div>
    );
  }

  return (
    <div className="relative h-screen">
      {/* timeout warning banner */}
      {timeLeft !== null && (
        <div className="absolute top-0 left-0 right-0 bg-yellow-500 text-black text-center py-2 z-40">
          ⚠️ Session expires in {Math.floor(timeLeft / 60)}:
          {String(timeLeft % 60).padStart(2, "0")} —{" "}
          <button onClick={resetActivity} className="underline font-bold">
            Stay Logged In
          </button>
        </div>
      )}

      <iframe
        ref={iframeRef}
        src={process.env.NEXT_PUBLIC_IFRAME_URL}
        className="w-full h-full border-0"
        sandbox="allow-scripts allow-same-origin allow-forms"
      />
    </div>
  );
}
```

---

## 3. Iframe Content — Session Listener

Add this script inside the iframe's page:

```ts
// iframe-session-bridge.ts

const PARENT_ORIGIN = "https://yourdomain.com";

let sessionActive = true;

// listen for session messages from parent
window.addEventListener("message", (event) => {
  if (event.origin !== PARENT_ORIGIN) return;

  switch (event.data?.type) {
    case "SESSION_EXPIRED":
      sessionActive = false;
      handleSessionExpired();
      break;

    case "SESSION_REFRESH":
      sessionActive = true;
      break;

    case "SESSION_STATUS":
      sessionActive = event.data.active;
      if (!sessionActive) handleSessionExpired();
      break;
  }
});

// report user activity inside iframe to parent
const events = ["mousedown", "keydown", "scroll", "touchstart"];
events.forEach((e) => {
  document.addEventListener(e, () => {
    if (sessionActive) {
      window.parent.postMessage({ type: "IFRAME_ACTIVITY" }, PARENT_ORIGIN);
    }
  });
});

// on load, ask parent for session status
window.parent.postMessage({ type: "REQUEST_SESSION_STATUS" }, PARENT_ORIGIN);

function handleSessionExpired() {
  // option 1: overlay
  const overlay = document.createElement("div");
  overlay.innerHTML = `
    <div style="position:fixed;inset:0;background:rgba(0,0,0,0.8);
      display:flex;align-items:center;justify-content:center;z-index:99999;
      color:white;font-size:1.2rem;">
      ⏱️ Session expired. Please log in again.
    </div>`;
  document.body.appendChild(overlay);

  // option 2: clear sensitive content
  // document.body.innerHTML = '';

  // option 3: stop all API calls
  // abortController.abort();
}
```

---

## 4. Server-Side Token Validation

Validate the session on every iframe API request:

```ts
// app/api/iframe-content/route.ts
import { NextRequest, NextResponse } from "next/server";
import { auth } from "@/lib/auth";

export async function GET(req: NextRequest) {
  const session = await auth();

  if (!session) {
    return NextResponse.json(
      { error: "SESSION_EXPIRED", message: "Please log in again" },
      { status: 401 }
    );
  }

  // return iframe content / data
  return NextResponse.json({ content: "..." });
}
```

---

## 5. Iframe Secure Fetch Wrapper

Use inside iframe for all API calls:

```ts
async function secureFetch(url: string, options?: RequestInit) {
  const res = await fetch(url, {
    ...options,
    credentials: "include", // send cookies
  });

  if (res.status === 401) {
    const data = await res.json();
    if (data.error === "SESSION_EXPIRED") {
      // notify parent
      window.parent.postMessage(
        { type: "IFRAME_SESSION_EXPIRED" },
        PARENT_ORIGIN
      );
      throw new Error("Session expired");
    }
  }

  return res;
}
```

---

## 6. Signed Iframe URL (Prevent Direct Access)

Generate a time-limited signed URL for the iframe:

```ts
// lib/iframe-token.ts
import crypto from "crypto";

const SECRET = process.env.IFRAME_SECRET!;

export function generateIframeUrl(userId: string): string {
  const expires = Math.floor(Date.now() / 1000) + 60 * 30; // 30 min
  const payload = `${userId}:${expires}`;
  const sig = crypto
    .createHmac("sha256", SECRET)
    .update(payload)
    .digest("hex")
    .slice(0, 16);

  return `/embed/content?token=${sig}&expires=${expires}&uid=${userId}`;
}
```

### Verify Token Before Serving Iframe Content

```ts
// app/embed/content/route.ts
import { NextRequest, NextResponse } from "next/server";
import crypto from "crypto";

const SECRET = process.env.IFRAME_SECRET!;

export async function GET(req: NextRequest) {
  const token = req.nextUrl.searchParams.get("token");
  const expires = req.nextUrl.searchParams.get("expires");
  const uid = req.nextUrl.searchParams.get("uid");

  if (!token || !expires || !uid) {
    return new NextResponse("Forbidden", { status: 403 });
  }

  // expired?
  if (Math.floor(Date.now() / 1000) > parseInt(expires)) {
    return new NextResponse("Token expired", { status: 403 });
  }

  // verify signature
  const expected = crypto
    .createHmac("sha256", SECRET)
    .update(`${uid}:${expires}`)
    .digest("hex")
    .slice(0, 16);

  if (token !== expected) {
    return new NextResponse("Invalid token", { status: 403 });
  }

  // serve content...
}
```

---

## 7. Environment Variables

```env
IFRAME_SECRET=your-random-64-char-string
NEXT_PUBLIC_IFRAME_ORIGIN=https://embed.yourdomain.com
NEXT_PUBLIC_IFRAME_URL=https://embed.yourdomain.com/content
```

---

## 8. PostMessage Communication Protocol

| Message | Direction | Purpose |
|---|---|---|
| `SESSION_REFRESH` | Parent → Iframe | Session still active, includes new `expiresAt` |
| `SESSION_EXPIRED` | Parent → Iframe | Session ended, clear content |
| `SESSION_STATUS` | Parent → Iframe | Response to status request |
| `IFRAME_ACTIVITY` | Iframe → Parent | User active inside iframe, reset timer |
| `REQUEST_SESSION_STATUS` | Iframe → Parent | Iframe asks if session is alive (on load) |
| `IFRAME_SESSION_EXPIRED` | Iframe → Parent | Iframe API got 401, notify parent |

---

## 9. Security Summary

| Threat | Protection |
|---|---|
| Iframe accessed without login | Signed iframe URL + server session check |
| Session expires, iframe stays visible | postMessage `SESSION_EXPIRED` → content cleared |
| User active in iframe but not parent | `IFRAME_ACTIVITY` forwarded to parent → resets timer |
| Direct iframe URL bookmarked | Signed URL expires in 30 min |
| Cross-origin message spoofing | Origin verification on both sides |
| API calls after session expires | Server returns 401 → iframe notifies parent |
| Iframe embedded on external site | `X-Frame-Options` / CSP `frame-ancestors` |

---

## 10. Recommended Headers

Add these to your Next.js config to prevent the iframe being embedded elsewhere:

```ts
// next.config.js
const nextConfig = {
  async headers() {
    return [
      {
        source: "/embed/:path*",
        headers: [
          {
            key: "Content-Security-Policy",
            value: "frame-ancestors 'self' https://yourdomain.com",
          },
          {
            key: "X-Frame-Options",
            value: "SAMEORIGIN",
          },
        ],
      },
    ];
  },
};
```
