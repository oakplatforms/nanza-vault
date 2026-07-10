# Frontend Reference Guide

Detailed codebase patterns, project structure, and token reference for `src/`.
Companion to the project root `CLAUDE.md` (rules). Consult this file when you need implementation details.

---

## Project Layout Index

- `src/screens/` — Feature screens organized by domain
  - Domain folders: `home/`, `auth/`, `product/`, `orders/`, etc.
  - Data fetching hooks: `src/screens/{domain}/data/` (e.g., `fetchBrands.ts`)
  - Examples: `HomeScreen`, `ProductScreen`, `OrderDetailScreen`, `LoginScreen`
- `src/components/` — Reusable UI components
  - `global/` — Cross-domain: Button, Icon, Header, Badge/*, Form/*, Panel, Tabs, LoadingIndicator
  - `feed/` — BrandFilterBar, FeedListingCard, FeedBidCard
  - `entity/` — EntityCard, EntityTags
  - `listings/` — Listing-specific components
  - `bids/` — Bid-specific components
  - `orders/` — Order-specific components
  - `modals/` — Modal components
  - `cart/` — Cart components
  - `search/` — Search components
  - `share/` — Share/social components
- `src/services/api/` — API service layer built on `fetchData`
  - Modules: `Entity.ts`, `Listing.ts`, `Bid.ts`, `Brand.ts`, `Order.ts`, `Profile.ts`, `User.ts`
- `src/styles/` — Theme system and style buckets
  - `tokens/dark.json` — Theme token definitions
  - `utils/` — ThemeProvider.tsx, types.ts, resolveThemeColors.ts
  - `components/` — Style bucket files
- `src/contexts/` — React contexts (SessionContext, CartContext, SavedItemsContext)
- `src/hooks/` — Shared custom hooks
- `src/navigation/` — Navigation configuration (**read-only, never modify**)
- `src/types/` — TypeScript types and DTOs (index.ts, oak-api-schemas.ts, navigation.ts)
- `src/constants.ts` — Domain config (STALE_TIME, brandTagConfig)
- `src/lang/` — Internationalization files

---

## Navigation (Read-Only)

- Defined in `src/navigation/AppNavigator.tsx` and `src/navigation/BottomTabNavigator.tsx`
- Uses React Navigation (`@react-navigation/stack`, `@react-navigation/bottom-tabs`)
- Deep linking configured in `linking` object
- Screen params: `route.params`
- Navigate: `navigation.navigate('ScreenName', { params })`
- **Never change screens, params, screen order, or navigation logic. Escalate if needed.**

---

## Data Access Patterns

### Service Layer (`src/services/api/*`)

All API calls go through `fetchData` in `src/services/api/index.ts`. Each resource has a service module with thin wrappers.

```typescript
// src/services/api/Entity.ts
import { fetchData } from './index'

export const entityService = {
  async get(entityId: string, parameters = '') {
    return fetchData({ url: `/entity/${entityId}${parameters}` })
  },
  async list(parameters = '') {
    return fetchData({ url: `/entities${parameters}` })
  },
}
```

Endpoint updates must preserve: `fetchData` usage, URL path conventions (`/resource/...`), existing auth/token behavior.

### React Query Hooks (`src/screens/*/data/*`)

Named `useFetchX`, live in screen-specific `data/` folders.

```typescript
// src/screens/home/data/fetchBrands.ts
import { useQuery } from '@tanstack/react-query'
import { brandService } from '../../../services/api/Brand'
import { STALE_TIME } from '../../../constants'

export const useFetchBrands = () => {
  const { data, refetch, error, isLoading } = useQuery({
    queryKey: ['brands'],
    queryFn: () => brandService.list('?usePagination=false'),
    staleTime: STALE_TIME['LIVE'],
  })

  return { brands: data?.data || [], refetch, isLoading }
}
```

Hook conventions: `useQuery` or `useInfiniteQuery` for paginated data, `queryKey` includes key + identifying params, `enabled` guards on required params, `staleTime` from `STALE_TIME` constants, return object exposes `{ data, refetch, isLoading, error }` with feature-specific names. Keep hooks read-only by default.

### STALE_TIME Constants

```typescript
// src/constants.ts
export const STALE_TIME = {
  'LIVE': 0,
  '5_MINS': 300000,
  '1_HOUR': 3600000,
  '6_HOURS': 21600000,
  '12_HOURS': 43200000,
  '1_DAY': 86400000,
}
```

---

## Type Conventions

- DTOs from `src/types/index.ts` (generated schemas in `oak-api-schemas.ts`)
- Prefer DTO types (`EntityDto`, `ListingDto`, `BidDto`, `OrderDto`) for component and hook typing
- Extend payload types by mirroring existing `*Payload` patterns
- Use `components['schemas']['X']` for API schema types

```typescript
// src/types/index.ts
import type { components } from './oak-api-schemas'

export type EntityDto = components['schemas']['Entity']
export type ListingDto = components['schemas']['Listing']
export type BidDto = components['schemas']['Bid']

export type OrderPayload = {
  customerId?: string
  sellerId?: string
  // ...
}
```

---

## Theme Token Reference

### Colors
```typescript
theme.colors.text              // Primary text
theme.colors.textMuted         // Muted text
theme.colors.textSubtle        // Subtle text
theme.colors.background        // Background
theme.colors.surface           // Surface/card background
theme.colors.well              // Well/inset background
theme.colors.border            // Borders
theme.colors.primary['500']    // Primary color scale
theme.colors.secondary['700']  // Secondary color scale
theme.colors.neutral['700']    // Neutral color scale
theme.colors.error             // Error state
theme.colors.success           // Success state
```

### Spacing (numeric values)
```typescript
theme.spacing.none   // 0
theme.spacing.xxs    // 4
theme.spacing.xs     // 6
theme.spacing.sm     // 8
theme.spacing.md     // 14
theme.spacing.lg     // 24
theme.spacing.xl     // 40
theme.spacing.xxl    // 80
theme.spacing.gap    // 9
theme.spacing.gutter // 11
theme.spacing.base   // 20
```

### Border Radius
```typescript
theme.radius.xxs     // 6
theme.radius.xs      // 9
theme.radius.sm      // 11
theme.radius.md      // 14
theme.radius.lg      // 21
theme.radius.xl      // 30
theme.radius.xxl     // 50
theme.radius.max     // 9999
```

### Icon Sizes
```typescript
theme.icons.xxs      // 9
theme.icons.xs       // 12
theme.icons.sm       // 16
theme.icons.md       // 20
theme.icons.base     // 24
theme.icons.lg       // 32
theme.icons.xl       // 48
```

### Font Sizes
```typescript
theme.fonts.size.xxs  // 11
theme.fonts.size.xs   // 12.5
theme.fonts.size.sm   // 13
theme.fonts.size.base // 14
theme.fonts.size.md   // 16
theme.fonts.size.lg   // 20
theme.fonts.size.xl   // 24
```

### Font Families
```typescript
theme.fonts.family.primary       // 'Roobert' (single app font; weight via fontWeight)
theme.fonts.family.secondary     // 'Roobert'
// Roobert ships 400/500/600/700 only. Its "$" glyph renders correctly at every
// weight, so prices use the primary font like everything else (no separate price
// font, no PriceText requirement).
```

### Font Weights
```typescript
theme.fonts.weight.normal     // '400'
theme.fonts.weight.medium     // '500'
theme.fonts.weight.semiBold   // '600'
theme.fonts.weight.bold       // '700'
```

---

## Component Conventions

### File Structure
- Each component: `ComponentName/index.tsx`
- Optional test: `ComponentName/ComponentName.test.tsx`
- Related subcomponents in same folder

### Component Pattern
```typescript
// src/components/global/Button/index.tsx
import React from 'react'
import { TouchableOpacity, Text, View } from 'react-native'
import { useTheme } from '../../../styles/utils/ThemeProvider'
import { getButtonStyles } from '../../../styles/components/buttons'

type ButtonProps = {
  title: string
  onPress: () => void
  variant?: 'primary' | 'secondary' | 'muted'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}

const Button = ({ title, onPress, variant = 'primary', size = 'md', disabled }: ButtonProps) => {
  const { theme } = useTheme()
  const buttonStyles = getButtonStyles(theme)

  // ... implementation
}

export default Button
```

### Global Components (`src/components/global/`)
Cross-domain, reusable UI primitives. Import via direct path: `import Button from '../../../components/global/Button'`

### Domain Components
Domain-specific components in `src/components/{domain}/` (e.g., `feed/BrandFilterBar`, `entity/EntityCard`)

---

## Style Refactoring Pattern Catalog

Complete before/after reference for all style patterns. Rules are in `CLAUDE.md`; these are the implementation examples.

### Pattern 1: Inline Style Objects → Stylesheet Classes
```typescript
// ❌ Before
<View style={{ marginTop: 6, paddingHorizontal: 20 }}>
<Text style={{ fontSize: 14, textAlign: 'left' }}>

// ✅ After
<View style={[layoutStyles.marginTopXs, layoutStyles.paddingHorizontalLg]}>
<Text style={typographyStyles.textSmLeft}>
```

### Pattern 2: Conditional Styles → Conditional Classes
```typescript
// ❌ Before
[baseStyle, condition && { paddingRight: 50 }, multiline && { height: 'auto' }]
[baseStyle, fullWidth && { flex: 1 }]

// ✅ After
[baseStyle, condition && inputStyles.inputWithToggle, multiline && inputStyles.inputMultiline]
[baseStyle, fullWidth && tabStyles.tabItemFullWidth]
```

### Pattern 3: Conditional Props → Separate Classes
```typescript
// ❌ Before
<View style={[baseStyle, { width: fullWidth ? '100%' : 'auto' }]}>

// ✅ After — create classes for both branches
// In stylesheet:
buttonContainerFullWidth: { width: '100%' },
buttonContainerAuto: { width: 'auto' },

// In component:
<View style={[baseStyle, fullWidth ? buttonStyles.buttonContainerFullWidth : buttonStyles.buttonContainerAuto]}>
```

### Pattern 4: Getter Functions → Class References
```typescript
// ❌ Before
const getPadding = () => {
  switch (size) {
    case 'sm': return { paddingVertical: 4, paddingHorizontal: 16 }
    case 'md': return { paddingVertical: 9, paddingHorizontal: 16 }
    default: return { paddingVertical: 18, paddingHorizontal: 24 }
  }
}

// ✅ After
const getPadding = () => {
  switch (size) {
    case 'sm': return buttonStyles.buttonActionPaddingSm
    case 'md': return buttonStyles.buttonActionPaddingMd
    default: return buttonStyles.buttonActionPaddingXxl
  }
}
```

### Pattern 5: Simple Single-Property Inline Styles → Classes
```typescript
// ❌ Before
<View style={{ marginRight: 16 }}>{icon}</View>
<View style={{ flex: 1 }}>

// ✅ After
<View style={buttonStyles.buttonCTAIconContainer}>{icon}</View>
<View style={buttonStyles.buttonCTAContent}>
```

### Pattern 6: Inline Text Styles → Classes
```typescript
// ❌ Before
<Text style={[typographyStyles.text, {
  fontSize: 15,
  fontWeight: theme.fonts.weight.semiBold,
  lineHeight: 28,
  color: theme.colors.textSecondary,
  fontFamily: theme.fonts.family.primary,
}]}>{title}</Text>

// ✅ After
<Text style={buttonStyles.buttonCTATitle}>{title}</Text>
```

### Pattern 7: Positioned Elements → Classes
```typescript
// ❌ Before
<View style={{ position: 'relative' }}>
  <TouchableOpacity style={{
    position: 'absolute', right: 15, top: 0, bottom: 0,
    justifyContent: 'center', alignItems: 'center',
  }}>

// ✅ After
<View style={inputStyles.inputWrapper}>
  <TouchableOpacity style={inputStyles.toggleButton}>
```

### Pattern 8: StyleSheet.create → Style Bucket Functions
```typescript
// ❌ Before
const styles = StyleSheet.create({
  container: { width: '100%' },
  textArea: { borderWidth: 1, borderRadius: 8 }
})

// ✅ After — in style bucket file:
export const getInputStyles = (theme: ThemeTokens) => ({
  textAreaContainer: { width: '100%' },
  textAreaInput: { borderWidth: 1, borderRadius: theme.radius.xxs },
})

// In component:
const inputStyles = getInputStyles(theme)
```

### Pattern 9: Array.push with Inline Styles → Classes
```typescript
// ❌ Before
containerStyle.push({ backgroundColor: 'transparent' })

// ✅ After
containerStyle.push(badgeStyles.priceContainerTransparent)
```

### Pattern 10: LinearGradient Styles → Classes
```typescript
// ❌ Before
<LinearGradient style={{
  position: 'absolute', left: 0, top: 0, bottom: 0,
  width: 14, pointerEvents: 'none',
}} />

// ✅ After
<LinearGradient style={layoutStyles.headerGradientOverlayLeft} />
```

### Pattern 11: Animated.View — Split Static/Dynamic
```typescript
// ❌ Before
<Animated.View style={{
  position: 'absolute', top: 0, height: '100%',
  backgroundColor: theme.colors.surface,
  borderRadius: theme.radius.md, zIndex: -1,
}} />

// ✅ After — static in class, dynamic inline
<Animated.View style={[
  tabStyles.tabAnimatedBackgroundPositioned,
  { left: activePosition, width: activeWidth }  // Dynamic runtime only
]} />
```

### Pattern 12: Hardcoded Hex Colors → Theme Colors
```typescript
// ❌ Before
<Icon fill="#4CAF50" />
<Icon fill={state === 'increase' ? '#4CAF50' : '#D94437'} />
color: '#FFFFFF'
backgroundColor: '#262626'

// ✅ After
<Icon fill={theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent} />
<Icon fill={state === 'increase'
  ? (theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent)
  : (theme.colors.error || theme.colors.primary?.['500']Scale?.['700'] || theme.colors.border)
} />
color: theme.colors.text
backgroundColor: theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border
```

### Pattern 13: Gradient Colors with Theme-Only Fallbacks
```typescript
// ❌ Before
const gradientColors = [
  theme.colors.secondary['500'] || '#6056ED',
  theme.colors.secondary['400'] || '#8384F6'
]

// ✅ After — no hex fallbacks ever
const gradientColors = [
  theme.colors.secondary['500'] || theme.colors.primary?.['500'] || theme.colors.textAccent || theme.colors.border,
  theme.colors.secondary['400'] || theme.colors.primary?.['500']Scale?.['400'] || theme.colors.secondary['500'] || theme.colors.textAccent || theme.colors.border
]
```

### Pattern 14: EntityTags customColors
```typescript
// ❌ Before
customColors={{
  backgroundColor: 'transparent',
  textColor: '#FFFFFF',
  borderColor: theme.colors.primary?.['500']Scale?.['700'] || '#4F4F4F',
}}

// ✅ After
customColors={{
  backgroundColor: 'transparent',
  textColor: theme.colors.text,
  borderColor: theme.colors.primary?.['500']Scale?.['700'] || theme.colors.border,
}}
```

### Pattern 15: Action Menu Colors
```typescript
// ❌ Before
{ color: '#262626', labelColor: '#FFFFFF' }

// ✅ After
{
  color: theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border,
  labelColor: theme.colors.text,
}
```

### Pattern 16: Theme Variables Inline → Stylesheet Classes
```typescript
// ❌ Before — theme tokens inline is still wrong
<View style={{ marginTop: theme.spacing.xs }}>
<View style={{ paddingVertical: theme.spacing.sm }}>

// ✅ After — theme tokens belong in stylesheet definitions only
<View style={layoutStyles.marginTopXs}>
<View style={layoutStyles.paddingVerticalSm}>
```

### Adding New Styles to a Bucket File

1. Add to type definition:
```typescript
type LayoutStyles = {
  // ... existing
  marginRightXs: ViewStyle  // New
}
```

2. Add implementation:
```typescript
export const getLayoutStyles = (theme: ThemeTokens): LayoutStyles => ({
  // ... existing
  marginRightXs: { marginRight: 5 },
})
```

3. Use in component:
```typescript
const layoutStyles = getLayoutStyles(theme)
<View style={layoutStyles.marginRightXs}>
```

### Style Exceptions (OK to keep inline)
- `Animated.View` transform properties (truly dynamic)
- Conditional style logic based on props/state (but still use theme values)
- Component-specific dynamic width/height from props
- Style spreads that override stylesheet classes: `style={[baseStyle, { width: propWidth }]}`

---

## React Native Patterns

### Safe Area Handling
```typescript
import { useSafeAreaInsets } from 'react-native-safe-area-context'
const insets = useSafeAreaInsets()
// Use insets.top, insets.bottom for padding
```

### FlatList Performance
```typescript
<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
  removeClippedSubviews={true}
  maxToRenderPerBatch={5}
  windowSize={10}
  initialNumToRender={5}
  onEndReached={handleEndReached}
  onEndReachedThreshold={0.5}
/>
```

### Animated Values (inline is OK)
```typescript
<Animated.View style={[
  staticStyles.container,
  { transform: [{ translateY: animatedValue }], opacity: fadeAnim }
]} />
```

---

## Images and Assets

- Image URLs use environment-based base URLs
- Fallback handling for failed loads
- SVG icons: `assets/icons/` → mapped via `src/components/global/Icon/iconMap.ts`
- Fonts: `assets/fonts/` → linked via `react-native.config.js`

## Environment Configuration

- Variables via `@env` (react-native-dotenv)
- API base URL: `API_BASE_URL` from `@env`
- Environment values live in the repo `.env` files (not committed); switch by pointing `@env` at the target environment.

---

## Plan Structure Reference

When presenting plans, use this structure:

1. **Problem Analysis** — what needs to change, why, evidence from codebase — Confidence: X.XX
2. **Proposed Solution** — high-level approach, why chosen — Confidence: X.XX
3. **Alternatives Considered** — at least 2 for any confidence < 0.9, each with score and rejection reason
4. **Implementation Steps** — each with confidence score and risks
5. **Files Modified** — list all files, mark NEW vs edit, estimate lines
6. **DRY Check Results** — what you searched for, what you found, reuse vs create decision
7. **Escalation Check** — YES/NO with reasoning
8. **Overall Confidence Score** — X.XX with key uncertainties
