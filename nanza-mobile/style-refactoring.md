# Style Refactoring Specification

## Overview
This specification defines the standards for refactoring React Native components to remove inline styles and hardcoded values in favor of theme-based stylesheet classes and theme tokens.

## Core Principles

1. **No inline style objects** - All static styles must be in stylesheet classes
2. **No hardcoded colors** - All colors must use theme color tokens
3. **No hardcoded sizes** - Icon sizes, spacing, and dimensions must use theme tokens
4. **Theme variables only for colors and component props** - Spacing and layout values should be in stylesheet classes, not inline theme variables
5. **Dynamic styles remain inline** - Only truly dynamic values (e.g., Animated.View transforms, conditional logic) can stay inline

## Style Organization

Styles should be organized into existing style buckets in `src/styles/components/`:

- **`layout.ts`** - Layout, positioning, margins, padding, flex properties
- **`typography.ts`** - Text styles, font sizes, text alignment, line heights
- **`buttons.ts`** - Button styles and button-related layouts
- **`cards.ts`** - Card component styles
- **`inputs.ts`** - Form input styles
- **`badges.ts`** - Badge and tag styles
- **`entity.ts`** - Entity-specific styles
- **`panels.ts`** - Panel and container styles
- **`cart.ts`** - Cart-specific styles

## Refactoring Rules

### 1. Inline Style Objects → Stylesheet Classes

**❌ Before:**
```tsx
<View style={{ marginTop: 6, paddingHorizontal: 20 }}>
  <Text style={{ fontSize: 14, textAlign: 'left' }}>
```

**✅ After:**
```tsx
<View style={[layoutStyles.marginTopXs, layoutStyles.paddingHorizontalLg]}>
  <Text style={typographyStyles.textSmLeft}>
```

**Action Items:**
- Extract all inline style objects to appropriate stylesheet files
- Create new style classes if they don't exist
- Use array syntax `[style1, style2]` when combining multiple styles
- Prefer combining styles in stylesheet rather than inline arrays when possible

**Detection Patterns:**
Watch for these common inline style patterns that should be converted to classes:
- **Conditional styles in arrays**: `condition && { ... }` → Create conditional classes
  ```tsx
  // ❌ Detect this pattern
  [baseStyle, condition && { paddingRight: 50 }, multiline && { height: 'auto' }]
  // ✅ Should be
  [baseStyle, condition && inputStyles.inputWithToggle, multiline && inputStyles.inputMultiline]
  ```
- **Positioned elements**: `style={{ position: 'absolute', right: X, top: Y }}` → Create positioned class
  ```tsx
  // ❌ Detect this pattern
  style={{ position: 'absolute', right: 15, top: 0, bottom: 0, justifyContent: 'center' }}
  // ✅ Should be
  style={inputStyles.toggleButton}
  ```
- **Wrapper/container styles**: `style={{ position: 'relative' }}` → Create wrapper class
  ```tsx
  // ❌ Detect this pattern
  <View style={{ position: 'relative' }}>
  // ✅ Should be
  <View style={inputStyles.inputWrapper}>
  ```
- **Multiline input styles**: `multiline && { height: 'auto', minHeight: X, paddingTop: Y }` → Create multiline class
- **Getter functions returning inline styles**: Functions that return style objects should return class references
  ```tsx
  // ❌ Detect this pattern
  const getStyle = () => {
    switch (variant) {
    case 'default':
      return { flexDirection: 'row', justifyContent: 'space-between' }
    default:
      return { marginBottom: 20 }
    }
  }
  // ✅ Should be
  const getStyle = () => {
    switch (variant) {
    case 'default':
      return inputStyles.toggleFieldSectionDefault
    default:
      return inputStyles.imageFieldSectionDefault
    }
  }
  ```
