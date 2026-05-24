# Sites & Ingestion Strategy Registry (`SITELIST.md`)

This file logs target Australian retail platforms, grouped programmatically by their underlying e-commerce architecture. This structure directly dictates how the n8n ingestion arrays route each site.

---

## Strategy A: Shopify JSON Feed Exploitation
**Programmatic Route:** `Strategy_JSON_Feed`
**Execution Overhead:** Near 0% (Pure HTTP requests).
**Exploitation Method:** Append `/products.json?limit=250` directly to the collection URL path to completely bypass HTML rendering and capture raw data objects (Title, Handle, Variant Prices, Vendor, Tags, Images).

### Active Target Array
* **Barrel and Batch**
    * Base Collection: `https://barrelandbatch.com.au/collections/rum`
    * JSON Feed Payload: `https://barrelandbatch.com.au/collections/rum/products.json?limit=250`
* **Bottle Stop**
    * Base Collection: `https://bottle-stop.com.au/collections/rum`
    * JSON Feed Payload: `https://bottle-stop.com.au/collections/rum/products.json?limit=250`
* **Caravan Wines and Spirits**
    * Base Collection: `https://caravanwinesandspirits.com.au/collections/rum`
    * JSON Feed Payload: `https://caravanwinesandspirits.com.au/collections/rum/products.json?limit=250`
* **Kent Street Cellars**
    * Base Collection: `https://kentstreetcellars.com.au/collections/rum`
    * JSON Feed Payload: `https://kentstreetcellars.com.au/collections/rum/products.json?limit=250`
* **Liquor Loot**
    * Base Collection: `https://liquorloot.com/collections/rum-bottles`
    * JSON Feed Payload: `https://liquorloot.com/collections/rum-bottles/products.json?limit=250`
* **Liquor Wine Cave**
    * Base Collection: `https://liquorwinecave.com.au/collections/rums`
    * JSON Feed Payload: `https://liquorwinecave.com.au/collections/rums/products.json?limit=250`
* **Luca Collections**
    * Base Collection: `https://lucacollections.com.au/collections/all-rum-collection`
    * JSON Feed Payload: `https://lucacollections.com.au/collections/all-rum-collection/products.json?limit=250`
* **Market Wine Store**
    * Base Collection: `https://marketwinestore.com.au/collections/rum`
    * JSON Feed Payload: `https://marketwinestore.com.au/collections/rum/products.json?limit=250`
* **Pauls Liquor**
    * Base Collection: `https://paulsliquor.com.au/collections/rum`
    * JSON Feed Payload: `https://paulsliquor.com.au/collections/rum/products.json?limit=250`
* **PNV Merchants**
    * Base Collection: `https://pnvmerchants.com/collections/rum`
    * JSON Feed Payload: `https://pnvmerchants.com/collections/rum/products.json?limit=250`
* **Sense of Taste**
    * Base Collection: `https://senseoftaste.com.au/collections/rum`
    * JSON Feed Payload: `https://senseoftaste.com.au/collections/rum/products.json?limit=250`
* **Spirit of the Maker** (Specialist: Australian Rum)
    * Base Collection: `https://spiritofthemaker.com.au/collections/australian-rum`
    * JSON Feed Payload: `https://spiritofthemaker.com.au/collections/australian-rum/products.json?limit=250`
* **Spirits of France**
    * Base Collection: `https://spiritsoffrance.com.au/collections/rum`
    * JSON Feed Payload: `https://spiritsoffrance.com.au/collections/rum/products.json?limit=250`
* **United Cellars**
    * Base Collection: `https://unitedcellars.com.au/collections/rum`
    * JSON Feed Payload: `https://unitedcellars.com.au/collections/rum/products.json?limit=250`
* **Uptown Liquor**
    * Base Collection: `https://uptownliquor.com.au/collections/rum`
    * JSON Feed Payload: `https://uptownliquor.com.au/collections/rum/products.json?limit=250`
* **Wine Sellers Direct**
    * Base Collection: `https://winesellersdirect.com.au/collections/spirits-rum`
    * JSON Feed Payload: `https://winesellersdirect.com.au/collections/spirits-rum/products.json?limit=250`

---

## Strategy B: Standard HTML DOM / Semantic Extraction
**Programmatic Route:** `Strategy_HTML_DOM`
**Execution Overhead:** Low (Standard HTTP page sourcing with direct HTML DOM cheerio parsing).
**Exploitation Method:** Target standard layouts (like WooCommerce) using unified CSS class selectors (`.product`, `.price`, `.onsale`) to isolate items.

### Active Target Array
* **Barrel House Cellars** (Platform: WooCommerce)
    * Target: `https://barrelhousecellars.com/product-category/spirits/rum/?orderby=price-desc`
* **De Vine Cellars** (Platform: WooCommerce)
    * Target: `https://devinecellars.com.au/shop/spirits/rum/?orderby=price-desc`
* **Liquor Mojo** (Platform: WooCommerce)
    * Target: `https://www.liquormojo.com.au/category/rum/?orderby=price-desc`
* **Rum Club Australia** (Platform: WordPress Custom Grid)
    * Target: `https://www.rumclubaustralia.com.au/bottle-shop/`
* **Rum Tribe** (Platform: WooCommerce Core Layout)
    * Target: `https://rumtribe.com.au/shop/`
* **Casa de Vinos** (Platform: Maropost / Neto E-commerce Engine)
    * Target: `https://www.casadevinos.com.au/all-rum-rhum/?rf=&sortby=highest_price`

---

## Strategy C: Headless Browser JS Rendering
**Programmatic Route:** `Strategy_Headless_JS`
**Execution Overhead:** Moderate (Requires routing calls through the local Docker `browserless/chrome` container).
**Exploitation Method:** Used for legacy setups, multi-parameter search structures, or client-side application platforms where data elements are completely invisible within the raw source HTML.

### Active Target Array
* **Copper and Oak** (Platform: ASP.NET Enterprise Frame)
    * Target: `https://www.copperandoak.com.au/products/spirits/rum/default.aspx#pageSize=48`
    * *Note: Heavy client-side hashing parameters require fully rendered engine state capture.*
* **Cheapest Liquor** (Custom SPA Architecture)
    * Target: `https://cheapestliquor.com.au/rum?sort=high`
* **Nicks (Wine Merchants)** (Advanced Enterprise Platform Layout)
    * Target: `https://www.nicks.com.au/categories/spirits-liqueurs/rum-cachaca?sort=priceDesc`
* **World Wine** (Search-Driven Client Result Block)
    * Target: `https://worldwine.com.au/search?q=rhum%20rum&sort=price%2Cdesc`