# Exhaustive Implementation Log: Local Catalog Synchronization & Geo-Sorting

## 1. Full Process Flow: What triggers what, and when?

The entire process revolves around ensuring that when a shop is created or its profile is updated, the catalog is sorted according to local demand, and the correct local S3 URLs are served to the frontend.

**Sequence of Events:**
1. **Trigger Event:** An action occurs that requires the shop's catalog to be rebuilt.
   - **API Call:** `PUT http://localhost:1215/rms/apis/v1/shop/updateProfileJSON/:shopId`
   - **Internal Triggers:** 
     - When a new shop is created via `addShop` or `addShopData` (for Google/Market unverified shops).
     - When shop details are modified via `updateShopDetails`.
     - When shop status changes via `updateShopStatus`.
2. **Controller Layer:** The request reaches `src/apis/controllers/v1/Shop.js` -> `updateProfileJSON`.
3. **Service Layer Execution:** The controller delegates to `src/apis/services/v1/Shop.js` -> `updateShopProfile`.
4. **Geo-Data Fetching:** Inside `updateShopProfile`, it calls `fetchGeoCatalog(pincode, shopId)`. This sends an HTTP GET request to the `catalogue_mgmt_service` (`/cms/apis/v1/geo/resolvedCatalog`).
5. **Sorting:** The returned demand data is passed to `reSortCatalogWithGeo()`, which reorders the `catalog` array in-memory so popular local items appear first.
6. **S3 Upload:** The sorted catalog is converted to JSON and uploaded using `uploadProfileToS3` (which uses `config.media_s3` to determine if it goes to AWS or Local Storage).
7. **Database Sync (The Fix):** The system calls `ShopMetaDataService.addMetaData` to update the `store_meta_data` table in PostgreSQL with the new categories and the exact URL of the generated `profile.json`.
8. **Frontend Retrieval:** When the frontend asks for the shop details, `ShopMetaDataController.getShopMetaData` is called. It fetches the URL from `store_meta_data`. If the environment is set to local (`USE_LOCAL_S3=true`), it rewrites the AWS URL to `http://localhost:1215/local_s3/...` on the fly before sending the response.

---

## 2. Exact Code Changes by File

Below is the absolute granular detail of every file modified, the exact code written, and the purpose.

### 1. Environment Configurations
**File Path:** `d:\Sarvm\backend\retailer_service\.dev.env`
**File Path:** `d:\Sarvm\backend\retailer_service\.lcl.env`

**Changes Made:**
*   **Removed/Commented:** `# MEDIA_S3=https://s3.ap-south-1.amazonaws.com/dev.sarvm.com`
*   **Added:**
    ```env
    MEDIA_S3=http://localhost:1215/local_s3
    USE_LOCAL_S3=true
    ```
**Why:** To route S3 operations to the local backend route (`/local_s3`) instead of the actual AWS S3 bucket during development, preventing cache issues and saving bandwidth.

---

### 2. Configuration Loader
**File Path:** `d:\Sarvm\backend\retailer_service\src\config\index.js`

**Changes Made:**
Added the boolean resolution for the new environment variable.
*   **Added at Line 62:**
    ```javascript
    const USE_LOCAL_S3 = AccessEnv('USE_LOCAL_S3', false);
    ```
*   **Added inside `module.exports` object (Line 161):**
    ```javascript
    useLocalS3: USE_LOCAL_S3 === 'true' || USE_LOCAL_S3 === true
    ```
**Why:** So that the rest of the node application can check `config.useLocalS3` as a strict boolean rather than dealing with string parsing of environment variables.

---

### 3. Core Service Logic & 500 Error Fixes
**File Path:** `d:\Sarvm\backend\retailer_service\src\apis\services\v1\Shop.js`

**A. Fixing Missing Imports (Lines 20-23)**
The API was crashing (Code 500) because dependencies were missing or conflicting.
*   **Old Code:**
    ```javascript
    const ShopMetaDataService = require('./ShopMetaData');
    const ShopLocation = require('../../models/ShopLocation');
    ```
