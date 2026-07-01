# Frontend Specification

## Scope
- Frontend-only guidance scoped to `src/`.
- This spec is the source of truth for future Cursor plans/prompts in this repo.
- Navigation is read-only reference; do not change it.
- For detailed styling rules, see `style-refactoring.md` in this folder.

## MANDATORY: Before Creating Any Plan

Complete this checklist BEFORE presenting solutions:

- [ ] **Confidence scoring added** - Every solution/decision has 0.0-1.0 score
- [ ] **Alternatives considered** - Listed other approaches with trade-offs
- [ ] **DRY check performed** - Searched for existing patterns/components
- [ ] **Surgical changes verified** - Minimized file count and code deltas
- [ ] **Escalation assessed** - Flagged non-trivial or risky changes
- [ ] **Examples/evidence provided** - Cited specific files/lines

If ANY checkbox is unchecked, the plan is incomplete.

## Engineering Standards and Approach

When working on any task in this codebase:
- **Ask clarifying questions** before starting implementation. Understand requirements fully.
- **Approach problems like a senior engineering team** at Meta or Shopify: prioritize scalability, maintainability, and production-ready code.
- **Break complex problems into smaller pieces**: decompose difficult tasks into manageable, reviewable chunks.
- **Peer review your solutions**: check your approach against what senior developers would recommend. Consider alternatives, edge cases, and long-term maintainability.
- **CRITICAL: Confidence Scoring (MANDATORY)** - Every solution, decision, and approach MUST have a confidence score (0.0 → 1.0):
  - **0.9-1.0**: High confidence - Well-tested approach, clear best practices, minimal unknowns, low risk
    - Example: *"Use existing getButtonStyles pattern - **Confidence: 0.95**"*
  - **0.7-0.8**: Good confidence - Sound approach with minor trade-offs, some uncertainties but manageable
    - Example: *"Add styles to filters.ts vs badges.ts - **Confidence: 0.8**"*
  - **0.5-0.6**: Moderate confidence - Multiple valid approaches, unclear winner, significant trade-offs (**requires listing alternatives**)
  - **0.0-0.4**: Low confidence - High uncertainty or risk (**MUST escalate or rethink approach**)
  - **Required: Score these elements** - Problem identification, overall solution approach, each major implementation decision, each alternative considered, individual implementation steps
  - **Minimum standard**: Any plan with <3 confidence scores is incomplete
- **Reflect and fix weak solutions**: any solution scoring below 0.8 should be reconsidered, refined, or escalated with clear reasoning about trade-offs.

## Non-Negotiables

- **Do not modify navigation**: no edits to `src/navigation/AppNavigator.tsx`, `src/navigation/BottomTabNavigator.tsx`, or screen registration order/params without escalation.
- **API endpoint changes must follow existing conventions**: update endpoints only via `src/services/api/*`, preserve `fetchData` behavior and established patterns.
- **Surgical changes only**: touch as few files as possible, prefer additive changes, minimize code deltas.
- **No inline StyleSheet.create**: all styles must go in style bucket files (`src/styles/components/*.ts`).
- **No hardcoded colors**: all colors must use `theme.colors.*` tokens.
- **No hardcoded sizes**: all spacing, icons, radius, fonts must use theme tokens.

## Significant Change Escalation (Required)

### When to Escalate (ASK BEFORE PROCEEDING)
- **Confidence <0.8** on any major decision
- Modifying navigation, API patterns, or shared utilities
- Touching >5 files for a single feature
- Changing behavior of existing components
- Any "Non-Negotiable" violation
- Deleting or renaming existing exports
- Changes to `src/contexts/*` or `src/styles/utils/*`

### Escalation Template (MANDATORY FORMAT)
If a change is non-trivial or risks behavior shifts, **ask before proceeding** using this format:
```
**ESCALATION REQUIRED**

Proposed change:
- Files: [list]
- Impact: [what changes]

Cause for concern:
- What could break: [specific risks]
- Affected areas: [screens/features]

Why this is the best option:
- [reasoning with Confidence: X.XX]

Alternatives considered:
1. [Option] - Confidence: X.XX - Why rejected: [reason]
2. [Option] - Confidence: X.XX - Why rejected: [reason]

Risk mitigation:
- [how to reduce risk]
```

## Project Layout Index (src/)

- `src/screens/`: Feature screens organized by domain
  - Each domain folder (e.g., `home/`, `auth/`, `product/`, `orders/`) contains screen components
  - Data fetching hooks live in `data/` subfolders (e.g., `src/screens/home/data/fetchBrands.ts`)
  - Examples: `HomeScreen`, `ProductScreen`, `OrderDetailScreen`, `LoginScreen`
