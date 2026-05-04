# Popular Product Feature – Frontend Implementation

## Overview
This document covers the Angular frontend changes for the admin panel to support the Popular Product feature.

**Files to create:**
1. `src/app/pages/popular-product/popular-product.component.ts`
2. `src/app/pages/popular-product/popular-product.component.html`
3. `src/app/pages/popular-product/popular-product.component.scss`
4. `src/app/pages/popular-product/popular-product-update/popular-product-update.component.ts`
5. `src/app/pages/popular-product/popular-product-update/popular-product-update.component.html`
6. `src/app/pages/popular-product/popular-product-update/popular-product-update.component.scss`
7. `src/app/lib/services/popular-product.service.ts`

**Files to modify:**
1. `src/app/config/constants.ts` — Add API URLs
2. `src/app/app-routing.module.ts` — Add routes
3. `src/app/app.module.ts` — Declare components
4. `src/app/layout/sidebar/sidebar.component.html` — Add sidebar link

---

## File 1: NEW – `src/app/lib/services/popular-product.service.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\lib\services\popular-product.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { CommonApi } from './api/common.api';
import { ApiUrls } from 'src/app/config/constants';

@Injectable({
  providedIn: 'root',
})
export class PopularProductService {
  constructor(private commonApi: CommonApi) {}

  getUnmatchedProducts(page: number, pageSize: number, search: string): Observable<any> {
    const params = `?page=${page}&pageSize=${pageSize}&search=${search}`;
    return this.commonApi.getData(`${ApiUrls.popularProduct.list}${params}`);
  }

  getSimilarProducts(name: string, page: number, pageSize: number): Observable<any> {
    const params = `?name=${encodeURIComponent(name)}&page=${page}&pageSize=${pageSize}`;
    return this.commonApi.getData(`${ApiUrls.popularProduct.similar}${params}`);
  }

  updateProductName(oldName: string, newName: string, pincode: string, city: string, state: string): Observable<any> {
    return this.commonApi.putData(`${ApiUrls.popularProduct.update}`, {
      oldName,
      newName,
      pincode,
      city,
      state,
    });
  }

  sendAddProductRequest(productName: string, category: string): Observable<any> {
    return this.commonApi.postData(`${ApiUrls.popularProduct.addRequest}`, {
      productName,
      category,
    });
  }
}
```

---

## File 2: MODIFY – `src/app/config/constants.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\config\constants.ts`

Add inside the `ApiUrls` object (after the `wallet` block, before the closing `};` on ~line 199):

```diff
   wallet: {
     pendingPayouts: `/wlt/api/reconcile/retailer-pending-payouts`,
     pay_now: (retailerId: any, safeUtr: any, adminUser: any) => `/wlt/api/settlements/${retailerId}/${safeUtr}?adminUser=${adminUser}`,
     settlement_History: `/wlt/api/settlements/history`,
   },
+
+  popularProduct: {
+    list: '/cms/apis/v1/popularProduct',
+    similar: '/cms/apis/v1/popularProduct/similar',
+    update: '/cms/apis/v1/popularProduct/update',
+    addRequest: '/cms/apis/v1/popularProduct/addRequest',
+  },
 };
```

---

## File 3: NEW – Main Listing Component

### `popular-product.component.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { ToastrService } from 'ngx-toastr';
import { PopularProductService } from 'src/app/lib/services/popular-product.service';

@Component({
  selector: 'app-popular-product',
  templateUrl: './popular-product.component.html',
  styleUrls: ['./popular-product.component.scss'],
})
export class PopularProductComponent implements OnInit {
  products: any[] = [];
  searchText: string = '';
  currentPage: number = 1;
  pageSize: number = 10;
  totalItems: number = 0;
  totalPages: number = 0;
  loading: boolean = false;

