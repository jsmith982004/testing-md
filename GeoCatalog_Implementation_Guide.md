# Geo-Intelligence Catalog Implementation Guide (Detailed)

This document provides a deep technical breakdown of the product sorting and matching engine implemented in `productlisting.page.ts`. It explains the architecture, the specific code changes, and the rationale behind every design decision.

---

## 1. The Core Problem
The application needed to prioritize "Trending" and "Popular" products globally. Previously, sorting only worked within individual categories, and products with minor naming differences (like "Freedom Oil, 1L" vs "Freedom Oil 1L") would fail to match, causing the Geo-Catalog to look empty.

---

## 2. Robust Name Normalization (`cleanName`)
**What changed**: I implemented a regex-based cleaning function to normalize all product names before comparison.

**The Code:**
```typescript
const cleanName = (s: string) => (s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
```

**Why we did it**: Retailers often use different punctuation (commas, dots, spaces). Standard string comparison fails in these cases.
**How it works**:
1. Converts name to lowercase.
2. Removes all non-alphanumeric characters.
3. Example: `"Fortune Oil, 1L."` becomes `"fortuneoil1l"`, ensuring a perfect match with the Geo-Catalog key.

---

## 3. Global Priority Scoring (The 200,000 Bracket)
**What changed**: I replaced the old category-based sorting with a **Global Bracket System**.

**The Logic:**
- **Matches (Trending/Popular)**: Base Score = **200,000**
- **Standard Products**: Base Score = **100,000**

**Score Calculation Code:**
```typescript
let score = 200000; // Match Base
score += (10000 - catIndex * 100); // Category Rank (Water > Grocery)
score += (p.source === 'TRENDING_MATCH' ? 2000 : 1000); // Trend vs Popular Bonus
score += (p.count || 0); // Tie-breaker count
```

**Why we did it**: In the "All" tab, we needed matched products from *any* category to appear above *every* unmatched product. By using 200,000 as a base, even the lowest-ranked trending item (200,001) is mathematically guaranteed to stay above the highest-ranked standard item (110,000).

---

## 4. Recursive Folder Sorting Engine
**What changed**: I implemented a recursive tree-traversal logic to ensure sorting applies to all nested UI elements.

**How it works**:
1. **`performSort()`**: The entry point that identifies if we are looking at a flat list or a folder structure.
2. **`recursiveSort()`**: A function that drills down into `category.categories` and `subCategory.categories`.
3. **`sortProductArray()`**: The final engine that sorts the actual `products[]` using the `scoresMap`.

**The Sort Logic:**
```typescript
products.sort((a, b) => {
  const scoreA = scoresMap.get(cleanName(a.product_name)) || 100000;
  const scoreB = scoresMap.get(cleanName(b.product_name)) || 100000;
  return scoreB - scoreA; // Descending Priority
});
```

---

## 5. Data Flow: From Backend to UI
**The Workflow:**
1. **Fetch**: `this.catalogueService.getGeoCatalog(pincode, shopId)` retrieves the local trending data.
2. **Ingest**: The system loops through the response and fills a `Map<string, number>` where the key is the `cleanName` and the value is the calculated `Priority Score`.
3. **Apply**: The UI calls `performSort()`, which traverses the retailer's inventory and re-orders every array based on the scores found in the Map.
4. **Result**: The "All" tab and individual categories now show Trending items at the very top, followed by Popular items, then standard inventory.

---

## 6. Performance Optimization
- **O(1) Lookups**: By using a `Map` instead of nested loops for matching, the sorting remains fast even if a shop has 5,000+ products.
- **Single Pass**: The recursion ensures that we only traverse the catalog once to apply the new order.
- **Production Cleanliness**: All debug logs and temporary sorting files have been removed to ensure zero overhead in the production build.

---
**Implementation Date**: May 2026
**Primary File**: `src/app/pages/store/productlisting/productlisting.page.ts`