- **StyleSheet.create usage**: Replace with theme-based stylesheet classes
  ```tsx
  // ❌ Detect this pattern
  const styles = StyleSheet.create({
    container: { width: '100%' },
    textArea: { borderWidth: 1, borderRadius: 8 }
  })
  // ✅ Should be
  // In stylesheet file:
  textAreaContainer: { width: '100%' },
  textAreaInput: { borderWidth: 1, borderRadius: 8 }
  // In component:
  const inputStyles = getInputStyles(theme)
  <View style={inputStyles.textAreaContainer}>
  ```
- **Array.push with inline styles**: `containerStyle.push({ backgroundColor: 'transparent' })` → Use class
  ```tsx
  // ❌ Detect this pattern
  containerStyle.push({ backgroundColor: 'transparent' })
  // ✅ Should be
  containerStyle.push(badgeStyles.priceContainerTransparent)
  ```
- **Any style object with 2+ properties** should be a class, even if conditional
- **Style objects in array conditionals** should be extracted to classes
- **Conditional style objects in switch/default cases** should be classes
- **Getter functions that return inline style objects**: All getter functions should return class references, not inline objects
  ```tsx
  // ❌ Detect this pattern
  const getStyle = () => {
    switch (variant) {
    case 'default':
      return { flexDirection: 'row', justifyContent: 'space-between' }
    default:
      return { marginBottom: 20 }
    }
  }
  // ✅ Should be
  const getStyle = () => {
    switch (variant) {
    case 'default':
      return inputStyles.toggleFieldSectionDefault
    default:
      return inputStyles.imageFieldSectionDefault
    }
  }
  ```
- **Getter functions returning padding objects**: Functions that return padding style objects should use classes
  ```tsx
  // ❌ Detect this pattern
  const getPadding = () => {
    switch (size) {
    case 'sm': return { paddingVertical: 4, paddingHorizontal: 16 }
    case 'md': return { paddingVertical: 9, paddingHorizontal: 16 }
    default: return { paddingVertical: 18, paddingHorizontal: 24 }
    }
  }
  // ✅ Should be
  const getPadding = () => {
    switch (size) {
    case 'sm': return buttonStyles.buttonActionPaddingSm
    case 'md': return buttonStyles.buttonActionPaddingMd
    default: return buttonStyles.buttonActionPaddingXxl
    }
  }
  ```
- **Simple inline styles (marginRight, flex: 1)**: Even single-property inline styles should use classes
  ```tsx
  // ❌ Detect this pattern
  <View style={{ marginRight: 16 }}>{icon}</View>
  <View style={{ flex: 1 }}>
  // ✅ Should be
  <View style={buttonStyles.buttonCTAIconContainer}>{icon}</View>
  <View style={buttonStyles.buttonCTAContent}>
  ```
- **Inline text style objects with multiple properties**: Text styles with 2+ properties should be classes
  ```tsx
  // ❌ Detect this pattern
  <Text style={[
    typographyStyles.text,
    {
      fontSize: 15,
      fontWeight: theme.fonts.weight.semiBold,
      lineHeight: 28,
      color: theme.colors.textSecondary,
      fontFamily: theme.fonts.family.primary,
    }
  ]}>
  // ✅ Should be
  <Text style={buttonStyles.buttonCTATitle}>
  ```
- **LinearGradient inline styles**: All LinearGradient style props should use classes
  ```tsx
  // ❌ Detect this pattern
  <LinearGradient
    style={{
      position: 'absolute',
      left: 0,
      top: 0,
      bottom: 0,
      width: 14,
      pointerEvents: 'none',
    }}
  />
  // ✅ Should be
  <LinearGradient
    style={layoutStyles.headerGradientOverlayLeft}
  />
  ```
- **Animated.View inline styles**: Animated.View styles should use classes (except for truly dynamic runtime values like animated positions)
  ```tsx
  // ❌ Detect this pattern
  <Animated.View
    style={{
      position: 'absolute',
      top: 0,
      height: '100%',
      backgroundColor: theme.colors.surface,
      borderRadius: theme.radius.md,
      zIndex: -1,
    }}
  />
  // ✅ Should be (static properties in class, dynamic in inline)
  <Animated.View
    style={[
      tabStyles.tabAnimatedBackgroundPositioned,
      {
        left: activePosition,  // Dynamic runtime value - OK to keep inline
        width: activeWidth,     // Dynamic runtime value - OK to keep inline
      }
    ]}
  />
  ```