  constructor(
    private popularProductService: PopularProductService,
    private router: Router,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(): void {
    this.loading = true;
    this.popularProductService
      .getUnmatchedProducts(this.currentPage, this.pageSize, this.searchText)
      .subscribe({
        next: (res: any) => {
          this.products = res?.data?.products || [];
          this.totalItems = res?.data?.total || 0;
          this.totalPages = res?.data?.totalPages || 0;
          this.currentPage = res?.data?.currentPage || 1;
          this.loading = false;
        },
        error: (err) => {
          this.toastr.error('Failed to load products');
          this.loading = false;
        },
      });
  }

  onSearch(): void {
    this.currentPage = 1;
    this.loadProducts();
  }

  onPageChange(event: any): void {
    if (event.next) {
      this.currentPage++;
    } else if (event.prev) {
      this.currentPage--;
    } else {
      this.pageSize = event.pageSize;
      this.currentPage = 1;
    }
    this.loadProducts();
  }

  goToUpdate(product: any): void {
    this.router.navigate(['/popular-product-update'], {
      queryParams: {
        productName: product.productName,
        pincode: product.pincode,
        city: product.city,
        state: product.state,
        category: product.categoryName,
      },
    });
  }
}
```

### `popular-product.component.html`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product.component.html`

```html
<div class="main-content p-4 popular-product-wrapper bg-white">
  <h1 class="page-title mb-4">Popular Product</h1>

  <!-- Search Bar -->
  <div class="search-bar mb-3">
    <input
      type="text"
      [(ngModel)]="searchText"
      (keyup.enter)="onSearch()"
      placeholder="Search by Product Name, Pincode, City, State..."
      class="form-input w-100"
    />
  </div>

  <!-- Info Row -->
  <div class="info-row mb-2">
    <span>Total Products: <strong>{{ totalItems }}</strong></span>
  </div>

  <!-- Table -->
  <div class="table-wrapper">
    <table class="table table-bordered table-hover">
      <thead class="table-header">
        <tr>
          <th>#</th>
          <th>Product Name</th>
          <th>Pincode</th>
          <th>City</th>
          <th>State</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let product of products; let i = index">
          <td>{{ (currentPage - 1) * pageSize + i + 1 }}</td>
          <td>{{ product.productName }}</td>
          <td>{{ product.pincode }}</td>
          <td>{{ product.city }}</td>
          <td>{{ product.state }}</td>
          <td>
            <button class="btn update-btn" (click)="goToUpdate(product)">
              Update
            </button>
          </td>
        </tr>
        <tr *ngIf="products.length === 0 && !loading">
          <td colspan="6" class="text-center py-4">No data available</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- Pagination -->
  <app-pagination
    [totalItems]="totalItems"
    [pageNo]="currentPage"
    [itemsPerPage]="pageSize"
    (pageChange)="onPageChange($event)"
  ></app-pagination>
</div>
```

### `popular-product.component.scss`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product.component.scss`

```scss
.popular-product-wrapper {
  .page-title {
    font-size: 20px;
    color: #212529;
    margin-bottom: 24px;
  }

  .search-bar {
    .form-input {
      padding: 7px 12px;
      border: 1px solid rgba(33, 37, 41, 0.32);
      border-radius: 8px;
      font-size: 14px;
      outline: none;

      &::placeholder {
        color: rgba(33, 37, 41, 0.6);
      }
    }
  }

  .info-row {
    font-size: 14px;
    color: #555;
  }

  .table-header {
    background-color: #DEF5DF;
    th {
      font-size: 14px;
      font-weight: 600;
      color: #000;
      text-align: center;
    }
  }

  tbody td {
    font-size: 14px;
    text-align: center;
    vertical-align: middle;
  }

  .update-btn {
    background-color: #0BBDB5;
    color: #fff;
    font-size: 12px;
    font-weight: 700;
    border-radius: 4px;
    padding: 4px 16px;
  }
}
```

---

## File 4: NEW – Update Page Component

### `popular-product-update.component.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product-update\popular-product-update.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { ToastrService } from 'ngx-toastr';
import { PopularProductService } from 'src/app/lib/services/popular-product.service';

@Component({
  selector: 'app-popular-product-update',
  templateUrl: './popular-product-update.component.html',
  styleUrls: ['./popular-product-update.component.scss'],
})
export class PopularProductUpdateComponent implements OnInit {
  // Original product details
  productName: string = '';
  pincode: string = '';
  city: string = '';
  state: string = '';
  category: string = '';

  // Similar products
  similarProducts: any[] = [];
  similarCurrentPage: number = 1;
  similarPageSize: number = 10;
  similarTotalItems: number = 0;
  similarTotalPages: number = 0;
  loadingSimilar: boolean = false;

  // Replacement flow
  selectedReplacement: any = null;
  showReplaceUI: boolean = false;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private popularProductService: PopularProductService,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.route.queryParams.subscribe((params) => {
      this.productName = params['productName'] || '';
      this.pincode = params['pincode'] || '';
      this.city = params['city'] || '';
      this.state = params['state'] || '';
      this.category = params['category'] || '';
      this.loadSimilarProducts();
    });
  }

  loadSimilarProducts(): void {
    if (!this.productName) return;
    this.loadingSimilar = true;
    this.popularProductService
      .getSimilarProducts(this.productName, this.similarCurrentPage, this.similarPageSize)
      .subscribe({
        next: (res: any) => {
          this.similarProducts = res?.data?.products || [];
          this.similarTotalItems = res?.data?.total || 0;
          this.similarTotalPages = res?.data?.totalPages || 0;
          this.loadingSimilar = false;
        },
        error: () => {
          this.toastr.error('Failed to load similar products');
          this.loadingSimilar = false;
        },
      });
  }