- `src/components/`: Reusable UI components
  - `global/` - Cross-domain components (Button, Icon, Header, Badge, Form/*, Panel, Tabs, etc.)
  - `feed/` - Feed-related components (BrandFilterBar, FeedListingCard, FeedBidCard)
  - `entity/` - Entity display components (EntityCard, EntityTags)
  - `listings/` - Listing-specific components
  - `bids/` - Bid-specific components
  - `orders/` - Order-specific components
  - `modals/` - Modal components
  - `cart/` - Cart components
  - `search/` - Search components
  - `share/` - Share/social components
- `src/services/api/`: API service layer built on `fetchData`
  - Each resource has a service module: `Entity.ts`, `Listing.ts`, `Bid.ts`, `Brand.ts`, `Order.ts`, `Profile.ts`, `User.ts`, etc.
- `src/styles/`: Theme system and style buckets
  - `tokens/dark.json` - Theme token definitions (colors, spacing, fonts, icons, radius)
  - `utils/` - Theme utilities (`ThemeProvider.tsx`, `types.ts`, `resolveThemeColors.ts`)
  - `components/` - Style bucket files (see Styling Conventions below)
- `src/contexts/`: React contexts
  - `SessionContext.tsx` - Auth state, current user, brand selection
  - `CartContext.tsx` - Cart state management
  - `SavedItemsContext.tsx` - Saved items state
- `src/hooks/`: Shared custom hooks
- `src/navigation/`: Navigation configuration (read-only)
- `src/types/`: TypeScript types and DTOs
  - `index.ts` - DTO exports and payload types
  - `oak-api-schemas.ts` - Generated API schemas
  - `navigation.ts` - Navigation types
- `src/constants.ts`: Domain config constants (`STALE_TIME`, `brandTagConfig`)
- `src/lang/`: Internationalization files

## Navigation (Read-Only Reference)

- Navigation is defined in `src/navigation/AppNavigator.tsx` and `src/navigation/BottomTabNavigator.tsx`
- Uses React Navigation with `@react-navigation/stack` and `@react-navigation/bottom-tabs`
- Deep linking configured in `linking` object
- **Do not change screens, params, screen order, or navigation logic**
- If a new feature appears to require navigation changes, **escalate** before any changes
- Screen params accessed via `route.params` in screen components
- Navigation performed via `navigation.navigate('ScreenName', { params })`

## Data Access Conventions

### Service Layer (`src/services/api/*`)
- All API calls go through `fetchData` in `src/services/api/index.ts`
- Each resource has a service module with thin wrappers around `fetchData`
- Service methods accept URL parameters as strings (e.g., `?include=entity.brand`)
- **Endpoint updates are allowed only in services**, and must preserve:
  - `fetchData` usage
  - URL path conventions (leading `/resource/...`)
  - Existing auth/token behavior

**Service Pattern:**
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

### React Query Hooks (`src/screens/*/data/*`)
- Hooks are named `useFetchX` and live in screen-specific `data/` folders
- Standard pattern:
  - `useQuery` (or `useInfiniteQuery` for paginated data)
  - `queryKey` includes the key and identifying params
  - `queryFn` calls service methods with URL parameters
  - `enabled` guards on required params
  - `staleTime` from `STALE_TIME` constants in `src/constants.ts`
  - Returned object exposes `{ data, refetch, isLoading, error }` with feature-specific names
- Keep hooks read-only by default. Add mutations only when explicitly requested.

**Hook Pattern:**
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

### STALE_TIME Constants
```typescript
// src/constants.ts
export const STALE_TIME = {
  'LIVE': 0,           // Always refetch
  '5_MINS': 300000,
  '1_HOUR': 3600000,
  '6_HOURS': 21600000,
  '12_HOURS': 43200000,
  '1_DAY': 86400000,
}
```

## Type Conventions

- DTOs come from `src/types/index.ts` (generated schemas in `oak-api-schemas.ts`)
- Prefer DTO types (`EntityDto`, `ListingDto`, `BidDto`, `OrderDto`, etc.) for component and hook typing
- Extend payload types by mirroring existing `*Payload` patterns
- Use `components['schemas']['X']` for API schema types

**Type Pattern:**
```typescript
// src/types/index.ts
import type { components } from './oak-api-schemas'

export type EntityDto = components['schemas']['Entity']
export type ListingDto = components['schemas']['Listing']
export type BidDto = components['schemas']['Bid']

// Payload types for mutations
export type OrderPayload = {
  customerId?: string
  sellerId?: string
  // ...
}
```

## Styling Conventions

**IMPORTANT:** For comprehensive styling rules, see `documentation/style-refactoring.md`. This section provides a summary.

### Core Principles
1. **No inline style objects** - All static styles must be in style bucket files
2. **No hardcoded colors** - All colors must use `theme.colors.*` tokens
3. **No hardcoded sizes** - Icon sizes, spacing, dimensions must use theme tokens
4. **No `StyleSheet.create` in components** - Use style bucket functions instead
5. **Dynamic styles remain inline** - Only truly dynamic values (Animated transforms, runtime-computed values)

### Style Bucket Files (`src/styles/components/`)
Styles are organized into domain-specific files:

| File | Purpose |
|------|---------|
| `layout.ts` | Layout, positioning, margins, padding, flex properties |
| `typography.ts` | Text styles, font sizes, text alignment, line heights |
| `buttons.ts` | Button styles and button-related layouts |
| `cards.ts` | Card component styles |
| `inputs.ts` | Form input styles |
| `badges.ts` | Badge and tag styles |
| `entity.ts` | Entity-specific styles |
| `panels.ts` | Panel and container styles |
| `filters.ts` | Filter and brand toggle styles |
| `headers.ts` | Header component styles |
| `tabs.ts` | Tab component styles |
| `modals.ts` | Modal styles |
| `menus.ts` | Menu and popover styles |
| `cart.ts` | Cart-specific styles |
| `profile.ts` | Profile-related styles |
| `branding.ts` | Logo and branding styles |
| `alert.ts` | Alert component styles |

### Style Bucket Pattern
```typescript
// src/styles/components/filters.ts
import { ViewStyle, TextStyle, ImageStyle } from 'react-native'
import { ThemeTokens } from '../utils/types'

type FilterStyles = {
  brandToggleContainer: ViewStyle
  brandToggleItem: ViewStyle
  brandToggleItemSelected: ViewStyle
  brandToggleText: TextStyle
  // ... type definitions
}

export const getFilterStyles = (theme: ThemeTokens): FilterStyles => ({
  brandToggleContainer: {
    paddingHorizontal: theme.spacing.xs,
    paddingBottom: theme.spacing.lg,
  },
  brandToggleItem: {
    backgroundColor: theme.colors.neutral['700'],
    borderRadius: theme.radius.md,
    // ...
  },
  // ... implementations
})
```

### Using Styles in Components
```typescript
// In component file
import { useTheme } from '../../../styles/utils/ThemeProvider'
import { getFilterStyles } from '../../../styles/components/filters'

const MyComponent = () => {
  const { theme } = useTheme()
  const filterStyles = getFilterStyles(theme)
  
  return (
    <View style={filterStyles.brandToggleContainer}>
      {/* ... */}
    </View>
  )
}
```

### Theme Token Reference
```typescript
// Colors
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

