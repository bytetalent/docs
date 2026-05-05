---
name: Marketing Layout Patterns
description: Spacing scale, hero heights, carousel dimensions, CTA labels, and nav conventions for marketing/landing pages.
category: design
applicable_phases: [code_gen, design_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## Spacing scale

Marketing sections use a fixed responsive spacing scale. Always include the responsive pair — never bare `py-20` without `md:py-24`:

| Section role   | Classes          |
| -------------- | ---------------- |
| Hero           | `py-20 md:py-24` |
| Primary body   | `py-16 md:py-20` |
| Secondary body | `py-12 md:py-16` |
| CTA band       | `py-16 md:py-20` |
| Footer         | `py-10 md:py-12` |

```tsx
// bad
<section className="py-20">

// good
<section className="py-20 md:py-24">
```

## Hero dimensions

- Single-column hero: `min-h-[320px]`, `max-w-[800px]`
- Two-column hero: `h-[380px]`, `max-w-[1200px]`

```tsx
// two-column hero wrapper
<section className="h-[380px] max-w-[1200px] mx-auto ...">
```

## Carousel

- Outer container has a **fixed height** (`h-[380px]`); never let height grow with content
- Inactive slides use `position: absolute` so they don't affect document flow
- Auto-advance default: **7000ms**

```tsx
// bad — variable height breaks layout shift
<div className="relative">
  {slides.map(...)}
</div>

// good — fixed-height outer, absolute inactive slides
<div className="relative h-[380px] overflow-hidden">
  {slides.map((slide, i) => (
    <div
      key={slide.id}
      className={cn(
        "absolute inset-0 transition-opacity",
        i === activeIndex ? "opacity-100" : "opacity-0 pointer-events-none"
      )}
    >
      {slide.content}
    </div>
  ))}
</div>
```

## CTA labels

Approved label set — do not invent variants:

| Variant           | Label              | Destination         |
| ----------------- | ------------------ | ------------------- |
| Primary (default) | `Get Started Free` | `ROUTES.SIGNUP`     |
| Primary (alt)     | `Get Early Access` | `ROUTES.SIGNUP`     |
| Secondary         | `View Pricing`     | pricing page/anchor |
| Secondary (sales) | `Contact Sales`    | contact page        |

Never use "Book a demo", "Request a demo", or free-form verb phrases unless a live demo booking flow exists.

At most **2 CTAs per hero**: primary (`default` variant) and secondary (`outline` variant). They must link to different destinations.

```tsx
// bad — 3 CTAs, invented label
<Button>Start Building</Button>
<Button variant="outline">Watch Demo</Button>
<Button variant="ghost">Learn More</Button>

// good — 2 CTAs, approved labels
<Button asChild>
  <Link href={ROUTES.SIGNUP}>Get Started Free</Link>
</Button>
<Button variant="outline" asChild>
  <Link href="#pricing">View Pricing</Link>
</Button>
```

## Navigation

- Nav is sticky, height `h-[72px]`
- The nav's primary CTA is always **"Get Started Free"** pointing to `ROUTES.SIGNUP`
- The sign-in link is always visible in the nav (never hidden behind a dropdown or hamburger only)

```tsx
// nav skeleton
<nav className="sticky top-0 z-50 h-[72px] ...">
  {/* logo */}
  {/* nav links */}
  <div className="flex items-center gap-3">
    <Link href={ROUTES.SIGNIN} className="...">
      Sign in
    </Link>
    <Button asChild>
      <Link href={ROUTES.SIGNUP}>Get Started Free</Link>
    </Button>
  </div>
</nav>
```
