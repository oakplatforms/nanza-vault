# Cart Toast Implementation Plan

## Overview
Create a "cart toast" — a duplicate of the cart preview bar that behaves like the existing toast system but appears inside modals (specifically the EntityModal) when items are added to the cart.

## Behavior
- Slides in from the bottom and fixes to `bottom: 0`
- Auto-dismisses after **7500ms** (existing toast is 2500ms + 5 seconds longer)
- Higher z-index than modals — displays over modal content
- When the EntityModal closes before the toast expires, the toast disappears (component unmounts naturally)
- Same visual style as `cartPreviewBar` except `bottom: 0` instead of `bottom: 48`

## Architecture
Follows the existing **listener-based singleton pattern** from `src/components/global/Toast/`.

## Files

| Action | File | What |
|--------|------|------|
| Create | `src/components/global/CartToast/index.ts` | Singleton API: `cartToast.show(message)` / `cartToast.hide()` |
| Create | `src/components/global/CartToast/CartToast.tsx` | Animated component — slides up from bottom, auto-dismiss at 7500ms |
| Add styles | `src/styles/components/cart.ts` | `cartToastBar` — clone of `cartPreviewBar` with `bottom: 0` |
| Modify | `src/components/modals/EntityModal/index.tsx` | Render `<CartToast />` inside the modal (above other content, below nested modals) |
| Modify | `src/components/feed/FeedListingCard/index.tsx` | Call `cartToast.show('Added to cart')` alongside existing `toast.show()` in `handleBuyNow` |

## Implementation Details

### 1. CartToast singleton (`src/components/global/CartToast/index.ts`)
Same shape as existing toast singleton:
```typescript
type CartToastConfig = { message: string }
type CartToastListener = (config: CartToastConfig) => void

export const cartToast = {
  setListener: (listener: CartToastListener | null) => { ... },
  show: (message: string) => { ... },
  hide: () => { ... },  // extra method to force-dismiss
}
```

### 2. CartToast component (`src/components/global/CartToast/CartToast.tsx`)
Mirrors `Toast.tsx` with these differences:
- **Slide direction**: Up from bottom (positive `translateY` → 0) instead of down from top
- **Auto-dismiss**: 7500ms instead of 2500ms
- **Slide distance**: ~60px upward
- **Styles**: Uses `cartToastBar` from cart styles instead of alert styles
- **Cleanup**: Clears listener on unmount (natural dismiss when EntityModal closes)

### 3. Cart toast styles (`src/styles/components/cart.ts`)
Clone `cartPreviewBar` with `bottom: 0`:
```typescript
cartToastBar: {
  // Same as cartPreviewBar but:
  bottom: 0,        // instead of 48
  zIndex: 9999,     // higher than modals
  // Keep: backgroundColor, alignItems, justifyContent, elevation, shadow, border, borderRadius
}
```

### 4. EntityModal integration
Render `<CartToast />` inside the `<Modal>` children, after the ScrollView content. Because it's inside the modal, it:
- Naturally appears above modal content (z-index)
- Naturally unmounts when modal closes (auto-dismiss on close)

### 5. FeedListingCard trigger
In `handleBuyNow` success path (line ~238):
```typescript
refreshCart()
setSelectedQuantity(1)
toast.show('Added to cart')
cartToast.show('Added to cart')  // Add this line
```

## Key Decisions

| Decision | Confidence | Alternative considered |
|----------|-----------|----------------------|
| Listener-based singleton (matches existing toast) | 0.95 | Context-based — rejected, adds unnecessary provider nesting |
| Render inside EntityModal (auto-cleanup) | 0.90 | Render at app root with manual dismiss on `closeEntityModal` — rejected, couples context to toast |
| 7500ms auto-dismiss | 0.85 | Configurable duration — not needed yet, easy to change later |
