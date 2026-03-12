# DG Platform - Multi-Restaurant Game & Review System

## Overview
Multi-restaurant platform ("DG" - Belgique) where restaurants are partners. Three separate spaces:
1. **Public Homepage** (`/`) - Discovery page with popular items, featured selection, restaurant listings, offers, search
2. **Admin DG** (`/admin`) - Platform owner dashboard with global stats and restaurant management
3. **Restaurant Dashboard** (`/dashboard/:id`) - Per-restaurant profile editing (incl. coverImage, cuisineType, shortDescription), menu management (categories + items + badges + promotions), reviews, pending wins, QR code
4. **Client-facing** (`/r/:qrCode`) - Full game flow: email → payment → wheel → result

## Architecture
- **Frontend**: React + Vite + TanStack Query + wouter routing + Tailwind CSS + shadcn/ui
- **Backend**: Express.js + PostgreSQL + Drizzle ORM
- **Payments**: Stripe Checkout Sessions API (via `stripe` + `stripe-replit-sync` packages, Replit Stripe connector)
- **Styling**: Premium dark theme with glass-card-premium effects, animated gradients, btn-premium CTA buttons, glow effects, framer-motion staggered animations, `prefers-reduced-motion` support, French language
- **Session**: localStorage for email/restaurant/method tracking during game flow

## Database Schema (shared/schema.ts)
- `restaurants` - Partner restaurants: name, slug, shortDescription, email, phone, address, logo, coverImage, cuisineType, socials, QR code, game settings, paymentPrice, isActive, isFeatured, displayOrder
- `reviews` - Client reviews (1-5 stars + comment) per restaurant
- `participants` - Game players scoped per restaurant (email + restaurantId unique)
- `spins` - Game spin results with claimed/claimedAt for win validation
- `platformSettings` - Global platform configuration
- `menu_categories` - Restaurant menu categories: id, restaurantId, name, slug, shortDescription, displayOrder, isActive
- `menu_items` - Restaurant menu items: id, restaurantId, categoryId, name, slug, description, price, image, mainImage, galleryImages, tags, available, displayOrder, isActive, isPopular, isFeatured, isBestSeller, isRecommended, isTopToday, marketingLabel, hasPromotion, promotionLabel, promotionType, promotionValue, oldPrice, newPrice, promotionStartDate, promotionEndDate
- `stripe.*` - Stripe sync tables (managed by stripe-replit-sync)

## Marketing Badge System
- Boolean fields on menu_items: isPopular, isFeatured, isBestSeller, isRecommended, isTopToday
- Legacy `tags` field (comma-separated) still works as fallback in all queries
- `marketingLabel` overrides default badge text when set
- Promotion support: hasPromotion, promotionLabel, oldPrice, newPrice (shows crossed-out price in UI)
- Restaurant-level: isFeatured (shows "En vedette" badge), displayOrder (controls listing order)
- isActive on restaurants, categories, and items controls visibility in public pages

## Food Images
- AI-generated food images stored in `client/public/images/menu/` (36 images total)
- All 35 MIOZZA menu items have `mainImage` set to `/images/menu/<item>.png`
- MIOZZA cover image: `/images/menu/miozza_cover.png`
- Images served via Vite's public directory (production-safe)
- RestaurantPublic shows cover image hero at top when `coverImage` exists
- MenuPage shows cover image in restaurant info card
- Fallback: items without images show gradient placeholder with utensils icon

## Likes / Thumbs-Up System
- `likesCount` field on menu_items (integer, default 0)
- `POST /api/menu/items/:itemId/like` — increments likesCount, returns 404 for invalid items, only targets active items
- Client-side: `LikeButton` component with localStorage tracking (`loukfaf_liked_items`) prevents double-likes
- Double-like prevention: checks localStorage before POST, syncs across duplicate cards for same item
- HomePage shows passive like counts on item cards
- MenuPage shows interactive like buttons on all product cards (featured, popular, category lists)
- Seeded initial likes for key MIOZZA items for social proof

## Support Ticket System (Admin-Restaurant Private Channel)
- `support_tickets` table: id, restaurantId, subject, category, message, priority, status, adminReply, repliedAt, createdAt
- Categories: supply_request, staff_request, menu_update, photo_update, promotion, operational, urgent, other
- Priority levels: normal, important, urgent (color-coded badges)
- Status workflow: new → seen → in_progress → replied → resolved → closed
- Restaurant Dashboard (SupportPanel): quick request buttons (8 predefined), custom form with priority selector, request history with status tracking
- Admin DG (AdminSupportInbox): all tickets sorted by priority/status, status progression buttons, quick reply templates (6 predefined), filter tabs (all/active/new/in_progress/replied/resolved/closed)
- Urgent/important tickets highlighted with colored borders; auto-refresh every 8s admin, 15s restaurant
- API: POST/GET /api/restaurant/:id/support, GET /api/admin/support, POST /api/admin/support/:id/reply, POST /api/admin/support/:id/close, POST /api/admin/support/:id/status

