# NOVA — Flutter Commerce Demo

A responsive Flutter storefront for Android, iOS, and Web. The app is runnable without external credentials and includes a local demo catalog, account flow, persistent shopping state, checkout simulation, order history, and an admin workspace.

> **Demo-mode boundary:** this project currently uses a local demo authentication repository and `shared_preferences`; it does not connect to Firebase or a payment provider yet. Local roles, demo orders, and client-side checkout are for development and UX review only. Do not use them for real users, payments, or authorization. Firebase project setup and production security rules are described below.

## Project architecture

```text
lib/
├── main.dart                         # Provider setup and named routes
├── core/
│   ├── data/mock_catalog.dart         # Seed products/categories
│   ├── models.dart                    # User, product, address, and order models
│   ├── services/repositories.dart    # Auth/catalog/persistence contracts + demo adapters
│   ├── state/commerce_store.dart     # ChangeNotifier commerce state and calculations
│   ├── theme/app_theme.dart          # Shared NOVA design tokens and Material theme
│   └── widgets/
│       ├── app_shell.dart             # Responsive header, footer, mobile navigation
│       └── product_card.dart          # Reusable image, product card, and responsive grid
└── features/
    ├── auth/auth_pages.dart           # Sign-in, registration, access denied
    ├── store/store_pages.dart         # Home, categories, search/list, product details
    ├── cart/cart_pages.dart           # Cart, coupon, checkout, demo payment, confirmation
    ├── customer/customer_pages.dart  # Wishlist, profile, addresses, order tracking
    └── admin/admin_pages.dart         # Dashboard, product/category/order/user management

firestore.rules                        # Suggested production Firestore rules
storage.rules                          # Suggested product-media Storage rules
test/widget_test.dart                  # Storefront smoke test
```

Legacy UI files under `lib/src/` are retained from the original sample; the app entry point uses the `core/` and `features/` implementation above.

## Implemented flows

- Responsive home, category browsing, product cards/details, and image fallbacks.
- Search by product, brand, or category; category, price-range, and rating filters; featured, price, newest, popularity, and rating sorts.
- Demo sign-in and account registration; protected customer/admin routes.
- Wishlist, stock-aware cart quantity updates, `WELCOME10` coupon, shipping threshold, tax, and total calculations.
- Saved delivery addresses, default address selection, checkout form validation, simulated payment/cash-on-delivery selection, and order confirmation.
- Persistent local cart, wishlist, profile, catalog edits, addresses, and order history.
- Order tracking/status updates and admin CRUD for products/categories, plus demo user-role management.
- Empty, loading, and error states throughout the primary flows.

### Demo accounts

| Role | Email | Password |
|---|---|---|
| Customer | `user@shop.com` | `User@123` |
| Admin | `admin@shop.com` | `Admin@123` |

New demo registrations are available for the active app session; they are not real authenticated accounts. The seeded accounts work on every launch. Local preferences are not encrypted and must not hold production secrets or sensitive personal data.

## Run the app

Install Flutter 3.33+ (Dart 3.3+) and enable the desired platform. From the project root:

```powershell
flutter pub get
flutter run -d chrome
```

For Android, connect a device or start an emulator, then:

```powershell
flutter devices
flutter run -d <device-id>
```

Build release outputs:

```powershell
flutter build web
flutter build apk --release
```

The catalog uses remote Unsplash image URLs; a network connection is needed for those images. Broken/unavailable image URLs show a fallback image tile.

## Local data model

The local adapter stores JSON in `shared_preferences`, namespaced with `nova.*`:

- `nova.user` — current demo profile and role.
- `nova.users` — demo profile/role list shown in admin.
- `nova.products`, `nova.categories` — locally edited demo catalog.
- `nova.cart`, `nova.wishlist` — product IDs and quantities.
- `nova.addresses` — saved recipient and delivery information.
- `nova.orders` — order lines, totals, status, delivery address, and demo payment method.

`CommerceStore` depends on `AuthRepository`, `CatalogRepository`, and `PersistenceRepository` interfaces. Replace the demo adapters in `lib/core/services/repositories.dart` with Firebase-backed adapters to synchronize accounts, catalog, addresses, carts, and orders between devices. `shared_preferences` is device-local and is not a substitute for Firestore.

## Firebase production setup

Firebase credentials/project configuration are not included. Configure your own Firebase project before enabling a remote backend:

1. Create a Firebase project; enable Email/Password in Authentication, create Firestore, and enable Storage.
2. Install the FlutterFire CLI and configure platform apps:

   ```powershell
   dart pub global activate flutterfire_cli
   flutterfire configure
   ```

3. Add the SDKs and initialize Firebase from the generated `firebase_options.dart` before creating the app:

   ```powershell
   flutter pub add firebase_core firebase_auth cloud_firestore firebase_storage
   ```

   Follow the FlutterFire setup guide for generated options and platform configuration. Do not check in private service-account credentials. Firebase client API keys identify a project; authorization must come from Firebase Auth, claims, and rules, not from hiding a client key.

4. Implement Firebase adapters for the repository interfaces and inject them in `main.dart`. The checked-in app deliberately defaults to local demo adapters until that project-specific work is configured; adding Firebase packages alone does not switch the app to Firebase.
5. Deploy the starter rules below with the Firebase CLI, assign the `admin` custom claim only from a trusted Admin SDK environment, and test rules in the Firebase Emulator Suite before launch.
6. Use Firebase Storage for product media and save only download URLs/paths in product documents. Limit file type/size and authorize uploads with a server-controlled admin claim.
7. Store user profiles at `users/{uid}`, and scope carts and addresses to the authenticated UID. Orders must include the authenticated `userId`; never trust a client-supplied role, price, discount, or payment result.

### Suggested Firestore collections

```text
products/{productId}
  name, categoryId, brand, price, originalPrice, rating, reviews,
  imageUrl, description, stock, isNew, createdAt, updatedAt

categories/{categoryId}
  name, imageUrl, sortOrder, createdAt

users/{uid}
  name, email, phone, role             # role is not owner-editable

carts/{uid}
  items: [{ productId, quantity, updatedAt }]

addresses/{uid}/items/{addressId}
  name, phone, line1, city, postalCode, isDefault

orders/{orderId}
  userId, lines, subtotal, shipping, discount, tax, total,
  deliveryAddress, status, paymentStatus, paymentReference, createdAt
```

Deploy the included starter rules:

```powershell
firebase deploy --only firestore:rules,storage
```

The rules require a trusted `admin: true` custom claim for administrative writes. Add any project-specific field validation and run the emulator tests before production use.

## Payments

Checkout is intentionally simulated: it collects no card information and does not charge money. For Razorpay or Stripe, create a backend endpoint/Cloud Function to create the payment intent/order, verify the provider signature/webhook server-side, and only then mark a Firestore order paid. Never place a secret key or payment-signature verification in Flutter client code. Keep production payment credentials in server-side secret management.

## Validation

```powershell
flutter analyze lib\core lib\features lib\main.dart test
flutter test test\widget_test.dart
```
