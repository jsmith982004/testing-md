# Geo-Intelligence Catalog Optimization Guide

This document provides a detailed technical overview of the product sorting and matching system implemented in the `hha_web` project.

## 1. Overview of the Sorting Logic
The goal of this implementation is to prioritize products from the **Geo-Catalog** (Trending and Popular lists) at the top of the retailer's catalog, while maintaining the correct category hierarchy.

## 2. Data Fetching & Ingestion
The process is handled by the `applyGeoIntelligence` method in `productlisting.page.ts`.

### Data Source
- **API Call**: `catalogueService.getGeoCatalog(pincode, shopId)`
- **Response Structure**: The system parses the `categories` array. Each category contains a `products` array.
- **Identification**: Products are identified by a `source` field:
  - `TRENDING_MATCH`: High-priority trending items.
  - `POPULAR_MATCH`: Medium-priority popular items.

## 3. Scoring & Ranking System
To ensure perfect sorting in the "All" tab and individual category tabs, we use a **Global Priority Scoring** system.

### Priority Brackets
| Product Type | Base Score | Range |
| :--- | :--- | :--- |
| **Matched (Trending/Popular)** | **200,000** | 200,000 - 215,000 |
| **Unmatched (Retailer Only)** | **100,000** | 100,000 - 110,000 |

### Score Calculation Formula
For each product, the final `geoScore` is calculated as follows:
`Final Score = Base (200k/100k) + CategoryPriority + TypeBonus + TrendingCount`

- **CategoryPriority**: Based on the Geo-Catalog order (e.g., Water & Beverages = 10,000, Grocery = 9,900).
- **TypeBonus**: Trending matches get an additional `+2000`, Popular matches get `+1000`.
- **TrendingCount**: The `count` property from the backend is added as a final tie-breaker.

## 4. Robust Name Matching
One of the key improvements is the name normalization engine. It ensures that punctuation or formatting differences don't break the matching logic.

```typescript
const cleanName = (s: string) => (s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
```
- **Action**: Removes commas, dots, spaces, and special characters.
- **Example**: `"Freedom Cooking Oil, 1L"` becomes `"freedomcookingoil1l"`, allowing for a 100% accurate match regardless of retailer naming conventions.

## 5. Implementation Workflow
1. **Normalization**: The frontend cleans the names of all Geo-Catalog items.
2. **Map Building**: A `scoresMap` is created to store the calculated priority for each matched product.
3. **Recursive Sorting**: The `performSort` method traverses the entire catalog tree (Folders -> Categories -> Subcategories) and applies the sorting to every product array found.
4. **UI Update**: Once sorting is complete, the `vendorDetails.dataList` is updated, triggering an immediate UI refresh.

## 6. Maintenance & Performance
- **Map-Based Lookup**: The use of `Map<string, number>` ensures $O(1)$ lookup time, making the sorting extremely fast even for large catalogs.
- **Private Methods**: All sorting logic is encapsulated in `private` methods to keep the class clean and prevent external interference.
- **No Backend Changes**: All logic is handled client-side, making it easy to adjust priority weights in the future without redeploying backend services.

---
**Date**: May 2026
**File**: `productlisting.page.ts`