## Stripe Integration
- **stripeClient.ts** - Connects via Replit connector to get Stripe API keys
- **webhookHandlers.ts** - Processes Stripe webhooks via stripe-replit-sync
- **server/index.ts** - Webhook route registered BEFORE express.json() middleware
- **Flow**: Client clicks "Payer" → POST /api/create-checkout → Stripe Checkout → redirect back with session_id → GET /api/verify-payment/:sessionId → wheel access granted only if paid

## Client Game Flow
1. `/r/:qrCode` - Landing page: email entry + "Laisser un avis gratuitement" button
2. `/r/:qrCode/payment` - Payment page with Stripe Checkout button (0.99€)
3. `/r/:qrCode/wheel?session_id=...` - Stripe payment verified, then SpinningWheel
4. `/r/:qrCode/result` - Win (BINGO! + confetti + reward) or Loss (Oups!)

## Key Routes
### API
- `GET /api/discover` - Public discovery data (restaurants, popular items, featured items)
- `GET /api/restaurant/qr/:qrCode` - Public restaurant data (includes hasMenu flag)
- `POST /api/restaurant/:id/reviews` - Submit review
- `POST /api/play/check` - Check if player can play today
- `POST /api/play/spin` - Spin the wheel (body: email, method, restaurantId)
- `POST /api/create-checkout` - Create Stripe Checkout Session (body: email, restaurantId, qrCode)
- `GET /api/verify-payment/:sessionId` - Verify Stripe payment status
- `POST /api/stripe/webhook` - Stripe webhook handler (raw body)
- `GET /api/restaurant/:id/menu` - Menu categories + items (public)
- `POST /api/restaurant/:id/menu/categories` - Create menu category
- `POST /api/restaurant/:id/menu/items` - Create menu item
- `PUT /api/restaurant/:id/menu/items/:itemId` - Update menu item (scoped by restaurantId)
- `DELETE /api/restaurant/:id/menu/items/:itemId` - Delete menu item (scoped by restaurantId)
- `GET /api/restaurant/:id/pending-wins` - Unclaimed wins
- `POST /api/restaurant/:id/claim-win/:spinId` - Validate a win
- `PUT /api/restaurant/:id/menu/categories/:categoryId` - Update menu category
- `DELETE /api/restaurant/:id/menu/categories/:categoryId` - Delete category + its items
- `GET /api/admin/restaurants` - All restaurants
- `GET /api/admin/stats` - Global stats

### Pages
- `/` - Public discovery homepage (restaurants, popular items, featured selection, search)
- `/admin` - Admin DG dashboard
- `/register` - Register new restaurant
- `/r/:qrCode` - Client landing (email entry)
- `/r/:qrCode/payment` - Payment before spin
- `/r/:qrCode/menu` - Restaurant menu page (shown if restaurant has menu data)
- `/r/:qrCode/after-review` - Post-review confirmation
- `/r/:qrCode/wheel` - Spinning wheel game (requires session_id from Stripe)
- `/r/:qrCode/result` - Win/loss result
- `/r/:qrCode/review` - Direct review form
- `/dashboard/:id` - Restaurant dashboard

## Current Restaurants
- **MIOZZA** (ID=1, QR=MIOZZA-001) - probabilities: Café 33%, Remise 12%, Plat 7%, Boisson 15%, Dessert 8%, Cadeau 2%
- **FAFI** (ID=2, QR=FAFI-MMKH98V3) - probabilities: Café 30%, Plat/remise 15%, Remise 50% 10%, Boisson 2%, Cadeau 2%

## Security (Spin/Play System)
- **Payment required**: All spins require a valid Stripe session ID (no free method bypass)
- **Session reuse prevention**: `stripeSessionId` stored on each spin record with DB unique constraint — same session cannot be used twice (with race-condition protection via catch on unique violation)
- **Email verification**: Stripe session email must exactly match the request email (case-insensitive)
- **Restaurant ID verification**: Stripe metadata restaurantId must match the requested restaurant
- **Daily limit**: One spin per email per restaurant per calendar day
- **Server-side outcomes**: Win/loss determined entirely server-side using restaurant-specific probabilities from DB
- **Default win rate**: ~31% total (15+8+5+2+1), configurable per restaurant via `rewardsList` + `rewardsProbabilities`
- **Contact email**: support@dg-belgique.com shown on all error messages and page footers

## SEO & Performance
- `index.html`: lang=fr, meta description, OG/Twitter tags, theme-color for loukfaf.com
- Fonts trimmed to DM Sans + Outfit only (from 30+ fonts), loaded via HTML `<link>` only (no CSS @import duplicate)
- `prefers-reduced-motion` support on all animations
- Dynamic `document.title` per restaurant on client-facing pages (RestaurantPublic, Payment)

## Public API Enrichment
- `GET /api/restaurant/qr/:qrCode` now returns: address, totalPlayers, totalReviews, averageRating, rewardNames[]
- All data-driven: any restaurant with data automatically gets social proof + rewards preview on landing page

## Important Notes
- No auth system - admin and dashboard accessible by URL only
- Game price: 0.99€ (configurable per restaurant via paymentPrice field)
- Win validation: restaurant staff marks wins as claimed via dashboard
- SpinningWheel component uses named export (not default)
- Resend email integration dismissed - do not propose again
- Target domain: loukfaf.com