- **Hardcoded hex colors in component props**: Icon fill, backgroundColor, etc. should use theme colors
  ```tsx
  // ❌ Detect this pattern
  <Icon fill={state === 'increase' ? '#4CAF50' : '#D94437'} />
  // ✅ Should be
  <Icon fill={state === 'increase' 
    ? (theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent)
    : (theme.colors.error || theme.colors.primary?.['500']Scale?.['700'] || theme.colors.border)
  } />
  ```
- **Conditional inline styles in style arrays**: Conditional styles in arrays should use conditional classes
  ```tsx
  // ❌ Detect this pattern
  style={[
    baseStyle,
    fullWidth && { flex: 1 }
  ]}
  // ✅ Should be
  style={[
    baseStyle,
    fullWidth && tabStyles.tabItemFullWidth
  ]}
  ```

### 2. Hardcoded Colors → Theme Colors

**❌ Before:**
```tsx
color: '#FFFFFF'
backgroundColor: '#262626'
borderColor: '#4F4F4F'
fill="#white"
colors={['#090909', '#262626', '#090909']}
```

**✅ After:**
```tsx
color: theme.colors.text
backgroundColor: theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border
borderColor: theme.colors.border
fill={theme.colors.text}
colors={[
  theme.colors.background,
  theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border,
  theme.colors.background
]}
```