  onSimilarPageChange(event: any): void {
    if (event.next) {
      this.similarCurrentPage++;
    } else if (event.prev) {
      this.similarCurrentPage--;
    } else {
      this.similarPageSize = event.pageSize;
      this.similarCurrentPage = 1;
    }
    this.loadSimilarProducts();
  }

  onReplace(product: any): void {
    this.selectedReplacement = product;
    this.showReplaceUI = true;
  }

  cancelReplace(): void {
    this.selectedReplacement = null;
    this.showReplaceUI = false;
  }

  onUpdate(): void {
    if (!this.selectedReplacement) return;
    this.popularProductService
      .updateProductName(
        this.productName,
        this.selectedReplacement.productName,
        this.pincode,
        this.city,
        this.state
      )
      .subscribe({
        next: () => {
          this.toastr.success('Product updated');
          this.router.navigate(['/popular-product']);
        },
        error: () => {
          this.toastr.error('Failed to update product');
        },
      });
  }

  onSendRequest(): void {
    this.popularProductService
      .sendAddProductRequest(this.productName, this.category)
      .subscribe({
        next: () => {
          this.toastr.success('Request to add new product is send');
          this.router.navigate(['/popular-product']);
        },
        error: () => {
          this.toastr.error('Failed to send request');
        },
      });
  }
}
```

### `popular-product-update.component.html`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product-update\popular-product-update.component.html`

```html
<div class="main-content p-4 update-wrapper bg-white">
  <h1 class="page-title mb-4">Update Popular Product</h1>

  <!-- Selected Product Details -->
  <div class="product-details-card mb-4">
    <h5>Selected Product Details</h5>
    <div class="detail-row"><strong>Product Name:</strong> {{ productName }}</div>
    <div class="detail-row"><strong>Pincode:</strong> {{ pincode }}</div>
    <div class="detail-row"><strong>City:</strong> {{ city }}</div>
    <div class="detail-row"><strong>State:</strong> {{ state }}</div>
  </div>

  <!-- Replacement UI (shown after clicking Replace on a similar product) -->
  <div class="replace-card mb-4" *ngIf="showReplaceUI && selectedReplacement">
    <h5>Replacement Preview</h5>
    <div class="detail-row">
      <strong>Original Product:</strong> {{ productName }}
    </div>
    <div class="detail-row">
      <strong>Replace With:</strong> {{ selectedReplacement.productName }}
      <span class="badge bg-info ms-2">{{ selectedReplacement.similarity }}% match</span>
    </div>
    <div class="action-buttons mt-3">
      <button class="btn btn-update me-2" (click)="onUpdate()">Update</button>
      <button class="btn btn-add-request me-2" (click)="onSendRequest()">
        Send request to add new product
      </button>
      <button class="btn btn-cancel" (click)="cancelReplace()">Cancel</button>
    </div>
  </div>

  <!-- Similar Products Table -->
  <div class="similar-section">
    <h5 class="mb-3">Similar Products (30%+ match)</h5>

    <div class="table-wrapper">
      <table class="table table-bordered table-hover">
        <thead class="table-header">
          <tr>
            <th>#</th>
            <th>Product Name</th>
            <th>Category</th>
            <th>Similarity</th>
            <th>Action</th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let product of similarProducts; let i = index">
            <td>{{ (similarCurrentPage - 1) * similarPageSize + i + 1 }}</td>
            <td>{{ product.productName }}</td>
            <td>{{ product.categoryPath?.join(' > ') }}</td>
            <td>
              <span class="badge bg-success">{{ product.similarity }}%</span>
            </td>
            <td>
              <button class="btn replace-btn" (click)="onReplace(product)">Replace</button>
            </td>
          </tr>
          <tr *ngIf="similarProducts.length === 0 && !loadingSimilar">
            <td colspan="5" class="text-center py-4">No similar products found</td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Similar Products Pagination -->
    <app-pagination
      [totalItems]="similarTotalItems"
      [pageNo]="similarCurrentPage"
      [itemsPerPage]="similarPageSize"
      (pageChange)="onSimilarPageChange($event)"
    ></app-pagination>
  </div>
</div>
```

### `popular-product-update.component.scss`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\pages\popular-product\popular-product-update\popular-product-update.component.scss`