// Spacing (numeric values)
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

// Border Radius
theme.radius.xxs     // 6
theme.radius.xs      // 9
theme.radius.sm      // 11
theme.radius.md      // 14
theme.radius.lg      // 21
theme.radius.xl      // 30
theme.radius.xxl     // 50
theme.radius.max     // 9999

// Icon Sizes
theme.icons.xxs      // 9
theme.icons.xs       // 12
theme.icons.sm       // 16
theme.icons.md       // 20
theme.icons.base     // 24
theme.icons.lg       // 32
theme.icons.xl       // 48

// Font Sizes
theme.fonts.size.xxs  // 11
theme.fonts.size.xs   // 12.5
theme.fonts.size.sm   // 13
theme.fonts.size.base // 14
theme.fonts.size.md   // 16
theme.fonts.size.lg   // 20
theme.fonts.size.xl   // 24

// Font Families
theme.fonts.family.primary    // 'EuclidCircularB'
theme.fonts.family.secondary  // 'Geist'
// Also available: 'Figtree-SemiBold', 'Figtree-Bold', etc.

// Font Weights
theme.fonts.weight.normal     // '400'
theme.fonts.weight.medium     // '500'
theme.fonts.weight.semiBold   // '600'
theme.fonts.weight.bold       // '700'
```

## Component Conventions

### File Structure
- Each component lives in its own folder: `ComponentName/index.tsx`
- Optional test file: `ComponentName/ComponentName.test.tsx`
- Related subcomponents can live in same folder

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
- Cross-domain, reusable UI primitives
- Import via direct path: `import Button from '../../../components/global/Button'`
- Examples: `Button`, `Icon`, `Header`, `Badge/*`, `Form/*`, `Panel`, `Tabs`, `LoadingIndicator`

### Domain Components
- Domain-specific components live in `src/components/{domain}/`
- Examples: `feed/BrandFilterBar`, `entity/EntityCard`, `listings/ListingCard`

## React Native Patterns

### Safe Area Handling
```typescript
import { useSafeAreaInsets } from 'react-native-safe-area-context'

const MyScreen = () => {
  const insets = useSafeAreaInsets()
  
  return (
    <View style={{ paddingTop: insets.top, paddingBottom: insets.bottom }}>
      {/* content */}
    </View>
  )
}
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

