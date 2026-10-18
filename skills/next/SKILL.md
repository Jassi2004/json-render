---
name: next
description: Next.js renderer for json-render that turns JSON specs into full Next.js applications with routes, layouts, SSR, and metadata. Use when working with @json-render/next, building Next.js apps from JSON specs, or creating AI-generated multi-page applications.
---

# @json-render/next

Next.js renderer that converts JSON specs into full Next.js applications with routes, pages, layouts, metadata, and SSR support.

## Quick Start

```bash
npm install @json-render/core @json-render/react @json-render/next
```

### 1. Define Your Spec

```typescript
// lib/spec.ts
import type { NextAppSpec } from "@json-render/next";

export const spec: NextAppSpec = {
  metadata: {
    title: { default: "My App", template: "%s | My App" },
    description: "A json-render Next.js application",
  },
  layouts: {
    main: {
      root: "shell",
      elements: {
        shell: { type: "Container", props: {}, children: ["nav", "slot"] },
        nav: { type: "NavBar", props: { links: [
          { href: "/", label: "Home" },
          { href: "/about", label: "About" },
        ]}, children: [] },
        slot: { type: "Slot", props: {}, children: [] },
      },
    },
  },
  routes: {
    "/": {
      layout: "main",
      metadata: { title: "Home" },
      page: {
        root: "hero",
        elements: {
          hero: { type: "Card", props: { title: "Welcome" }, children: [] },
        },
      },
    },
    "/about": {
      layout: "main",
      metadata: { title: "About" },
      page: {
        root: "content",
        elements: {
          content: { type: "Card", props: { title: "About Us" }, children: [] },
        },
      },
    },
  },
};
```

### 2. Create the App

```typescript
// lib/app.ts
import { createNextApp } from "@json-render/next/server";
import { spec } from "./spec";

export const { getPageData, generateMetadata, generateStaticParams } = createNextApp({
  spec,
  loaders: {
    // Server-side data loaders (optional)
    loadPost: async ({ slug }) => {
      const post = await getPost(slug as string);
      return { post };
    },
  },
});
```

### 3. Wire Up Route Files

```tsx
// app/[[...slug]]/page.tsx
import { notFound } from "next/navigation";
import { getPageData, generateMetadata, generateStaticParams } from "@/lib/app";
import { SiteRenderer } from "./renderer";

export { generateMetadata, generateStaticParams };

export default async function Page({
  params,
}: {
  params: Promise<{ slug?: string[] }>;
}) {
  const data = await getPageData({ params });
  if (!data) notFound();
  return <SiteRenderer {...data} />;
}
```

The component registry and action handlers contain functions, so import them inside a Client Component instead of passing them from the server.

```tsx
// app/[[...slug]]/renderer.tsx
"use client";

import {
  NextAppProvider,
  PageRenderer,
  type PageRendererProps,
} from "@json-render/next";
import { registry, handlers } from "@/lib/registry";

export function SiteRenderer(props: PageRendererProps) {
  return (
    <NextAppProvider registry={registry} handlers={handlers}>
      <PageRenderer {...props} />
    </NextAppProvider>
  );
}
```

The examples assume `lib/registry.tsx` exports your `registry` and `handlers`. Register the component types used by the spec (`Container`, `NavBar`, and `Card`); `PageRenderer` supplies `Slot` automatically. If you do not have custom action handlers, omit `handlers` from the import and provider.

Keep the document shell in the root layout:

```tsx
// app/layout.tsx
import type { ReactNode } from "react";

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

## Key Concepts

### NextAppSpec

The top-level spec defines an entire Next.js application:

- **metadata**: Root-level SEO metadata (title template, description, OpenGraph)
- **layouts**: Reusable layout element trees (each must include a `Slot` component)
- **routes**: Route definitions keyed by URL pattern
- **state**: Global initial state shared across all routes

### Route Patterns

Routes use Next.js URL conventions:

- `"/"` -- home page
- `"/about"` -- static route
- `"/blog/[slug]"` -- dynamic segment
- `"/docs/[...path]"` -- catch-all segment
- `"/settings/[[...path]]"` -- optional catch-all segment

### Layouts

Layouts wrap page content. Every layout MUST include a `Slot` component where page content will be rendered. Layouts are defined once in `spec.layouts` and referenced by routes via the `layout` field.

### Built-in Components

- **Slot**: Placeholder in layouts where page content is rendered
- **Link**: Client-side navigation link (wraps `next/link`)

### Built-in Actions

- **setState**: Update state value. Params: `{ statePath, value }`
- **pushState**: Append to array. Params: `{ statePath, value, clearStatePath? }`
- **removeState**: Remove from array by index. Params: `{ statePath, index }`
- **navigate**: Client-side navigation. Params: `{ href }`

### Data Loaders

Server-side async functions that run in the Server Component before rendering. Results are merged into the page's initial state.

```typescript
createNextApp({
  spec,
  loaders: {
    loadPost: async ({ slug }) => {
      const post = await db.post.findUnique({ where: { slug } });
      return { post };
    },
  },
});
```

### SSR

Write an async Server Component that calls `getPageData({ params })`, handles a missing route with `notFound()`, and passes the returned data to a client renderer. `getPageData` matches the route, runs its server-side loader, and merges initial state. Next.js calls the separately exported `generateMetadata` and `generateStaticParams` helpers for SEO and static generation.

The client renderer imports the registry and handlers locally, then renders `PageRenderer` inside `NextAppProvider`. Only the resolved page data crosses the server/client boundary.

### Entry Points

- `@json-render/next` -- Client components (NextAppProvider, PageRenderer, Link)
- `@json-render/next/server` -- Server utilities (createNextApp, matchRoute, schema)

## API Reference

### Server Exports (`@json-render/next/server`)

- `createNextApp(options)` -- Create getPageData, generateMetadata, generateStaticParams
- `schema` -- Custom schema for Next.js apps (for AI catalog generation)
- `matchRoute(spec, pathname)` -- Match a URL to a route spec
- `resolveMetadata(spec, route)` -- Resolve metadata for a route
- `slugToPath(slug)` -- Convert catch-all slug array to pathname
- `collectStaticParams(spec)` -- Collect static params for all routes

### Client Exports (`@json-render/next`)

- `NextAppProvider` -- Context provider for registry and handlers
- `PageRenderer` -- Renders a page spec with optional layout
- `NextErrorBoundary` -- Error boundary component
- `NextLoading` -- Loading state component
- `NextNotFound` -- Not-found component
- `Link` -- Built-in navigation component (wraps next/link)