```scss
.update-wrapper {
  .page-title {
    font-size: 20px;
    color: #212529;
  }

  .product-details-card,
  .replace-card {
    background: #f8f9fa;
    border: 1px solid #dee2e6;
    border-radius: 8px;
    padding: 16px;

    h5 {
      font-size: 16px;
      font-weight: 600;
      margin-bottom: 12px;
    }

    .detail-row {
      font-size: 14px;
      margin-bottom: 6px;
    }
  }

  .replace-card {
    background: #fff3cd;
    border-color: #ffc107;
  }

  .action-buttons {
    .btn-update {
      background-color: #0BBDB5;
      color: #fff;
      font-weight: 700;
      font-size: 13px;
      padding: 6px 20px;
      border-radius: 4px;
    }
    .btn-add-request {
      background-color: #448FDA;
      color: #fff;
      font-weight: 700;
      font-size: 13px;
      padding: 6px 20px;
      border-radius: 4px;
    }
    .btn-cancel {
      background-color: #6c757d;
      color: #fff;
      font-size: 13px;
      padding: 6px 20px;
      border-radius: 4px;
    }
  }

  .table-header {
    background-color: #DEF5DF;
    th {
      font-size: 14px;
      font-weight: 600;
      color: #000;
      text-align: center;
    }
  }

  tbody td {
    font-size: 14px;
    text-align: center;
    vertical-align: middle;
  }

  .replace-btn {
    background-color: #0BBDB5;
    color: #fff;
    font-size: 12px;
    font-weight: 700;
    border-radius: 4px;
    padding: 4px 16px;
  }
}
```

---

## File 5: MODIFY – `src/app/config/constants.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\config\constants.ts`

Add the `popularProduct` block inside `ApiUrls` (after `wallet` block around line 199):

```typescript
  popularProduct: {
    list: '/cms/apis/v1/popularProduct',
    similar: '/cms/apis/v1/popularProduct/similar',
    update: '/cms/apis/v1/popularProduct/update',
    addRequest: '/cms/apis/v1/popularProduct/addRequest',
  },
```

---

## File 6: MODIFY – `src/app/app-routing.module.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\app-routing.module.ts`

### Step 6a: Add imports at top (after line 43)

```typescript
import { PopularProductComponent } from './pages/popular-product/popular-product.component';
import { PopularProductUpdateComponent } from './pages/popular-product/popular-product-update/popular-product-update.component';
```

### Step 6b: Add routes inside `routes` array (before the closing `];` around line 87)

```typescript
  { path: 'popular-product', component: PopularProductComponent, canActivate: [AuthGuard] },
  { path: 'popular-product-update', component: PopularProductUpdateComponent, canActivate: [AuthGuard] },
```

### Step 6c: Add to `routingComponents` array (line 96)

Add `PopularProductComponent, PopularProductUpdateComponent` to the end of the array.

---

## File 7: MODIFY – `src/app/app.module.ts`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\app.module.ts`

### Step 7a: Add imports at top

```typescript
import { PopularProductComponent } from './pages/popular-product/popular-product.component';
import { PopularProductUpdateComponent } from './pages/popular-product/popular-product-update/popular-product-update.component';
import { PopularProductService } from './lib/services/popular-product.service';
```

### Step 7b: Add to `declarations` array (after `WithdrawalRequestComponent` on line 114)

```typescript
    PopularProductComponent,
    PopularProductUpdateComponent,
```

### Step 7c: Add to `providers` array (after line 157)

```typescript
    PopularProductService,
```

---

## File 8: MODIFY – `src/app/layout/sidebar/sidebar.component.html`

**Path:** `d:\Sarvm\frontend\admin_panel\src\app\layout\sidebar\sidebar.component.html`

Add a new `<li>` block after the Subscription `<li>` (after line 216):

```html
        <!-- Popular Product -->
        <li class="menu" [routerLinkActive]="['active']">
          <a [routerLink]="['/popular-product']" aria-expanded="false" class="dropdown-toggle">
            <div>
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
                stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
                class="feather feather-star">
                <polygon
                  points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2">
                </polygon>
              </svg>
              <span>Popular Product</span>
            </div>
          </a>
        </li>
```

---

## API Endpoint Summary

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/cms/apis/v1/popularProduct?page=1&pageSize=10&search=` | Get unmatched popular products |
| GET | `/cms/apis/v1/popularProduct/similar?name=xyz&page=1&pageSize=10` | Get similar products (30%+ match) |
| PUT | `/cms/apis/v1/popularProduct/update` | Update product name in geo_catalog |
| POST | `/cms/apis/v1/popularProduct/addRequest` | Send request to add new product |

---

## Flow Summary

1. **Sidebar** → Click "Popular Product" → navigates to `/popular-product`
2. **Main Page** → Shows unmatched popular products (from `geo_catalogs` NOT in `products`) with search + pagination
3. **Click Update** → navigates to `/popular-product-update` with query params
4. **Update Page** → Shows product details + similar products table (30%+ match) with pagination
5. **Click Replace** → Shows replacement UI with original and new name
6. **Click "Update"** → Updates name in `geo_catalogs` → toaster "Product updated" → redirect back
7. **Click "Send request to add new product"** → Creates request via existing `requestMasterCatalog` service → toaster → redirect back
8. After either action, the product no longer appears in the listing
