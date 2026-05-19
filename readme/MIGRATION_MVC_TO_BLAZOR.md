# 🔄 MVC → Blazor Migration Playbook (WebShop)

This document defines the **scope, rules, and step-by-step process** to migrate the current **ASP.NET Core MVC** WebShop to a **pure Blazor Server** architecture.

---

## ✅ Rules

- Always append at the top (most recent section first).
- Checklist:
  - `- [ ]` = pending (to be developed)
  - `- ✅` = delivered
- Naming: use **WebShop** (avoid adding new “OnlineShoppingStore” strings; legacy namespaces will be migrated gradually).
- Writing pattern: scope → affected screens → rules → acceptance.
- Completion rule: only mark ✅ after manual UI verification (screens reviewed + validation messages + empty states + delete confirmations).

---

## 🎯 Goal

- Migrate **all user-facing and admin pages** that currently live in **MVC (Controllers + Views)** to **Blazor (.razor)**.
- After migration, **delete the legacy MVC architecture** (directories and files that are no longer needed) and keep **only Blazor**.
- Keep the UI consistent with our standards (menu/layout/cards/buttons) based on:
  - `readme/UI_PATTERNS_QUICK_START.md`
  - `readme/CODE_PATTERNS_AND_INFRASTRUCTURE.md`

---

## ✅ Non-Goals (for this migration pass)

- Rewriting the database schema unless required by the UI migration.
- Refactoring domain/data layers beyond what is necessary to support Blazor patterns (CancellationToken, async disposal, etc.).
- Changing business rules/flows (only the UI delivery mechanism changes).

---

## 🧭 Guiding Principles

- **One page at a time**: migrate incrementally, keep the app running.
- **Parity first**: replicate behavior and UI; optimize later.
- **Blazor patterns are mandatory** for async work and resilience:
  - `IAsyncDisposable` + `CancellationTokenSource` in data-loading pages
  - Error boundaries around critical UI sections
  - Propagate `CancellationToken` through Ports/Adapters/Repositories
- **UI patterns are mandatory**:
  - OpenIconic (`oi oi-*`) for navigation
  - Bootstrap Icons (`bi bi-*`) for actions
  - Standard button sizing and templates from the UI guide

---

## 🧱 Target Architecture (Blazor-Only)

### 📁 Suggested folder structure

- **`Components/`**: Reusable UI building blocks (tables, modals, cards, form helpers)
- **`Pages/`**: Routeable pages (`.razor`) replacing MVC views
  - `Pages/Store/` (catalog/product browsing)
  - `Pages/Cart/`
  - `Pages/Admin/` (products/categories/orders)
  - `Pages/Account/` (register/login/logout profile)
- **`Layouts/`**: App layout(s), nav menu, sidebars, headers
- **`wwwroot/`**: Static assets (css/js/images), Bootstrap, icons

> We keep existing Application/Infrastructure/Data layers as-is unless a page migration requires changes for Blazor patterns.

---

## 🗺️ MVC → Blazor Mapping

| MVC | Blazor Server |
|-----|---------------|
| Controller action (`HomeController.Index`) | Page route (`Pages/Home.razor` with `@page "/"`) |
| View (`.cshtml`) | Razor component (`.razor`) |
| ViewModel passed to View | Component state + DTO/ViewModel injected/loaded |
| Partial views | Components (`Components/*.razor`) |
| Layout (`_Layout.cshtml`) | `Layouts/MainLayout.razor` |
| TagHelpers / HTML helpers | Components + standard HTML + Bootstrap |
| TempData/ViewBag | Component state + scoped services |
| Server-side validation | DataAnnotations + validation patterns used in UI guide |

---

## 🔍 Phase 0 — Inventory

- ✅ Routes/pages: every MVC page and URL
- ✅ Dependencies per page (repos/services, auth/roles, forms/validation, tables/filters/paging, modals)
- ✅ Shared UI inventory (nav/menu items, cards, table/action patterns)

---

## 🧾 Current MVC Inventory (found in this repo)

### 🛣️ Routing

- **Route pattern**: `"{controller=Home}/{action=Index}/{id?}"`
- **MVC is enabled** via `AddControllersWithViews()`

### 🧩 Controllers → Actions → Views

> Default view name is inferred when not specified (e.g. `return View()` → `Views/{Controller}/{Action}.cshtml`).

