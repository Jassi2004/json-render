# @json-render/next

Next.js renderer for [@json-render/core](https://json-render.dev). JSON becomes full Next.js applications with routes, layouts, metadata, and SSR.

## Installation

```bash
npm install @json-render/core @json-render/react @json-render/next
```

## Quick Start

### 1. Define your application spec

```typescript
// lib/spec.ts
import type { NextAppSpec } from "@json-render/next";

export const spec: NextAppSpec = {
  metadata: {
    title: { default: "My App", template: "%s | My App" },
  },
  layouts: {
    main: {
      root: "shell",
      elements: {
        shell: { type: "Container", props: {}, children: ["nav", "slot"] },
        nav: { type: "NavBar", props: {}, children: [] },
        slot: { type: "Slot", props: {}, children: [] },
      },
    },
  },
  routes: {
    "/": {
      layout: "main",
      metadata: { title: "Home" },
      page: { root: "hero", elements: { hero: { type: "Card", props: { title: "Welcome" }, children: [] } } },
    },
    "/about": {
      layout: "main",
      metadata: { title: "About" },
      page: { root: "content", elements: { content: { type: "Card", props: { title: "About" }, children: [] } } },
    },
  },
};
```

### 2. Create the app

```typescript
// lib/app.ts
import { createNextApp } from "@json-render/next/server";
import { spec } from "./spec";

export const { getPageData, generateMetadata, generateStaticParams } =
  createNextApp({ spec });
```

### 3. Wire up Next.js routes

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

## Features

- **Pages as spec** -- Define entire multi-page apps in JSON
- **Route matching** -- Dynamic segments (`[slug]`), catch-all (`[...path]`), optional catch-all (`[[...path]]`)
- **Nested layouts** -- Reusable layouts with `Slot` component for content injection
- **SEO metadata** -- Per-route metadata with title templates, OpenGraph, Twitter cards
- **SSR** -- Server-side rendering via Next.js App Router
- **Data loaders** -- Server-side async data loading before page render
- **Static generation** -- `generateStaticParams` for pre-rendering
- **Client navigation** -- Built-in `Link` component wrapping `next/link`
- **Error/Loading/NotFound** -- Per-route error boundaries, loading states, and 404 pages
- **AI streaming** -- Generate entire apps with JSONL patches via `SpecStream`

## Entry Points

| Import                     | Description                                      |
| -------------------------- | ------------------------------------------------ |
| `@json-render/next`        | Client components (NextAppProvider, PageRenderer) |
| `@json-render/next/server` | Server utilities (createNextApp, matchRoute)      |

## Documentation

See the [json-render documentation](https://json-render.dev/docs/api/next) for full API reference.
