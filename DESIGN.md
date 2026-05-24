# Design System & UI/UX Specification (`DESIGN.md`)

## 1. Core Visual Principles & Design Tokens

This application must feel premium, data-dense, and highly scannable. It is an analytics engine masquerading as a retail directory. Avoid bloated whitespace; prioritize clarity at a glance.

### A. Colour Palette (Dark Mode First)
* **Surface Foundation:** Deep obsidian/charcoal spectrum (e.g., Background: `#0B0C10`, Card Surface: `#1F2833`).
* **Primary Accent:** Warm, rich rum amber (e.g., `#C5A059` or `#D4AF37`) used for highlights, primary call-to-actions, and interactive states.
* **Data Status Indicators:**
    * `Special / Promo`: Vibrant emerald green (`#2ECC71`).
    * `New Product`: Deep electric blue (`#3498DB`).
    * `Out of Stock / Dimmed`: Muted slate grey (`#7F8C8D`).
    * `Historical Best`: Intense amber/orange fire gradient (`#E67E22` to `#D35400`).

### B. Typography & Density
* **Scale:** Clean sans-serif hierarchy (system-ui standard) optimized for compact tabular data and numerals.
* **Density:** Tight component padding to ensure a desktop user can see at least 8 to 12 product cards above the fold without scrolling.

---

## 2. Global Component Anatomy

### A. The Master Rum Card
The fundamental building block of the primary user interface.

```
+---------------------------------------+
| [Badge: NEW] [Badge: SPECIAL]         |
|                                       |
|               IMAGE                   |
|       (Transparent Bottle Asset)      |
|                                       |
+---------------------------------------+
| BRAND / PRODUCER                      |
| Expression Name                       |
| Country of Origin | 700ml | 42.5% ABV |
+---------------------------------------+
| FROM $79.99                           |
| Price Range: $79.99 - $95.00          |
+---------------------------------------+
```


* **Image Handling:** Rendered within a fixed-aspect container. Must display your custom processed transparent alpha-channel `.png` assets cleanly against card backgrounds.
* **Card Badging Rules:**
    * `New`: Injected if the product profile is less than 14 days old.
    * `Back in Stock`: Injected if a product returns after a complete inventory hiatus.
    * `Special`: Injected if any linked store has a live promotion.
    * `Out of Stock`: If active across all stores, the entire card opacity drops to 60%, active prices are hidden, and it displays `"Out of Stock (Last Seen $XX.XX)"`.

---

## 3. Detailed Screen Specification

### Screen 1: Main User Discovery Grid
The entry point for standard users. Focuses on immediate tracking status and rapid filtration.

* **Layout Structure:**
    1.  **Top Navigation Utility Header:** Contains the application branding, high-level ecosystem metrics (`Total Expressions Tracked`, `Active Specials Today`), and an global search input.
    2.  **Filter Toolbar:** A horizontal row of compact pill-buttons for rapid data slicing:
        * `In Stock Only` (Toggle Switch)
        * `Specials Only` (Toggle Switch)
        * `Country` (Dropdown multiselect)
        * `ABV Range` (Dropdown selectors)
    3.  **The Infinite Product Grid:** A responsive grid displaying the Master Rum Cards (minimum 4 columns on desktop, 2 columns on mobile).

* **Component Interaction:**
    * Hovering over a card subtly scales the container and intensifies the amber accent border.
    * Clicking any card executes an asynchronous pull request to fetch deep listings data and slides out the Product Details Drawer from the right viewport boundary.

---

### Screen 2: Product Details (Slide-in Drawer Layout)
Provides absolute market transparency regarding a specific expression without forcing the user to leave their current scroll position on the main grid.

* **Layout Structure:**
    1.  **Hero Metadata Header:** Displays an enlarged view of the verified transparent bottle asset side-by-side with the fully normalised spirit specifications (Brand, Name, Exact ABV, Size, Age Statement, Country).
    2.  **The "Rum Hunter" Narrative Block:** Displays the automated, up-cycled description summary crafted by your local LLM (tasting notes, history, and production style).
    3.  **Marketplace Directory Table:** An explicit directory tracking where to source the bottle within Australia.
    4.  **Price Trend Charting Workspace:** A dedicated zone rendering a historical timeline graph of the item's price fluctuations over the past 6 months.

* **The Marketplace Directory Sorting Law:**
    * Rows **must** sort by absolute lowest numeric price first, completely ignoring stock availability.
    * If a retail site is out of stock, the text remains in position but dims to slate grey with a clear `Out of Stock` tag. This lets users accurately identify where the absolute best deal *normally* sits.
    * Each row contains a clean direct-link outbound button: `Visit Store ↗`.

* **The Analytics Engine Badge Trigger:**
    * If the lowest active price in the directory table matches the absolute historical minimum value recorded in the archival table database, a prominent `🔥 Historical Best Price` indicator is injected above the pricing table.

---

### Screen 3: Admin Triage Workspace ("Fix Me" Queue)
The operational command center where you handle low-confidence matches flagged by the local LLM normalisation engine.

* **Layout Structure:**
    * **Dual-Pane Split Layout (50% / 50% Desktop Viewport):**
    * **Left Pane (The Raw Payload Input):** Displays the raw un-sanitised scrapings captured by n8n. Includes raw scrape text, unparsed strings, source retail URL, current listed price, and the underlying failure flag (e.g., `Confidence Score: 42% - Low Matching Threshold`).
    * **Right Pane (The Resolution Control Panel):**
        * **Action Module A (Manual Association):** Features a prominent autocomplete search box connected directly to your live `Master_Products` table. Typing filters your entire catalog instantly. Selecting an item and clicking `Bind & Learn Alias` resolves the queue item, updates current pricing tables, and pushes the raw string into that Master Product's alias array.
        * **Action Module B (New Fork Generation):** A secondary button labeled `Draft New Master Product`. Clicking this passes the raw payload back to the local machine to draft a fresh profile, transforming the interface into a pre-populated form where you can verify fields before saving it to the live database.

---

### Screen 4: Admin Site Registry Controller
The administrative command cockpit used to configure scrape cadences, track source health, and expand target pipelines.

* **Layout Structure:**
    1.  **System Health Metric Ribbon:** High-level system indicators showing `Total Tracked Retailers`, `Failed Scrapes (24h)`, and `Pending Queue Items`.
    2.  **The Registry Management Data-Grid:** A dense tabular array parsing your `Site_Registry` table:
        * `Store Identifier`: Text label with direct link.
        * `Target Clusters`: An expander showing the array of hardlinked URLs mapped to this specific task profile (e.g., Jimmy's Rums 6 pages).
        * `Scrape Interval Selector`: Inline dropdown (`24h`, `48h`, `72h`, `Weekly`) updating backend cron rules instantly.
        * `Status Toggle`: An inline iOS-style toggle switch (`Active` / `Inactive`) to halt operations instantly.
        * `Action Core`: A secondary icon button labeled `Queue Quick Scrape` that safely flags the row for prioritized asynchronous background retrieval.
    3.  **The Pipeline Expansion Form:** A simple expandable form layout at the foot of the page allowing you to input a new store name, select an extraction strategy profile (JSON Feed vs HTML DOM vs Headless Javascript), and input an initial array of source URLs.

---

## 4. Universal Application States

Stitch must design clear UI states for the following conditions across all screens:
* **Loading State:** Custom skeleton components matching the exact dimensional layout of cards/tables. Avoid generic spinning loaders.
* **Empty State:** Clean, centered typography blocks with an amber accent action button (e.g., "Clear Filters" or "Add New Target URL").
*