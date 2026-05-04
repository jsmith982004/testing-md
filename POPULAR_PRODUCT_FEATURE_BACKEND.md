# Popular Product Feature – Backend Implementation

## Overview
This feature adds a "Popular Product" page to the admin panel. It fetches popular products from `geo_catalogs` collection, filters out those already in the `products` collection, and lets the admin either update the product name in geo_catalog or send a request to add a new product.

---

## BACKEND (catalogue_mgmt_service)

### File 1: NEW – `src/apis/routes/v1/popularProduct.js`

```js
const express = require('express');
const { Logger: log, HttpResponseHandler } = require('sarvm-utility');
const PopularProductController = require('../../controllers/v1/popularProduct');

const router = express.Router();

// GET /cms/apis/v1/popularProduct?page=1&pageSize=10&search=xyz
router.get('/', async (req, res, next) => {
  log.info({ info: 'PopularProduct route :: get unmatched popular products' });
  try {
    const { page = 1, pageSize = 10, search = '' } = req.query;
    const result = await PopularProductController.getUnmatchedPopularProducts(
      parseInt(page), parseInt(pageSize), search
    );
    HttpResponseHandler.success(req, res, result);
  } catch (error) {
    log.error({ error });
    next(error);
  }
});

// GET /cms/apis/v1/popularProduct/similar?name=xyz&page=1&pageSize=10
router.get('/similar', async (req, res, next) => {
  log.info({ info: 'PopularProduct route :: get similar products' });
  try {
    const { name, page = 1, pageSize = 10 } = req.query;
    const result = await PopularProductController.getSimilarProducts(
      name, parseInt(page), parseInt(pageSize)
    );
    HttpResponseHandler.success(req, res, result);
  } catch (error) {
    log.error({ error });
    next(error);
  }
});

// PUT /cms/apis/v1/popularProduct/update
router.put('/update', async (req, res, next) => {
  log.info({ info: 'PopularProduct route :: update product name in geo_catalog' });
  try {
    const { oldName, newName, pincode, city, state } = req.body;
    const result = await PopularProductController.updateProductInGeoCatalog(
      oldName, newName, pincode, city, state
    );
    HttpResponseHandler.success(req, res, result);
  } catch (error) {
    log.error({ error });
    next(error);
  }
});

// POST /cms/apis/v1/popularProduct/addRequest
router.post('/addRequest', async (req, res, next) => {
  log.info({ info: 'PopularProduct route :: send request to add new product' });
  try {
    const { productName, category } = req.body;
    const result = await PopularProductController.sendAddProductRequest(
      productName, category
    );
    HttpResponseHandler.success(req, res, result);
  } catch (error) {
    log.error({ error });
    next(error);
  }
});

module.exports = router;
```

---

### File 2: NEW – `src/apis/controllers/v1/popularProduct.js`

```js
const { Logger: log } = require('sarvm-utility');
const PopularProductService = require('../../services/v1/popularProduct.service');

const getUnmatchedPopularProducts = async (page, pageSize, search) => {
  log.info({ info: 'PopularProductController :: getUnmatchedPopularProducts' });
  return PopularProductService.getUnmatchedPopularProducts(page, pageSize, search);
};

const getSimilarProducts = async (name, page, pageSize) => {
  log.info({ info: 'PopularProductController :: getSimilarProducts' });
  return PopularProductService.getSimilarProducts(name, page, pageSize);
};

const updateProductInGeoCatalog = async (oldName, newName, pincode, city, state) => {
  log.info({ info: 'PopularProductController :: updateProductInGeoCatalog' });
  return PopularProductService.updateProductInGeoCatalog(oldName, newName, pincode, city, state);
};

const sendAddProductRequest = async (productName, category) => {
  log.info({ info: 'PopularProductController :: sendAddProductRequest' });
  return PopularProductService.sendAddProductRequest(productName, category);
};

module.exports = {
  getUnmatchedPopularProducts,
  getSimilarProducts,
  updateProductInGeoCatalog,
  sendAddProductRequest,
};
```