#### 🏠 Home

- **`Home/Index`** → `Views/Home/Index.cshtml`
- **`Home/Privacy`** → `Views/Home/Privacy.cshtml`
- **`Home/Contact`** → `Views/Shared/contact.cshtml` *(action name is `contact` in code)*
- **`Home/Error`** → `Views/Shared/Error.cshtml`

#### 🛍️ Store / Products

- **`Product/ShowAll`** → `Views/Product/Catalog.cshtml` *(explicit view: `"Catalog"`)*
- **`Product/Details?ProductId={id}&temp={bool?}`** → `Views/Product/item.cshtml` *(explicit view: `"Item"`, uses `TempData["Message"]` + `ViewBag.temp`)*
- **`Product/ProductsByCategory?CategoryId={id}`** → `Views/Product/ProductCategory.cshtml` *(explicit view: `"ProductCategory"`)*

#### 🧺 Cart

- **`Cart/ShowCartItems`** → `Views/Cart/ShowCartToEdit.cshtml`
- **`Cart/AddToCart`** *(POST)* → returns **`NoContent()`** or redirects to `Product/Details`
- **`Cart/DeleteItem`** → redirects to `Cart/ShowCartItems`
- **`Cart/UpdateCartItem`** → redirects to `Cart/ShowCartItems`
- **`Cart/EditQuantity`** → redirects to `Cart/ShowCartItems`
- **View** `Views/Cart/ShowCart.cshtml` exists (not referenced directly by the controller we read yet)

#### 🧑‍💼 Admin

- **`Admin/DashBoard`** → `Views/Admin/DashBoard.cshtml`
- **`Admin/AddProduct`** → `Views/Admin/AddProduct.cshtml`
- **`Admin/SaveNewProduct`** *(POST)* → redirects to `Admin/DashBoard` or returns `Views/Admin/AddProduct.cshtml`
- **`Admin/EditProduct?productId={id}&temp={bool}`** → `Views/Admin/EditProduct.cshtml`
- **`Admin/SaveUpdatedProduct`** *(POST)* → redirects to `Admin/SearchForProduct` or `Admin/ShowAllProducts`
- **`Admin/DeleteProduct?productId={id}&temp={bool?}`** → redirects back to product lists
- **`Admin/SearchForProduct?productId={string?}&name={string?}`** → `Views/Admin/SearchForProduct.cshtml`
- **`Admin/ShowAllOrders`** → `Views/Admin/ShowOrders.cshtml`
- **`Admin/UpdateStatus`** → redirects to `Admin/ShowAllOrders`
- **`Admin/SearchForOrder?OrderId={int?}`** → `Views/Admin/ShowOrders.cshtml`
- **`Admin/GetAllCustomers`** → `Views/Admin/AllCustomers.cshtml`
- **`Admin/ShowAllProducts`** → `Views/Admin/ShowAllProducts.cshtml`
- **`Admin/RepeatedProductName?Name={string}`** → JSON (validator endpoint)

#### 🧾 Orders

- **`Order/CheckOut`** → `Views/Order/CheckOut.cshtml` *(populates `ViewBag` with user data)*
- **`Order/AddOrder`** → redirects to `Order/CustomerOrders`
- **`Order/CustomerOrders`** → `Views/Order/CustomerOrders.cshtml`

#### 🔐 Account / Roles

- **`Account/Register`** → `Views/Account/Register.cshtml`
- **`Account/saveRegister`** *(POST)* → redirects to `Home/Index` or returns `Views/Account/Register.cshtml`
- **`Account/Login`** *(GET)* → `Views/Account/Login.cshtml`
- **`Account/Login`** *(POST)* → redirects to `Admin/DashBoard` (admin) or `Home/Index`
- **`Account/SignOut`** → redirects to `Home/Index`
- **`Account/ProfileAsync`** → `Views/Shared/Profile.cshtml`
- **`Account/CheckUserName`** → JSON
- **`Role/AddRole`** → `Views/Role/AddRole.cshtml`
- **`Role/SaveRole`** → `Views/Role/AddRole.cshtml`

#### 🧯 Other

- `CustomerController`, `CartItemController` currently only expose `Index()` without known views in the inventory above.
- `DiscountController` exists but is not an MVC controller (no `: Controller`) and has no active actions.

