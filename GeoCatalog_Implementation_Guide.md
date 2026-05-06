# Geo-Intelligence Catalog Implementation Guide (Detailed)

This document provides a deep technical breakdown of the product sorting and matching engine implemented in `productlisting.page.ts`. It explains the architecture, the specific code changes, and the rationale behind every design decision.

---

## 1. The Core Problem
The application needed to prioritize "Trending" and "Popular" products globally. Previously, sorting only worked within individual categories, and products with minor naming differences (like "Freedom Oil, 1L" vs "Freedom Oil 1L") would fail to match, causing the Geo-Catalog to look empty.

---

## 2. Ingestion Logic (`applyGeoIntelligence`)
This function processes the backend response and prepares the priority map.

**Line-by-Line Explanation:**
```typescript
applyGeoIntelligence(geoRes: any) {
  // 1. Identify the categories array from the Geo-Catalog response
  const categories = geoRes?.catalog?.categories || [];
  
  // 2. Clear the scores map to ensure fresh data for each session/pincode
  this.scoresMap.clear();

  // 3. Iterate through each category in the Geo-Catalog
  categories.forEach((cat: any, catIndex: number) => {
    const products = cat.products || [];

    // 4. Loop through each product within that category
    products.forEach((p: any) => {
      // 5. Check if the product is a Trending or Popular match
      if (p.source === 'TRENDING_MATCH' || p.source === 'POPULAR_MATCH') {
        
        // 6. Start with a high global base score (200,000)
        let score = 200000;

        // 7. Add Category Priority: Higher index in Geo-Catalog gets a slightly lower boost
        score += (10000 - catIndex * 100);

        // 8. Add Source Bonus: Trending items get a higher boost than Popular items
        score += (p.source === 'TRENDING_MATCH' ? 2000 : 1000);

        // 9. Add Tie-breaker: Use the 'count' from backend to sort within the same tier
        score += (p.count || 0);

        // 10. Normalize the name and store the score in a high-speed Map
        const normalizedName = this.cleanName(p.product_name);
        this.scoresMap.set(normalizedName, score);
      }
    });
  });
  
  // 11. Trigger the recursive sorting engine
  this.performSort();
}
```

---

## 3. Name Normalization (`cleanName`)
**Code:**
```typescript
private cleanName(s: string): string {
  // 1. Return empty string if input is null/undefined to prevent crashes
  // 2. Convert to lowercase for case-insensitive matching
  // 3. Regex [^a-z0-9]: Strip ALL characters that are NOT letters or numbers
  return (s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
}
```
**Why**: This ensures that `"Freedom Oil, 1L"` matches `"freedomoil1l"`, bypassing retailer typos or formatting choices.

---

## 4. Scoring Engine (`getDetailedScore`)
This function is used during the sort comparison to find the priority of a specific product.

**Line-by-Line Explanation:**
```typescript
private getDetailedScore(productName: string) {
  // 1. Normalize the retailer's product name
  const normalized = this.cleanName(productName);
  
  // 2. Look up the score in our pre-calculated Map
  const score = this.scoresMap.get(normalized);
  
  // 3. If found, return the high-priority score
  if (score !== undefined) {
    return { score, matchName: normalized };
  }
  
  // 4. Fallback: Return a base score of 0 for products with no geo-match
  return { score: 0, matchName: '' };
}
```

---

## 5. Comparison Engine (`sortProductArray`)
This is the core logic used inside the JavaScript `.sort()` function.

**Line-by-Line Explanation:**
```typescript
private sortProductArray(products: any[]) {
  if (!products) return;

  products.sort((a, b) => {
    // 1. Get scores for both products being compared
    const resA = this.getDetailedScore(a.product_name || a.name);
    const resB = this.getDetailedScore(b.product_name || b.name);

    // 2. Use 100,000 as the base for unmatched items (Category sorting fallback)
    const scoreA = resA.score || 100000;
    const scoreB = resB.score || 100000;

    // 3. Return Difference: Sort in descending order (highest score at top)
    return scoreB - scoreA;
  });
}
```

---

## 6. Recursive Traversal (`recursiveSort`)
This ensures the logic reaches products nested deep within folders.

**Line-by-Line Explanation:**
```typescript
private recursiveSort(categories: any[]) {
  if (!categories) return;

  categories.forEach(cat => {
    // 1. If this category has a 'products' array, sort it
    if (cat.products && Array.isArray(cat.products)) {
      this.sortProductArray(cat.products);
    }
    
    // 2. If it has sub-categories, call this function again (Recursion)
    if (cat.categories && Array.isArray(cat.categories)) {
      this.recursiveSort(cat.categories);
    }
  });
}
```

---

## 7. Performance & Optimization
- **Map Lookup ($O(1)$)**: By storing scores in a `Map`, we avoid nested loops during sorting.
- **Recursive Depth**: The system handles infinite category nesting.
- **Cleanup**: All `console.log` and temporary files were removed to keep the main bundle light.

---
**Implementation Date**: May 2026
**Primary File**: `src/app/pages/store/productlisting/productlisting.page.ts`

