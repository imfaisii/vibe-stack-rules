# Stack Rules — Supabase

> Covers: the three Supabase clients, the middleware/proxy session refresh, the auth gate, the 6 traps that will bite you, and query function shape.
>
> **Always keep the Supabase agent skills installed:** `npx skills add supabase/agent-skills`. They carry the latest SSR/RLS/schema patterns so this file doesn't have to.

---

## The three clients

`@supabase/ssr` gives you two factories. We wrap them in three files. **Never import `@supabase/ssr` or `@supabase/supabase-js` outside `lib/supabase/`.**

```ts
// lib/supabase/client.ts — for Client Components
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "@/types/database";

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!
  );
}
```

```ts
// lib/supabase/server.ts — for RSC, Server Actions, Route Handlers
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import type { Database } from "@/types/database";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Called from a Server Component — setAll will throw.
            // Safe to ignore IF the proxy/middleware is refreshing the session.
          }
        },
      },
    }
  );
}
```

```ts
// lib/supabase/admin.ts — service role, SERVER ONLY
import "server-only"; // crash the build if this is ever imported from a client component
import { createClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database";

export function createAdminClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    { auth: { persistSession: false, autoRefreshToken: false } }
  );
}
```

**Environment variable note:** Supabase is transitioning from `NEXT_PUBLIC_SUPABASE_ANON_KEY` to `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (new `sb_publishable_*` format). Both work during the transition. If the project is already using `_ANON_KEY`, keep it — don't do a gratuitous rename.

---

## The proxy/middleware — refreshes the session on every request

```ts
// lib/supabase/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";
import type { Database } from "@/types/database";

export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
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

  // IMPORTANT: getUser() MUST run here to refresh the JWT.
  // Do not remove this line even if you don't use `user` in middleware.
  await supabase.auth.getUser();

  return response;
}
```

```ts
// middleware.ts (or proxy.ts on Next 16+) at project root
import { updateSession } from "@/lib/supabase/middleware";
import type { NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    // Run on everything EXCEPT static assets & images
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

---

## Auth gate in the (app) layout

Route-group layout pattern — **one** place where auth is enforced for the whole app shell:

```tsx
// app/(app)/layout.tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export const dynamic = "force-dynamic"; // REQUIRED for any route reading session

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");
  return <>{children}</>;
}
```

---

## The 6 traps — read these before touching auth

1. **Never create a Supabase client in module scope.** On Vercel Fluid Compute (and any reused server process), a client created outside a function is shared across requests → one user's session leaks into another user's request. Always call `createClient()` **inside** the function that uses it.

2. **Never use ISR (`revalidate`) or `force-static` on authenticated routes.** The cached HTML includes the `Set-Cookie` header — serving it to another user logs them in as the first user. Use `dynamic = "force-dynamic"` instead.

3. **Never trust `supabase.auth.getSession()` on the server for authorization.** Use `supabase.auth.getUser()` — it hits the Supabase Auth server and verifies the JWT. `getSession()` reads cookies which can be spoofed.

4. **Never hand-write SQL in a component.** Queries live in `lib/db/queries/<domain>.ts` and take a client as their first argument. This is how we keep them testable and how RLS policies stay verifiable.

5. **Never import `lib/supabase/admin.ts` from anything under `app/` except Server Actions and Route Handlers.** The `import "server-only"` at the top will blow up the build if you try.

6. **RLS is on.** Always. If a query needs to bypass RLS it uses the admin client and you write a comment explaining why.

---

## Query function shape

```ts
// lib/db/queries/revenue.ts
import "server-only";
import type { SupabaseClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database";

type Client = SupabaseClient<Database>;

export async function getRevenueSummary(supabase: Client) {
  const { data, error } = await supabase
    .from("orders")
    .select("total, created_at")
    .gte("created_at", new Date(Date.now() - 30 * 864e5).toISOString());

  if (error) throw error; // let error.tsx catch it

  const total = data.reduce((acc, o) => acc + o.total, 0);
  return { total, deltaLabel: "Last 30 days" };
}
```

Every query function:
- takes `supabase` as the first arg (never creates its own)
- is marked `"server-only"`
- returns plain typed data
- throws on error (let the error boundary handle it)

---

## Database types

`types/database.ts` is **generated** from your Supabase schema. Never hand-edit it.

```bash
npx supabase gen types typescript --project-id <id> > src/types/database.ts
```

Add this as `pnpm db:types` in `package.json` and re-run it after every schema change. All query functions, mutations, and Server Actions derive their types from this file. It is the single source of truth for row shapes.