### 🧱 Shared MVC UI Assets (to be replaced by Blazor)

- **Layout**: `Views/Shared/_Layout1.cshtml` is the active layout (set by `Views/_ViewStart.cshtml`)
- Additional layout exists: `Views/Shared/_Layout.cshtml`
- Partials: `Views/Shared/_LoginPartial.cshtml`, `Views/Shared/_ValidationScriptsPartial.cshtml`
- View infrastructure: `Views/_ViewImports.cshtml`, `Views/_ViewStart.cshtml`

---

## 🧩 Phase 1 — Baseline Blazor Shell (no business logic changes)

- ✅ Add Blazor Server hosting (App/Routing) and keep app runnable
- ✅ Create `Layouts/MainLayout.razor` aligned with UI patterns
- ✅ Create navigation menu (OpenIconic icons for nav)
- ✅ Integrate Bootstrap + icon packs required by the UI patterns
- ✅ Add placeholder Blazor pages for the main sections (Store / Cart / Admin / Account)
- ✅ Update `ConnectionStrings:connWebshop` to SQL Server `MULLER` / database `WebShop`
- ✅ Run EF Core migrations against SQL Server (`dotnet ef database update`)

---

## 🧱 Phase 2 — Extract Shared Components

- ✅ Data table wrapper (responsive + `table-dark` header + `@key`)
- ✅ Grid action buttons component (edit/delete; Bootstrap icons)
- ✅ Delete confirmation modal component (standard modal template)
- ✅ Form buttons component (save/cancel + spinner while saving)
- ✅ Validation helpers/components per UI patterns
- ✅ Standard back button helper (OpenIconic + correct classes)

---

## 📄 Phase 3 — Migrate Pages

- [ ] **Home**: Index
- [ ] **Home**: Privacy
- [ ] **Home**: Contact
- [ ] **Products**: Catalog (ShowAll)
- [ ] **Products**: Details (Item)
- [ ] **Products**: By Category
- [ ] **Cart**: ShowCartItems (editable cart)
- [ ] **Cart**: ShowCart (legacy cart view, verify if used)
- [ ] **Orders**: Checkout
- [ ] **Orders**: Customer Orders
- [ ] **Admin**: Dashboard
- [ ] **Admin**: Products (Add / Edit / List / Search / Delete)
- [ ] **Admin**: Orders (List / Search / Update Status)
- [ ] **Admin**: Customers (List)
- [ ] **Account**: Register
- [ ] **Account**: Login
- [ ] **Account**: Logout
- [ ] **Account**: Profile
- [ ] **Roles**: Add Role / Save Role (if needed in the Blazor-only app)

---

## 🔐 Phase 4 — Auth & Authorization

- [ ] Standardize authorization in Blazor (`[Authorize]`, `AuthorizeView`)
- [ ] Protect Admin pages consistently (roles/policies)
- [ ] Ensure Register / Login / Logout work in Blazor
- [ ] Ensure Profile works in Blazor
- [ ] Remove dependency on MVC identity UI routes (if any remain)

---

## 🧹 Phase 5 — Remove MVC (only after parity)

- [ ] Delete legacy MVC directories/files (`Controllers/`, `Views/`, `Areas/`, MVC-only helpers)
- [ ] Remove MVC service registration (`AddControllersWithViews`) if unused
- [ ] Remove MVC routing (`MapControllerRoute`) if unused
- [ ] Ensure no menu links point to MVC URLs
- [ ] Build + run with Blazor-only architecture

---

## 🧪 Definition of Done (Migration Complete)

- [ ] All previous MVC routes have Blazor replacements (or intentional redirects)
- [ ] Admin/product/cart/account flows work end-to-end
- [ ] UI matches our button/icon/table/modal patterns
- [ ] No MVC folders remain (or they are empty and deleted)
- [ ] Solution builds clean and runs locally
- [ ] Basic smoke test passes (catalog, cart, admin CRUD, auth)

---

## 📌 Notes / Open Questions (to be answered during Phase 0)

- Which routes must remain stable for SEO/bookmarks?
- Do we need a separate admin layout?
- Which pages are most critical (MVP) vs optional?
- Any MVC-only behaviors that Blazor should replicate (e.g., server-side redirects, TempData messages)?

