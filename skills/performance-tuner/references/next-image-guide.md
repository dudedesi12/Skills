# next/image Complete Guide

## Basic Usage

```tsx
import Image from "next/image";

// Local image (imported — auto-sized, auto-blur placeholder)
import heroImage from "@/public/hero.jpg";
<Image src={heroImage} alt="Hero" priority />

// Remote image (must specify width/height)
<Image src="https://example.com/photo.jpg" alt="Photo" width={800} height={600} />
```

## Responsive Images

```tsx
// Full-width image that adapts to container
<div className="relative aspect-video w-full">
  <Image
    src="/banner.jpg"
    alt="Banner"
    fill
    sizes="100vw"
    className="object-cover rounded-lg"
  />
</div>

// Image that's full-width on mobile, half on tablet, third on desktop
<div className="relative aspect-square">
  <Image
    src="/card-image.jpg"
    alt="Card"
    fill
    sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
    className="object-cover"
  />
</div>
```

## The `sizes` Prop

**Always set `sizes` when using `fill`.** Without it, the browser downloads the largest image.

```tsx
// sizes tells the browser how wide the image will be at each breakpoint
sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
// Mobile: 100% of viewport width
// Tablet: 50% of viewport width
// Desktop: 33% of viewport width
```

## Priority Loading

```tsx
// Set priority on the largest image above the fold (improves LCP)
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority />

// Rules for priority:
// - Only ONE image per page should have priority
// - It should be the LCP element (largest visible content)
// - Hero images, banner images = priority
// - Below-fold images, thumbnails = NO priority
```

## Blur Placeholder

```tsx
// Option 1: Auto-generated for local images
import localImage from "@/public/photo.jpg";
<Image src={localImage} alt="Photo" placeholder="blur" />

// Option 2: Custom blur data URL for remote images
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
/>

// Option 3: Color placeholder
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="empty" // Default — no placeholder
/>
```

### Generate Blur Data URL

```typescript
// scripts/generate-blur.ts
import { getPlaiceholder } from "plaiceholder";
import fs from "fs";

async function generateBlurDataURL(imagePath: string) {
  const buffer = fs.readFileSync(imagePath);
  const { base64 } = await getPlaiceholder(buffer);
  return base64;
}
```

## Supabase Storage Images

```tsx
// next.config.ts — allow Supabase images
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "*.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
};
```

```tsx
// Using Supabase storage images with next/image
const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;

<Image
  src={`${supabaseUrl}/storage/v1/object/public/avatars/${user.avatar_path}`}
  alt={user.full_name}
  width={48}
  height={48}
  className="rounded-full"
/>
```

## Common Mistakes

### Missing sizes with fill

```tsx
// BAD: fill without sizes → downloads largest image
<Image src="/photo.jpg" alt="Photo" fill />

// GOOD: fill with sizes
<Image src="/photo.jpg" alt="Photo" fill sizes="(max-width: 768px) 100vw, 50vw" />
```

### Using priority on everything

```tsx
// BAD: Multiple priority images → defeats the purpose
<Image src="/hero.jpg" alt="Hero" priority />
<Image src="/card1.jpg" alt="Card" priority />
<Image src="/card2.jpg" alt="Card" priority />

// GOOD: Only the LCP image gets priority
<Image src="/hero.jpg" alt="Hero" priority />
<Image src="/card1.jpg" alt="Card" />
<Image src="/card2.jpg" alt="Card" />
```

### Not setting width/height

```tsx
// BAD: No dimensions → CLS (layout shift)
<img src="/photo.jpg" alt="Photo" />

// GOOD: Explicit dimensions prevent layout shift
<Image src="/photo.jpg" alt="Photo" width={800} height={600} />
```