---

### File 3: NEW – `src/apis/services/v1/popularProduct.service.js`

```js
const { Logger: log } = require('sarvm-utility');
const stringSimilarity = require('string-similarity');
const GeoCatalog = require('../../models/mongoCatalog/geoCatalogSchema');
const Product = require('../../mongoModels/productSchema');
const ReqMasterProductService = require('./mongoCatalog/requestMasterCatalog');

const normalize = (str = '') =>
  String(str).toLowerCase().replace(/[^a-z0-9]/g, '').trim();

/**
 * Step 1: Get all popular products from geo_catalogs
 * Step 2: Get all product names from products collection
 * Step 3: Filter out exact matches
 * Step 4: Apply search + pagination
 */
const getUnmatchedPopularProducts = async (page = 1, pageSize = 10, search = '') => {
  try {
    // Fetch ALL geo_catalog documents
    const geoDocs = await GeoCatalog.find({}).lean();

    // Fetch ALL product names from products collection
    const allProducts = await Product.find({ st: { $ne: 'DELETED' } })
      .select('prdNm')
      .lean();

    const masterSet = new Set(
      allProducts.map((p) => normalize(p.prdNm)).filter(Boolean)
    );

    // Extract popular products from all geo docs, all categories
    const unmatchedProducts = [];

    geoDocs.forEach((doc) => {
      const pincode = doc.pincode || '';
      const city = doc.city || '';
      const state = doc.state || '';

      (doc.categories || []).forEach((cat) => {
        const popularProducts = cat.sections?.popular || [];

        popularProducts.forEach((prod) => {
          const normalizedName = normalize(prod.name);
          // Only include if NOT in master product collection
          if (!masterSet.has(normalizedName)) {
            unmatchedProducts.push({
              productName: prod.name,
              categoryName: cat.name,
              pincode,
              city,
              state,
              source: prod.source || 'AI',
              count: prod.count || 0,
            });
          }
        });
      });
    });

    // Deduplicate by productName + pincode
    const seen = new Set();
    const unique = unmatchedProducts.filter((item) => {
      const key = `${normalize(item.productName)}_${item.pincode}`;
      if (seen.has(key)) return false;
      seen.add(key);
      return true;
    });

    // Apply search filter
    let filtered = unique;
    if (search && search.trim()) {
      const s = search.toLowerCase().trim();
      filtered = unique.filter(
        (item) =>
          item.productName.toLowerCase().includes(s) ||
          item.pincode.toLowerCase().includes(s) ||
          item.city.toLowerCase().includes(s) ||
          item.state.toLowerCase().includes(s)
      );
    }

    // Pagination
    const total = filtered.length;
    const totalPages = Math.ceil(total / pageSize);
    const start = (page - 1) * pageSize;
    const paginated = filtered.slice(start, start + pageSize);

    return {
      products: paginated,
      total,
      totalPages,
      currentPage: page,
      pageSize,
    };
  } catch (error) {
    log.error({ error: 'Error in getUnmatchedPopularProducts', details: error.message });
    throw error;
  }
};

/**
 * Find products from products collection with >=30% name similarity
 */
const getSimilarProducts = async (name, page = 1, pageSize = 10) => {
  try {
    const allProducts = await Product.find({ st: { $ne: 'DELETED' } })
      .select('prdNm dumK catPnm')
      .lean();

    const similarProducts = [];

    allProducts.forEach((product) => {
      if (!product.prdNm) return;
      const similarity = stringSimilarity.compareTwoStrings(
        name.toLowerCase(),
        product.prdNm.toLowerCase()
      );
      if (similarity >= 0.3) {
        similarProducts.push({
          _id: product._id,
          productName: product.prdNm,
          dummyKey: product.dumK,
          categoryPath: product.catPnm || [],
          similarity: Math.round(similarity * 100),
        });
      }
    });

    // Sort by similarity descending
    similarProducts.sort((a, b) => b.similarity - a.similarity);

    const total = similarProducts.length;
    const totalPages = Math.ceil(total / pageSize);
    const start = (page - 1) * pageSize;
    const paginated = similarProducts.slice(start, start + pageSize);

    return {
      products: paginated,
      total,
      totalPages,
      currentPage: page,
      pageSize,
    };
  } catch (error) {
    log.error({ error: 'Error in getSimilarProducts', details: error.message });
    throw error;
  }
};

/**
 * Update old product name with new name in geo_catalogs wherever it exists
 */
const updateProductInGeoCatalog = async (oldName, newName, pincode, city, state) => {
  try {
    // Build query to find relevant geo_catalog documents
    const query = {};
    if (pincode) query.pincode = pincode;

    const geoDocs = await GeoCatalog.find(query);

    let updateCount = 0;

    for (const doc of geoDocs) {
      let modified = false;

      (doc.categories || []).forEach((cat) => {
        ['trending', 'popular'].forEach((section) => {
          const products = cat.sections?.[section] || [];
          products.forEach((prod) => {
            if (normalize(prod.name) === normalize(oldName)) {
              prod.name = newName;
              modified = true;
              updateCount++;
            }
          });
        });
      });

      if (modified) {
        doc.markModified('categories');
        await doc.save();
      }
    }

    return {
      success: true,
      message: 'Product updated',
      updatedCount: updateCount,
    };
  } catch (error) {
    log.error({ error: 'Error in updateProductInGeoCatalog', details: error.message });
    throw error;
  }
};

/**
 * Send request to add new product using existing requestMasterCatalog service
 */
const sendAddProductRequest = async (productName, category) => {
  try {
    const result = await ReqMasterProductService.createGeoCatalogRequestIfMissing({
      productName,
      category: category || 'uncategorized',
    });

    return {
      success: true,
      message: 'Request to add new product is send',
      data: result,
    };
  } catch (error) {
    log.error({ error: 'Error in sendAddProductRequest', details: error.message });
    throw error;
  }
};

module.exports = {
  getUnmatchedPopularProducts,
  getSimilarProducts,
  updateProductInGeoCatalog,
  sendAddProductRequest,
};
```