*   **New Code Implemented:**
    ```javascript
    const { shopData } = require('./Catalog');
    const ShopMetaData = require('../../models/ShopMetaData');
    const ShopMetaDataService = require('./ShopMetaData');
    const ShopLocation = require('../../models/ShopLocation');
    ```
**Why:** We explicitly required `ShopMetaDataService` to handle the database sync later in the file. We also restored the `shopData` formatter and the `ShopMetaData` model which are required elsewhere in the file.

**B. Integrating Geo-Sorting & Fixing Variable Scope (Lines 458-464)**
Inside the `updateShopProfile` function, we integrated the sorting logic. A previous buggy iteration tried to use an undefined `items` variable. We corrected it to use the `catalog` array.
*   **Code Implemented:**
    ```javascript
    } else {
      // If catalog exists in storage, just re-sort it with Geo Intelligence
      const geoData = await fetchGeoCatalog(shop.pincode, shopId);
      if (geoData) {
        reSortCatalogWithGeo(catalog, geoData);
      }
      addAllCategory(catalog);
    }
    ```
**Why:** This ensures that if a catalog is fetched from S3, it is immediately re-sorted based on local demand (`fetchGeoCatalog`) before being saved again. `reSortCatalogWithGeo` sorts the `catalog` array in-place.

**C. Synchronizing Metadata to Database (Lines 470-472)**
*   **Code Implemented:**
    ```javascript
    // Update metadata in Database to ensure frontend gets the latest URL
    const categories = catalog.filter((item) => item.name !== 'All').map((item) => item.name);
    await ShopMetaDataService.addMetaData(shopId, categories, `${jsonURL}/profile.json`);
    ```
**Why:** This is the crucial fix for the "frontend not updating" issue. After generating and sorting the new `profile.json`, this code extracts the categories and forces the Postgres database (`store_meta_data` table) to update its record with the fresh `jsonURL`.

---

### 4. Controller Dynamic URL Update
**File Path:** `d:\Sarvm\backend\retailer_service\src\apis\controllers\v1\Shop.js`

**Changes Made in `updateProfileJSON`:**
*   **Old Code:**
    ```javascript
    const url = "https://s3.ap-south-1.amazonaws.com/dev.sarvm.com/new_shops/..."
    ```
*   **New Code Implemented (Line 695):**
    ```javascript
    const url = `${config.media_s3}/new_shops/${shop.guid}/profile.json`;
    ```
**Why:** Removed hardcoded AWS endpoints. The controller now constructs the response URL dynamically based on whether it is running locally or in production.

---

### 5. Frontend Path Rewriting (The Interceptor)
**File Path:** `d:\Sarvm\backend\retailer_service\src\apis\services\v1\ShopMetaData.js`

**Changes Made in `getMetaData` (Lines 44-52):**
*   **Code Implemented:**
    ```javascript
    if (config.useLocalS3 && result && result.length > 0) {
      result.forEach((item) => {
        if (item.url) {
          const urlParts = item.url.split('/');
          const guid = urlParts[urlParts.length - 2];
          item.url = `${config.media_s3}/new_shops/${guid}/profile.json`;
        }
      });
    }
    ```
**Why:** Even if the Postgres database contains a hardcoded AWS S3 URL from a previous deployment, if the developer is running with `USE_LOCAL_S3=true`, this block intercepts the database response and rewrites the URL to point to `http://localhost:1215/local_s3/...` before returning it to the frontend.

---

### 6. S3 Utility Dynamic Resolution
**File Path:** `d:\Sarvm\backend\retailer_service\src\common\libs\JsonToS3\JsonToS3.js`

**Changes Made in `getJsonUrl` (Line 115):**
*   **Old Code:** (Hardcoded string fallback)
*   **New Code Implemented:**
    ```javascript
    const s3URL = config.media_s3;
    ```
**Why:** Ensures that when the backend attempts to construct the final upload URL for the `profile.json`, it respects the `MEDIA_S3` variable from the `.env` file.

