---
tags: [nanza-mobile, audit]
---

# knip report — 2026-11-01

Raw output of `npx knip@5 --reporter markdown` on `dev` at `4f710f4d` (tree had uncommitted messaging-improvements edits). False positives, verified by hand in [[dead-code]]: `buffer` (undeclared dep of react-native-svg), `polyfill.js` (loaded through `NODE_OPTIONS`), `scripts/figma-token-sync.js` (Figma Scripter script, see the figma-sync skill), the `@env` unlisted imports (react-native-dotenv alias) and `@react-native/babel-preset` (ships inside `@react-native/metro-config`).

# Knip report

## Unused files (6)

* amplify/backend/types/amplify-dependent-resources-ref.d.ts
* polyfill.js
* scripts/figma-token-sync.js
* src/components/entity/EntityCard/EntityCardBase.tsx
* src/components/entity/EntityCard/List.tsx
* src/components/modals/ActionModal/Entity.tsx

## Unused dependencies (1)

| Name   | Location          | Severity |
| :----- | :---------------- | :------- |
| buffer | package.json:28:6 | error    |

## Unlisted dependencies (11)

| Name                       | Location                                                 | Severity |
| :------------------------- | :------------------------------------------------------- | :------- |
| @env                       | src/components/global/AddressAutocomplete/index.tsx:6:40 | error    |
| @env                       | src/screens/account/AccountScreen/Wallet/index.tsx:11:69 | error    |
| @env                       | src/screens/activity/PaymentMethodScreen/index.tsx:5:69  | error    |
| @env                       | src/navigation/AppNavigator.tsx:6:33                     | error    |
| @env                       | src/services/analytics/index.ts:2:47                     | error    |
| @env                       | src/services/realtime/socket.ts:2:30                     | error    |
| @env                       | src/services/auth/cognito.ts:14:84                       | error    |
| @env                       | src/services/api/index.ts:2:31                           | error    |
| @env                       | src/utils/shareUrl.ts:1:33                               | error    |
| @env                       | src/utils/getImage.ts:1:50                               | error    |
| @react-native/babel-preset | babel.config.js                                          | error    |

## Unresolved imports (1)

| Name               | Location     | Severity |
| :----------------- | :----------- | :------- |
| $(pwd)/polyfill.js | package.json | error    |

## Unused exports (48)

| Name                        | Location                                                                            | Severity |
| :-------------------------- | :---------------------------------------------------------------------------------- | :------- |
| COLLECTION_PAGE_SIZE        | src/screens/collections/CollectionDetailScreen/data/fetchCollectionEntities.ts:7:14 | error    |
| NANZA_PAGE_SIZE             | src/screens/collections/CollectionDetailScreen/data/fetchNanzaSearch.ts:7:14        | error    |
| useFetchExistingEntityLists | src/components/modals/ListModal/data/fetchExistingEntityLists.ts:49:14              | error    |
| fetchAffiliateRail          | src/components/profile/AffiliateCircleCarousel/index.tsx:53:14                      | error    |
| railItemHasUnseen           | src/components/profile/AffiliateCircleCarousel/index.tsx:80:14                      | error    |
| useReplyToConnection        | src/screens/messages/MessagesScreen/data/mutations.ts:236:14                        | error    |
| default                     | src/components/orders/OrderTrackingModal/index.tsx:48:16                            | error    |
| default                     | src/components/global/Form/NumericField/index.tsx:155:16                            | error    |
| default                     | src/components/global/Form/ImageField/index.tsx:194:16                              | error    |
| QUICK_REACTIONS             | src/components/messages/MessageActionSheet.tsx:11:14                                | error    |
| SCAN_ADD_MAX_QUANTITY       | src/components/modals/ScanAddModal/index.tsx:17:14                                  | error    |
| AFFILIATE_LINK_HOST         | src/services/attribution/pendingAffiliate.ts:15:14                                  | error    |
| BRAND_TAG_VALUES_LIMIT      | src/screens/search/data/fetchBrandTags.ts:11:14                                     | error    |
| hasEmoji                    | src/components/global/EmojiText/index.tsx:51:14                                     | error    |
| LIST_ENTITY_INCLUDES        | src/screens/profile/data/fetchLists.ts:104:14                                       | error    |
| useListingQuantity          | src/contexts/ListingQuantityContext.tsx:12:14                                       | error    |
| buildListQueryParams        | src/screens/profile/data/fetchLists.ts:13:14                                        | error    |
| getListsQueryConfig         | src/screens/profile/data/fetchLists.ts:69:14                                        | error    |
| draftHasImage               | src/components/posts/composer/draft.ts:58:14                                        | error    |
| NumericField                | src/components/global/Form/index.tsx:4:21                                           | error    |
| ImageField                  | src/components/global/Form/index.tsx:7:21                                           | error    |
| CodeField                   | src/components/global/Form/index.tsx:9:21                                           | error    |
| TextArea                    | src/components/global/Form/index.tsx:8:21                                           | error    |
| isShareGuardActive          | src/contexts/ShareGuardContext.tsx:30:14                                            | error    |
| useSetPrimaryContentItem    | src/components/posts/usePosts.ts:254:14                                             | error    |
| getConditionLabels          | src/constants/conditionLabels.ts:16:14                                              | error    |
| getConditionPills           | src/constants/conditionLabels.ts:61:14                                              | error    |
| getEntityHeroBackdropStyle  | src/styles/components/posts.ts:174:14                                               | error    |
| ENTITY_HERO_CARD_ASPECT     | src/styles/components/posts.ts:184:14                                               | error    |
| OrderTrackingModal          | src/components/orders/index.ts:6:21                                                 | error    |
| getEntityTagLine            | src/utils/getEntityTagLine.ts:52:14                                                 | error    |
| ART_TOTAL                   | src/screens/auth/entrance.tsx:59:14                                                 | error    |
| PUBLIC_AUTHOR_PROFILE_TYPES | src/constants/profileTypes.ts:8:14                                                  | error    |
| TAG_LINE_SEPARATOR          | src/utils/getEntityTagLine.ts:5:14                                                  | error    |
| isPostSeenTracked           | src/hooks/useMarkPostSeen.ts:15:14                                                  | error    |
| parseShareReferenceCode     | src/utils/referenceCode.ts:93:14                                                    | error    |
| isPushAvailable             | src/services/push/index.ts:52:14                                                    | error    |
| formatFetchTimeUtc          | src/utils/dateHelpers.ts:41:14                                                      | error    |
| OFFER_PRICE_TOLERANCE       | src/utils/offerPrice.ts:18:14                                                       | error    |
| offerPriceBounds            | src/utils/offerPrice.ts:20:14                                                       | error    |
| SERVICE_FEE_FIXED           | src/utils/serviceFee.ts:9:14                                                        | error    |
| SERVICE_FEE_RATE            | src/utils/serviceFee.ts:8:14                                                        | error    |
| generateShippingOptionsRate | src/utils/shipping.ts:52:14                                                         | error    |
| pluralizeProtectiveSleeve   | src/utils/pluralize.ts:5:14                                                         | error    |
| pluralizeShippingShield     | src/utils/pluralize.ts:9:14                                                         | error    |
| getReturnShipment           | src/utils/shipping.ts:24:14                                                         | error    |
| pluralize                   | src/utils/pluralize.ts:1:14                                                         | error    |
| default                     | src/lang/i18n.js:21:16                                                              | error    |