**Color Mapping:**
- `#FFFFFF` → `theme.colors.text` (or appropriate text color)
- `#262626` → `theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border`
- `#4F4F4F` → `theme.colors.border` or `theme.colors.primary?.['500']Scale?.['700']`
- `#090909` → `theme.colors.background`
- `#202020` → `theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border`
- `'white'` → `theme.colors.text` (or appropriate theme color)
- `'transparent'` → Can remain as-is (it's a valid CSS value, not a color)

**Fallback Strategy:**
- Always provide theme-based fallbacks: `theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border`
- **Never use hardcoded hex values as fallbacks** - all fallbacks must be theme-based
- When using conditional colors with fallbacks, chain theme-based fallbacks only:
  ```tsx
  // ❌ Bad - hardcoded hex as fallback
  theme.colors.secondary['500'] || '#6056ED'
  
  // ❌ Bad - hardcoded hex as last fallback
  theme.colors.secondary['500'] || theme.colors.primary?.['500'] || '#6056ED'
  
  // ✅ Good - theme-based fallback chain only
  theme.colors.secondary['500'] || theme.colors.primary?.['500'] || theme.colors.primary?.['500']Scale?.['500'] || theme.colors.primary?.['500']Scale?.['600'] || theme.colors.textAccent || theme.colors.border
  ```
- Chain multiple theme color options to ensure a valid theme color is always used
- If a specific color is critical and might not exist in theme, use a semantically similar theme color as fallback

### 3. Component Props with Hardcoded Colors → Theme Colors

**❌ Before:**
```tsx
<Icon fill="#4CAF50" />
<Icon fill={state === 'increase' ? '#4CAF50' : '#D94437'} />
backgroundColor="#FFFFFF"
```

**✅ After:**
```tsx
<Icon fill={theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent} />
<Icon fill={state === 'increase' 
  ? (theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent)
  : (theme.colors.error || theme.colors.primary?.['500']Scale?.['700'] || theme.colors.border)
} />
backgroundColor={theme.colors.background}
```

**Rules:**
- All color props (fill, stroke, backgroundColor, etc.) must use theme colors
- Use theme-based fallback chains (no hardcoded hex values, even as last fallback)
- For conditional colors, both branches must use theme-based fallbacks

### 4. Hardcoded Sizes → Theme Tokens

**❌ Before:**
```tsx
<Icon size={16} />
<Icon size={18} />
marginRight: 5
paddingVertical: 10
fontSize: 14
```

**✅ After:**
```tsx
<Icon size={theme.icons.sm} />  // 16
<Icon size={theme.icons.sm} />  // 18 → use sm (16) or check if base (24) is more appropriate
style={layoutStyles.marginRightXs}  // 5
style={layoutStyles.paddingVerticalSm}  // 10
style={typographyStyles.textSmLeft}  // fontSize: 14
```

**Size Mapping:**
- Icon sizes: Use `theme.icons.sm` (16), `theme.icons.base` (24), `theme.icons.lg` (32), `theme.icons.xl` (48)
- Spacing values: Create stylesheet classes in `layout.ts` (e.g., `marginRightXs`, `paddingVerticalSm`, `marginRightMd`)
- Font sizes: Create typography classes in `typography.ts` (e.g., `textSmLeft` for fontSize 14)
- Simple layout properties: Even single properties like `flex: 1` or `marginRight: 16` should use classes (e.g., `layoutStyles.flexOne`, `buttonStyles.buttonCTAIconContainer`)

### 4. Theme Variables in Inline Styles → Stylesheet Classes

**❌ Before:**
```tsx
<View style={{ marginTop: theme.spacing.xs }}>
<View style={{ paddingVertical: theme.spacing.sm }}>
```

**✅ After:**
```tsx
<View style={layoutStyles.marginTopXs}>
<View style={layoutStyles.paddingVerticalSm}>
```

**Rule:** Theme spacing variables should only be used:
- In stylesheet definitions (not inline)
- As component props when absolutely necessary
- For colors (always use theme colors)

### 5. Conditional Styles → Conditional Classes

**❌ Before (Inline Conditional Styles):**
```tsx
<View style={[
  baseStyle,
  { width: fullWidth ? '100%' : 'auto' }
]}>
<View style={[
  baseStyle,
  { flex: fullWidth ? 1 : undefined }
]}>
```

**✅ After (Conditional Classes):**
```tsx
// In stylesheet file (e.g., buttons.ts)
buttonContainerFullWidth: {
  width: '100%',
},
buttonContainerAuto: {
  width: 'auto',
},
buttonContentFullWidth: {
  flex: 1,
},
buttonContentAuto: {
  flex: undefined,
},

// In component
<View style={[
  baseStyle,
  fullWidth ? buttonStyles.buttonContainerFullWidth : buttonStyles.buttonContainerAuto
]}>
<View style={[
  baseStyle,
  fullWidth ? buttonStyles.buttonContentFullWidth : buttonStyles.buttonContentAuto
]}>
```

**Rule:** When styles vary based on props/conditions, create separate classes for each condition rather than using inline conditional style objects.

### 6. Dynamic Styles (Keep Inline)

**✅ Keep Inline (Acceptable):**
```tsx
// Animated transforms
<Animated.View style={{ transform: [{ translateX }] }} />

// Truly dynamic runtime values
<View style={{ width: dynamicWidth, height: dynamicHeight }} />

// Component props that are truly dynamic (computed at runtime)
paddingLeft: position === 'left' ? theme.spacing.gutter : 0
```

**Note:** Even dynamic styles should use theme values when possible:
```tsx
// ✅ Good - uses theme for the value
paddingLeft: position === 'left' ? theme.spacing.gutter : 0

// ❌ Bad - hardcoded value
paddingLeft: position === 'left' ? 11 : 0
```

**Distinction:**
- **Conditional styles** (based on props like `fullWidth`, `variant`, `size`) → Create conditional classes
- **Dynamic styles** (computed at runtime, animated values, user input) → Keep inline but use theme values

## Implementation Checklist

When refactoring a file, ensure:

- [ ] All inline style objects `style={{ ... }}` are moved to stylesheet classes
- [ ] All hardcoded colors (`#FFFFFF`, `'white'`, etc.) are replaced with theme colors
- [ ] All hardcoded icon sizes (`size={16}`, `size={18}`) use `theme.icons.*`
- [ ] All hardcoded spacing values are in stylesheet classes, not inline theme variables
- [ ] All `globalStyles.*` references are replaced with theme-based styles
- [ ] New styles are added to appropriate style buckets (layout, typography, buttons, etc.)
- [ ] Type assertions are used when needed (`as TextStyle['textAlign']`, `as ViewStyle['alignSelf']`)
- [ ] No linter errors are introduced
- [ ] Dynamic styles (Animated.View transforms, conditional props) remain inline but use theme values
- [ ] Conditional styles (based on props) use conditional classes instead of inline conditionals
- [ ] All fallback colors use theme-based fallback chains (no hardcoded hex values)
- [ ] Conditional inline styles in arrays (`condition && { ... }`) are converted to conditional classes
- [ ] Positioned elements (`position: 'absolute'`, `position: 'relative'`) use positioned classes
- [ ] Wrapper/container styles with multiple properties are extracted to classes
- [ ] Getter functions return class references, not inline style objects (especially padding functions)
- [ ] Simple inline styles (single properties like `marginRight: 16`, `flex: 1`) use classes
- [ ] Inline text style objects with 2+ properties are converted to classes
- [ ] LinearGradient and Animated.View style props use classes (except for truly dynamic runtime values)
- [ ] Component props with hardcoded hex colors (Icon fill, backgroundColor, etc.) use theme colors
- [ ] All color props use theme-based fallback chains, even in conditional expressions

## Common Patterns

### Pattern 1: Text with fontSize and textAlign
```tsx
// ❌ Before
<Text style={{ fontSize: 14, textAlign: 'left' }}>

// ✅ After - Create in typography.ts
textSmLeft: {
  color: theme.colors.text,
  fontSize: 14,
  textAlign: 'left' as TextStyle['textAlign'],
  fontFamily: theme.fonts.family.primary,
}

// Usage
<Text style={typographyStyles.textSmLeft}>
```

### Pattern 2: Container with padding
```tsx
// ❌ Before
<View style={{ paddingHorizontal: 28, paddingVertical: 10 }}>

// ✅ After - Create in layout.ts
paddingHorizontalLg: {
  paddingHorizontal: 28,
},
paddingVerticalSm: {
  paddingVertical: 10,
}

// Usage
<View style={[layoutStyles.paddingHorizontalLg, layoutStyles.paddingVerticalSm]}>
```

### Pattern 3: Icon with size and color
```tsx
// ❌ Before
<Icon name="heart" size={16} fill="#FFFFFF" />

// ✅ After
<Icon name="heart" size={theme.icons.sm} fill={theme.colors.text} />
```

### Pattern 4: Action Menu Colors
```tsx
// ❌ Before
{
  color: '#262626',
  labelColor: '#FFFFFF',
}

// ✅ After
{
  color: theme.colors.well || theme.colors.primary?.['500']Scale?.['950'] || theme.colors.border,
  labelColor: theme.colors.text,
}
```

### Pattern 5: EntityTags customColors
```tsx
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

### Pattern 6: Conditional Styles with Props
```tsx
// ❌ Before
<View style={[
  baseStyle,
  { width: fullWidth ? '100%' : 'auto' },
  { flex: fullWidth ? 1 : undefined }
]}>

// ✅ After - Create conditional classes
// In stylesheet:
buttonContainerFullWidth: {
  width: '100%',
},
buttonContainerAuto: {
  width: 'auto',
},
buttonContentFullWidth: {
  flex: 1,
},
buttonContentAuto: {
  flex: undefined,
},

// In component:
<View style={[
  baseStyle,
  fullWidth ? buttonStyles.buttonContainerFullWidth : buttonStyles.buttonContainerAuto,
  fullWidth ? buttonStyles.buttonContentFullWidth : buttonStyles.buttonContentAuto
]}>
```

### Pattern 7: Conditional Styles in Arrays
```tsx
// ❌ Before - Conditional inline styles in array
const inputStyleClasses = [
  baseStyle,
  isFocused && { borderColor: '#FF0000' },
  multiline && {
    height: 'auto',
    minHeight: 100,
    paddingTop: 15,
  },
  showToggle && { paddingRight: 50 },
]

// ✅ After - Use conditional classes
// In stylesheet:
inputFocused: {
  borderColor: theme.colors.error,
},
inputMultiline: {
  height: 'auto',
  minHeight: 100,
  paddingTop: 15,
  paddingBottom: 15,
  fontSize: 14,
  lineHeight: 20,
  fontFamily: theme.fonts.family.primary,
},
inputWithToggle: {
  paddingRight: 50,
},

// In component:
const inputStyleClasses = [
  baseStyle,
  isFocused && inputStyles.inputFocused,
  multiline && inputStyles.inputMultiline,
  showToggle && inputStyles.inputWithToggle,
]
```

### Pattern 8: Positioned Elements
```tsx
// ❌ Before
<View style={{ position: 'relative' }}>
  <TouchableOpacity style={{
    position: 'absolute',
    right: 15,
    top: 0,
    bottom: 0,
    justifyContent: 'center',
    alignItems: 'center',
  }}>

// ✅ After - Create positioned classes
// In stylesheet:
inputWrapper: {
  position: 'relative',
},
toggleButton: {
  position: 'absolute',
  right: 15,
  top: 0,
  bottom: 0,
  justifyContent: 'center',
  alignItems: 'center',
},

// In component:
<View style={inputStyles.inputWrapper}>
  <TouchableOpacity style={inputStyles.toggleButton}>
```

### Pattern 9: Getter Functions Returning Padding Objects
```tsx
// ❌ Before - Getter function returning inline padding objects
const getActionPadding = () => {
  switch (size) {
  case 'sm': return { paddingVertical: 4, paddingHorizontal: 16 }
  case 'md': return { paddingVertical: 9, paddingHorizontal: 16 }
  case 'lg': return { paddingVertical: 11, paddingHorizontal: 16 }
  default: return { paddingVertical: 18, paddingHorizontal: 24 }
  }
}

// ✅ After - Getter function returning class references
// In stylesheet (buttons.ts):
buttonActionPaddingSm: {
  paddingVertical: 4,
  paddingHorizontal: 16,
},
buttonActionPaddingMd: {
  paddingVertical: 9,
  paddingHorizontal: 16,
},
buttonActionPaddingLg: {
  paddingVertical: 11,
  paddingHorizontal: 16,
},
buttonActionPaddingXxl: {
  paddingVertical: 18,
  paddingHorizontal: 24,
},

// In component:
const getActionPadding = () => {
  switch (size) {
  case 'sm': return buttonStyles.buttonActionPaddingSm
  case 'md': return buttonStyles.buttonActionPaddingMd
  case 'lg': return buttonStyles.buttonActionPaddingLg
  default: return buttonStyles.buttonActionPaddingXxl
  }
}
```

### Pattern 10: Simple Inline Styles (Single Properties)
```tsx
// ❌ Before - Even single properties should use classes
<View style={{ marginRight: 16 }}>{icon}</View>
<View style={{ flex: 1 }}>

// ✅ After - Use classes for all inline styles
// In stylesheet (buttons.ts):
buttonCTAIconContainer: {
  marginRight: 16,
},
buttonCTAContent: {
  flex: 1,
},

// In component:
<View style={buttonStyles.buttonCTAIconContainer}>{icon}</View>
<View style={buttonStyles.buttonCTAContent}>
```

### Pattern 11: Inline Text Style Objects
```tsx
// ❌ Before - Text styles with multiple properties inline
<Text style={[
  typographyStyles.text,
  {
    fontSize: 15,
    fontWeight: theme.fonts.weight.semiBold,
    lineHeight: 28,
    color: theme.colors.textSecondary,
    fontFamily: theme.fonts.family.primary,
  }
]}>{title}</Text>

// ✅ After - Extract to dedicated class
// In stylesheet (buttons.ts):
buttonCTATitle: {
  fontSize: 15,
  fontWeight: theme.fonts.weight.semiBold as TextStyle['fontWeight'],
  lineHeight: 28,
  color: theme.colors.textSecondary,
  fontFamily: theme.fonts.family.primary,
},

// In component:
<Text style={buttonStyles.buttonCTATitle}>{title}</Text>
```

### Pattern 12: LinearGradient and Animated.View Styles
```tsx
// ❌ Before - Inline styles for LinearGradient
<LinearGradient
  style={{
    position: 'absolute',
    left: 0,
    top: 0,
    bottom: 0,
    width: 14,
    pointerEvents: 'none',
  }}
/>

// ✅ After - Use positioned class
// In stylesheet (layout.ts):
headerGradientOverlayLeft: {
  position: 'absolute',
  left: 0,
  top: 0,
  bottom: 0,
  width: 14,
  pointerEvents: 'none',
},

// In component:
<LinearGradient style={layoutStyles.headerGradientOverlayLeft} />
```

### Pattern 13: Hardcoded Hex Colors in Component Props
```tsx
// ❌ Before - Hardcoded hex in Icon fill prop
<Icon fill={state === 'increase' ? '#4CAF50' : '#D94437'} />

// ✅ After - Theme-based fallback chain
<Icon fill={state === 'increase' 
  ? (theme.colors.primary?.['500']Scale?.['400'] || theme.colors.primary?.['500'] || theme.colors.textAccent)
  : (theme.colors.error || theme.colors.primary?.['500']Scale?.['700'] || theme.colors.border)
} />
```

### Pattern 14: Gradient Colors with Fallbacks
```tsx
// ❌ Before - hardcoded hex fallbacks
const gradientColors = [
  theme.colors.secondary['500'] || '#6056ED',
  theme.colors.secondary['400'] || '#8384F6'
]

// ❌ Still Bad - hardcoded hex as last fallback
const gradientColors = [
  theme.colors.secondary['500'] || theme.colors.primary?.['500'] || '#6056ED',
  theme.colors.secondary['400'] || theme.colors.primary?.['500'] || '#8384F6'
]

// ✅ After - Theme-based fallback chain only (no hardcoded hex)
const gradientColors = [
  theme.colors.secondary['500'] || theme.colors.primary?.['500'] || theme.colors.primary?.['500']Scale?.['500'] || theme.colors.primary?.['500']Scale?.['600'] || theme.colors.textAccent || theme.colors.border,
  theme.colors.secondary['400'] || theme.colors.primary?.['500']Scale?.['400'] || theme.colors.secondary['500'] || theme.colors.primary?.['500'] || theme.colors.primary?.['500']Scale?.['500'] || theme.colors.textAccent || theme.colors.border
]
```

## File Structure

When adding new styles, follow this structure:

1. **Add to type definition** (if creating new style):
```typescript
type LayoutStyles = {
  // ... existing styles
  marginRightXs: ViewStyle  // New style
}
```

2. **Add implementation**:
```typescript
export const getLayoutStyles = (theme: ThemeTokens): LayoutStyles => ({
  // ... existing styles
  marginRightXs: {
    marginRight: 5,
  },
})
```

3. **Use in component**:
```typescript
const layoutStyles = getLayoutStyles(theme)
// ...
<View style={layoutStyles.marginRightXs}>
```

## Exceptions

The following can remain inline:
- `Animated.View` transform properties (truly dynamic)
- Conditional style logic based on props/state (but still use theme values)
- Component-specific dynamic width/height from props
- Style spreads that override stylesheet classes: `style={[baseStyle, { width: propWidth }]}`

## Testing

After refactoring:
1. Verify visual appearance matches original
2. Check that theme changes are reflected
3. Ensure no TypeScript/linter errors
4. Test responsive behavior if applicable