---

### File 4: MODIFY – `src/apis/routes/v1/index.js`

**Location:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\routes\v1\index.js`

Add these 2 lines:

```diff
 const geoRouter = require('./geo');
+const popularProductRouter = require('./popularProduct');

 const retailerCatalogRouter = require('./retailerCatalog/index');
```

```diff
 router.use('/geo', geoRouter);
+router.use('/popularProduct', popularProductRouter);
 router.use('/newProductReq', requestMasterCatalog);
```

After change the full file looks like:

```js
const express = require('express');
const router = express.Router();

const categoryRouter = require('./category');
const productRouter = require('./product');
const publishRouter = require('./publish');
const catalogRoutere = require('./catalog');
const metaDataRoutere = require('./metaData/index');
const DataTree = require('./DataTree');
const requestMasterCatalog = require('./requestMasterCatalog');
const customCatalog = require('./customCatalog');
const geoRouter = require('./geo');
const popularProductRouter = require('./popularProduct');

const retailerCatalogRouter = require('./retailerCatalog/index');
const bulkUpdateCatalog = require('./BulkUpdateProduct');

router.use('/customCatalog', customCatalog);
router.use('/geo', geoRouter);
router.use('/popularProduct', popularProductRouter);
router.use('/newProductReq', requestMasterCatalog);
router.use('/catalog', catalogRoutere);
router.use('/category', categoryRouter);
router.use('/product', productRouter);
router.use('/publish', publishRouter);
router.use('/metadata', metaDataRoutere);
router.use('/retailercatalog', retailerCatalogRouter);
router.use('/bulkupdate', bulkUpdateCatalog);
router.use('/dataTree', DataTree);

module.exports = router;
```

**API base path becomes:** `/cms/apis/v1/popularProduct/...`