## Unused exported types (83)

| Name                           | Location                                                                            | Severity |
| :----------------------------- | :---------------------------------------------------------------------------------- | :------- |
| CollectionSort                 | src/screens/collections/CollectionDetailScreen/data/fetchCollectionEntities.ts:9:13 | error    |
| EntityListData                 | src/components/modals/ListModal/data/fetchExistingEntityLists.ts:6:18               | error    |
| ProfileConnectState            | src/screens/messages/MessagesScreen/data/useProfileConnect.tsx:14:13                | error    |
| SupportedTagValueSearchOptions | src/screens/search/data/searchSupportedTagValues.ts:25:13                           | error    |
| SupportedTagValueFetchOptions  | src/screens/search/data/fetchSupportedTagValues.ts:16:13                            | error    |
| GroupMembershipState           | src/components/cards/GroupBannerThumb/index.tsx:24:13                               | error    |
| OfferLineLockupProps           | src/components/offers/OfferLineLockup/index.tsx:14:13                               | error    |
| OfferTermsRowProps             | src/components/offers/OfferTermsRow/index.tsx:11:13                                 | error    |
| OfferLineCardProps             | src/components/offers/OfferLineCard/index.tsx:9:13                                  | error    |
| BottomSheetRef                 | src/components/modals/BottomSheet/index.tsx:69:18                                   | error    |
| TabPill                        | src/components/global/TabPillsRow/index.tsx:16:18                                   | error    |
| StarRatingProps                | src/components/global/StarRating/index.tsx:11:13                                    | error    |
| OrdersRole                     | src/components/orders/OrdersList/index.tsx:14:13                                    | error    |
| BrandTagFacetOptions           | src/screens/search/data/fetchBrandTags.ts:13:13                                     | error    |
| SearchBarHandle                | src/components/search/SearchBar/index.tsx:35:13                                     | error    |
| EntityTabId                    | src/screens/entity/EntityScreen/index.tsx:60:13                                     | error    |
| SearchTab                      | src/screens/search/SearchScreen/index.tsx:54:13                                     | error    |
| JoinPromptRequest              | src/services/attribution/joinPrompt.ts:17:13                                        | error    |
| DraftTypeCounts                | src/components/posts/composer/draft.ts:79:13                                        | error    |
| PushPromptCopy                 | src/hooks/usePushPermissionMoment.tsx:18:13                                         | error    |
| TradesQuery                    | src/screens/trade/data/fetchTrades.ts:15:13                                         | error    |
| BadgeProps                     | src/components/global/Badge/index.tsx:42:15                                         | error    |
| ModalRef                       | src/components/modals/Modal/index.tsx:78:18                                         | error    |
| AuthScreenStyles               | src/styles/components/authScreens.ts:22:18                                          | error    |
| ResolvedGradient               | src/styles/components/gradients.ts:21:13                                            | error    |
| ScrollHost                     | src/contexts/ScrollHostContext.tsx:12:13                                            | error    |
| ModalStackEntry                | src/contexts/ModalStackContext.tsx:8:18                                             | error    |
| ReferenceRecordType            | src/hooks/useFetchByReference.ts:10:13                                              | error    |
| ReferenceResult                | src/hooks/useFetchByReference.ts:41:13                                              | error    |
| AnalyticsConfig                | src/services/analytics/config.ts:8:13                                               | error    |
| RealtimeFrame                  | src/services/realtime/socket.ts:25:13                                               | error    |
| EventProps                     | src/services/analytics/index.ts:22:26                                               | error    |
| EventName                      | src/services/analytics/index.ts:22:15                                               | error    |
| EmbedPlayerRelay               | src/components/posts/embed.ts:178:13                                                | error    |
| ConfirmResetPasswordInput      | src/services/auth/cognito.ts:285:13                                                 | error    |
| ResendSignUpCodeInput          | src/services/auth/cognito.ts:123:13                                                 | error    |
| UpdatePasswordInput            | src/services/auth/cognito.ts:306:13                                                 | error    |
| ConfirmSignUpInput             | src/services/auth/cognito.ts:104:13                                                 | error    |
| ResetPasswordInput             | src/services/auth/cognito.ts:269:13                                                 | error    |
| EntranceWindow                 | src/screens/auth/entrance.tsx:64:13                                                 | error    |
| SignInInput                    | src/services/auth/cognito.ts:141:13                                                 | error    |
| AuthSession                    | src/services/auth/cognito.ts:343:13                                                 | error    |
| ConnectionState                | src/hooks/useMessageUser.tsx:18:13                                                  | error    |
| EmbedPlatform                  | src/components/posts/embed.ts:5:13                                                  | error    |
| AgreementType                  | src/services/api/Agreement.ts:8:13                                                  | error    |
| SignUpInput                    | src/services/auth/cognito.ts:80:13                                                  | error    |
| CreateReportPayload            | src/services/api/Report.ts:29:13                                                    | error    |
| CustomerSnapshot               | src/utils/offerCheckout.ts:30:13                                                    | error    |
| ReferenceType                  | src/utils/referenceCode.ts:9:13                                                     | error    |
| RefundPayload                  | src/services/api/Refund.ts:3:13                                                     | error    |
| ImagePickerOptions             | src/utils/imagePicker.ts:37:18                                                      | error    |
| ImagePickerResult              | src/utils/imagePicker.ts:45:18                                                      | error    |
| ScanPingResponse               | src/services/api/Scan.ts:57:13                                                      | error    |
| SupportedGame                  | src/services/api/Scan.ts:69:13                                                      | error    |
| AddOnLike                      | src/utils/offerTotals.ts:50:13                                                      | error    |
| CameraPermissionResult         | src/utils/imagePicker.ts:7:13                                                       | error    |
| OfferTotalsInput               | src/utils/offerTotals.ts:7:13                                                       | error    |
| OrderServiceFee                | src/utils/serviceFee.ts:21:13                                                       | error    |
| OfferTotalsLine                | src/utils/offerTotals.ts:5:13                                                       | error    |
| OfferCopy                      | src/utils/offerCopy.ts:101:13                                                       | error    |
| QueryListResultType            | src/types/index.ts:178:13                                                           | error    |
| OrderDetails                   | src/types/index.ts:205:13                                                           | error    |
| ShipmentRate                   | src/types/index.ts:388:13                                                           | error    |
| PostCollectionReference        | src/types/index.ts:103:3                                                            | error    |
| PostEntityReference            | src/types/index.ts:102:3                                                            | error    |
| PostBulkReference              | src/types/index.ts:100:3                                                            | error    |
| PostBidReference               | src/types/index.ts:101:3                                                            | error    |
| MarketPricePoint               | src/types/index.ts:112:3                                                            | error    |
| ScanItemType                   | src/types/index.ts:76:13                                                            | error    |
| ConversationParticipantDto     | src/types/index.ts:60:3                                                             | error    |
| PostListingReference           | src/types/index.ts:99:3                                                             | error    |
| ShippingCarrier                | src/types/index.ts:47:3                                                             | error    |
| GroupListingDto                | src/types/index.ts:54:3                                                             | error    |
| TransactionDto                 | src/types/index.ts:35:3                                                             | error    |
| ContentItemDto                 | src/types/index.ts:50:3                                                             | error    |
| QueryListDto                   | src/types/index.ts:65:3                                                             | error    |
| PostLikeInfo                   | src/types/index.ts:98:3                                                             | error    |
| CategoryDto                    | src/types/index.ts:26:3                                                             | error    |
| ShipmentDto                    | src/types/index.ts:32:3                                                             | error    |
| GroupBidDto                    | src/types/index.ts:55:3                                                             | error    |
| ScanItemDto                    | src/types/index.ts:63:3                                                             | error    |
| ReviewDto                      | src/types/index.ts:51:3                                                             | error    |
| TagDto                         | src/types/index.ts:23:3                                                             | error    |

