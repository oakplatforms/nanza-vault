# Debug Border Styles Reference

These debug borders were removed from `src/styles/components/cards.ts`.
To restore any of them, paste the 3-line block back into the corresponding style key.

```typescript
borderWidth: 1,
borderColor: theme.colors.system.error,
borderStyle: 'dotted',
```

## Affected style keys

### 1. `listingCardXxl` (was ~line 2198)
Insert after `backgroundColor: theme.colors.neutral['950'],`

### 2. `feedListingCardHeader` (was ~line 3396)
Insert at the start of the style object (first properties)

### 3. `feedListingCardHeaderLeft` (was ~line 3405)
Insert at the start of the style object (first properties)

### 4. `feedListingCardMoreButton` (was ~line 3426)
Insert at the start of the style object (first properties)

### 5. `feedListingCardContent` (was ~line 3437)
Insert at the start of the style object (first properties)

### 6. `feedListingCardActionIcons` (was ~line 3445)
Insert at the start of the style object (first properties)

### 7. `feedListingCardActionButton` (was ~line 3458)
Insert after `backgroundColor: theme.colors.neutral['950'],`

### 8. `feedListingCardLg` (was ~line 3519)
Insert after `borderRadius: 0,`