### Animated Values (OK to keep inline)
```typescript
// Dynamic animated values are acceptable inline
<Animated.View
  style={[
    staticStyles.container,
    {
      transform: [{ translateY: animatedValue }],
      opacity: fadeAnim,
    }
  ]}
/>
```

### TouchableOpacity
```typescript
<TouchableOpacity
  onPress={handlePress}
  activeOpacity={0.7}
  disabled={isDisabled}
>
  {/* content */}
</TouchableOpacity>
```

### Memoization
```typescript
// Use useMemo for expensive computations
const processedItems = useMemo(() => {
  return items.map(item => /* expensive transform */)
}, [items])

// Use useCallback for stable function references
const handlePress = useCallback(() => {
  // handler logic
}, [dependencies])
```

## React Best Practices

- **Avoid unnecessary complexity**: Keep components simple.
- **Minimize hook usage**: Avoid overusing `useMemo`, `useEffect`, `useRef`, and `useState` for simple layouts or styling logic.
- **Performance alignment**: Only optimize when necessary. Avoid premature optimization that adds complexity without measurable benefit.
- **Simplicity first**: If a solution requires multiple hooks, refs, or complex state management, consider if there's a simpler approach.
- **Memoize expensive operations**: Use `useMemo` for computed values, `useCallback` for handlers passed to child components.
- **Avoid anonymous functions in render**: Extract handlers to `useCallback` or component methods.

## Component Reusability & DRY Principles (MANDATORY)

### Strict Rules
- **Never duplicate JSX blocks >15 lines** across screens - extract to `src/components/`
- **2+ uses = extract immediately** - Don't wait for 3rd duplication
- **Layout patterns** (headers, panels, toolbars) MUST be components, not copy-paste
- **Before creating any feature screen**, check for existing patterns

### Component Extraction Checklist
Before adding JSX to a feature screen:
- [ ] Searched codebase for similar UI patterns
- [ ] Checked if >1 screen will need this pattern
- [ ] If reusable: extracted to `src/components/` with props
- [ ] If domain-specific: used appropriate folder (`entity/`, `listings/`, etc.)

### Folder Structure
- **Global** `src/components/global/ComponentName/` - Cross-domain components
- **Domain-specific** `src/components/{domain}/ComponentName/` - Single-domain components

### Code Review Requirements
Plans must include:
- **DRY Check Results** - What you searched for, what you found
- **Reusability Justification** - Why creating new vs reusing existing

## Images and Assets

- Image URLs commonly use environment-based base URLs
- Use fallback handling patterns for failed image loads
- SVG icons are in `assets/icons/` and mapped via `src/components/global/Icon/iconMap.ts`
- Fonts are in `assets/fonts/` and linked via `react-native.config.js`

## Environment Configuration

- Environment variables loaded via `@env` (react-native-dotenv)
- API base URL: `API_BASE_URL` from `@env`
- See `documentation/environment-switching.md` for environment details

## Surgical Change Checklist

- Smallest possible file set
- Prefer additive changes over edits
- Follow existing naming, hooks, and service patterns
- Avoid changing navigation or shared fetch behavior
- Add new styles to appropriate style bucket file
- Use theme tokens for all colors, spacing, sizes

## Required Plan Structure

Every plan MUST include these sections in this order:

### 1. Problem Analysis
- What needs to change and why
- **Confidence: X.XX** - How certain are you about the problem?
- Evidence: specific files, lines, patterns found in codebase

### 2. Proposed Solution
- High-level approach
- **Confidence: X.XX** - How certain this is the best approach?
- Why chosen over alternatives

### 3. Alternatives Considered
- Option 1: [description] - **Confidence: X.XX** - Why not chosen?
- Option 2: [description] - **Confidence: X.XX** - Why not chosen?
- (List at least 2 alternatives for any significant decision)

### 4. Implementation Steps
1. Step description - **Confidence: X.XX** - Risks: [list]
2. Step description - **Confidence: X.XX** - Risks: [list]
(Each step must have confidence score and identified risks)

### 5. Files Modified
- List all files to create or modify
- Mark **NEW** files vs edits
- Estimate lines added/changed per file

### 6. DRY Check Results
- **Searched for**: [patterns, similar implementations]
- **Found existing**: [components/patterns discovered]
- **Decision**: Reusing [what] OR Creating new because [why with confidence score]

### 7. Escalation Check
- **Escalation needed?** YES/NO
- If YES: Use escalation template from "Significant Change Escalation" section
- If NO: Explain why changes are low-risk

### 8. Overall Confidence Score
**X.XX** - Summary of solution confidence with key uncertainties listed

**Minimum Requirements:**
- At least 5 confidence scores throughout the plan
- DRY check must show actual search performed
- Alternatives section required for any confidence <0.9
- Escalation check cannot be skipped
