# Technical Implementation Specification (`SPEC.md`)

## 1. Architectural Blueprint & Technical Stack

This specification defines the backend execution logic, data transformation pipelines, database models, and orchestration patterns for **Rum Hunter**. The architecture is optimised for a strict $0 software licensing footprint, leveraging a hybrid configuration of local hardware and secure cloud-native serverless systems.

### Component Map

* **Orchestration Engine:** n8n self-hosted via Docker Compose on a local Network Attached Storage (NAS) device. Handles cron scheduling, webhook listening, external HTTP routing, and database state synchronisation.
* **Headless Rendering Cluster:** Browserless/Chrome standalone container running locally on the NAS, exposed to n8n via a local network port. Used strictly for rendering JavaScript-heavy client-side SPAs.
* **Spirits Data Extraction Engine:** Local Large Language Model (LLM) running on a local workstation equipped with a 12GB VRAM GPU. Exposed via a secure local network API endpoint (Ollama/Open-WebUI wrapper).
* **Persistence & API Gateway:** Supabase Free Tier. Provides a cloud-hosted PostgreSQL instance and instantly auto-generates native, secure, authenticated JSON REST APIs for all relational entities, removing the requirement for legacy database drivers or raw connection strings.
* **Binary Media Repository:** Supabase Storage Buckets (1GB Free Tier), hosting transparent alpha-channel product images.
* **Production UI Layer:** Next.js or Nuxt static-site framework compiled and served via the Vercel Free Tier. Communicates securely with Supabase via client-side JSON REST API service calls.

---

## 2. Complete Database Architecture (Supabase PostgreSQL DDL)

Execute the following relational schema definition inside the Supabase SQL Editor. This script establishes the four core tables, performance indices, optimisation layers, and automated event triggers required for historical tracking and data isolation.

```sql
-- Enable cryptographic and programmatic UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Drop existing assets to ensure a clean state during instantiation
DROP TRIGGER IF EXISTS trigger_archive_price_mutation ON retail_listings;
DROP FUNCTION IF EXISTS archive_price_mutation();
DROP FUNCTION IF EXISTS check_historical_best(UUID, NUMERIC);
DROP TABLE IF EXISTS price_history CASCADE;
DROP TABLE IF EXISTS retail_listings CASCADE;
DROP TABLE IF EXISTS master_products CASCADE;
DROP TABLE IF EXISTS site_registry CASCADE;

-- Table 1: Master Products Catalog (Normalised Global Reference)
CREATE TABLE master_products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    brand_producer VARCHAR(255) NOT NULL,
    expression_name VARCHAR(255) NOT NULL,
    origin_country VARCHAR(100) NOT NULL,
    size_ml INTEGER NOT NULL,
    abv_percentage NUMERIC(4, 2) NOT NULL,
    age_statement VARCHAR(50) DEFAULT 'NAS',
    image_url_local TEXT,
    description TEXT,
    description_locked BOOLEAN DEFAULT FALSE,
    description_revisions INTEGER DEFAULT 0,
    aliases TEXT[] DEFAULT '{}'::TEXT[],
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Table 2: Active Retail Listings (Current Point of Sale per Merchant)
CREATE TABLE retail_listings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    master_product_id UUID REFERENCES master_products(id) ON DELETE CASCADE,
    store_name VARCHAR(155) NOT NULL,
    store_url TEXT NOT NULL UNIQUE,
    current_price NUMERIC(10, 2) NOT NULL,
    in_stock BOOLEAN DEFAULT TRUE,
    special_notes TEXT,
    last_scraped_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Table 3: Historical Price Archive (Append-Only Event Stream)
CREATE TABLE price_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    retail_listing_id UUID REFERENCES retail_listings(id) ON DELETE CASCADE,
    price NUMERIC(10, 2) NOT NULL,
    is_special BOOLEAN DEFAULT FALSE,
    recorded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Table 4: Scrape Target Control Registry (Dynamic Orchestration Drive)
CREATE TABLE site_registry (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    store_name VARCHAR(155) NOT NULL UNIQUE,
    strategy_type VARCHAR(50) NOT NULL CHECK (strategy_type IN ('Strategy_JSON_Feed', 'Strategy_HTML_DOM', 'Strategy_Headless_JS')),
    target_urls TEXT[] NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    scrape_interval_hours INTEGER DEFAULT 24,
    queue_urgent BOOLEAN DEFAULT FALSE,
    last_scraped_at TIMESTAMP WITH TIME ZONE
);

-- Performance and High-Concurrency API Query Indices
CREATE INDEX idx_master_products_aliases ON master_products USING gin (aliases);
CREATE INDEX idx_retail_listings_master ON retail_listings(master_product_id);
CREATE INDEX idx_retail_listings_lookup ON retail_listings(store_name, in_stock);
CREATE INDEX idx_price_history_listing_date ON price_history(retail_listing_id, recorded_at DESC);

-- Automated Event Trigger: Append historical entries only upon genuine price changes
CREATE OR REPLACE FUNCTION archive_price_mutation()
RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'INSERT') OR (OLD.current_price IS DISTINCT FROM NEW.current_price) OR (OLD.in_stock IS DISTINCT FROM NEW.in_stock) THEN
        INSERT INTO price_history (retail_listing_id, price, is_special, recorded_at)
        VALUES (
            NEW.id, 
            NEW.current_price, 
            (NEW.special_notes IS NOT NULL AND NEW.special_notes <> ''), 
            NOW()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_archive_price_mutation
AFTER INSERT OR UPDATE ON retail_listings
FOR EACH ROW
EXECUTE FUNCTION archive_price_mutation();

-- Analytics Optimisation Function: Computes if a listing price is a historical minimum
CREATE OR REPLACE FUNCTION check_historical_best(target_listing_id UUID, validation_price NUMERIC)
RETURNS BOOLEAN AS $$
DECLARE
    lowest_archived_price NUMERIC;
BEGIN
    SELECT MIN(price) INTO lowest_archived_price
    FROM price_history
    WHERE retail_listing_id = target_listing_id;
    
    IF lowest_archived_price IS NULL THEN
        RETURN TRUE;
    END IF;
    
    RETURN validation_price <= lowest_archived_price;
END;
$$ LANGUAGE plpgsql;

```

---

## 3. Data Ingestion Edge Processing

To combat format fragmentation, local hardware constraints, and parsing vulnerabilities, n8n utilises specialised inline programmatic guards.

### A. The Post-LLM JavaScript Sanitiser Node

Local LLMs regularly append conversational prose or wrap structured data inside Markdown text blocks. This JavaScript block must execute immediately after the LLM extraction step to sanitise the payload before it reaches database injection states.

```javascript
/**
 * Post-LLM Structural JSON Sanitiser and Safety Guard
 * Context: Processes raw text string outputs from local 12GB VRAM models.
 */
let rawText = $input.item.json.output || "";

// Step 1: Strip out all markdown syntax blocks and trailing whitespace
rawText = rawText.replace(/```json/gi, '').replace(/```/g, '').trim();

// Step 2: Establish array/object starting constraints
const braceStartPos = rawText.indexOf('{');
const braceEndPos = rawText.lastIndexOf('}');

if (braceStartPos !== -1 && braceEndPos !== -1) {
    rawText = rawText.substring(braceStartPos, braceEndPos + 1);
} else {
    return {
        json: {
            parsing_failed: true,
            error_reason: "Malformed structural string: Structural curly braces missing.",
            raw_payload: $input.item.json.output
        }
    };
}

try {
    // Step 3: Validate text parse into genuine JSON syntax
    const verifiedObject = JSON.parse(rawText);
    return { json: verifiedObject };
} catch (jsonException) {
    return { 
        json: { 
            parsing_failed: true, 
            error_reason: `JSON syntax violation: ${jsonException.message}`,
            raw_payload: $input.item.json.output
        } 
    };
}

```

### B. Image Translucency & Extraction Node

When downloading raw binary images from mixed merchant sources, n8n runs an edge background removal function using standard canvas/matrix operations. If a clean white boundary (`#FFFFFF`) or high-luminance studio background is verified, it is programmatically converted to an alpha-transparent channel. If a complex composition is detected, the workflow automatically routes the image file to a free-tier background-removal API endpoint before writing the final `.png` file to the Supabase storage vault.

---

## 4. Multi-Tiered Normalisation & Alias Engine

To prevent multiple instances of identical expressions from fragmenting your catalogue, data is routed through a rigid 3-Tier Normalisation Engine:

1. **Tier 1 (Database Index Matching):** n8n queries `master_products`. It checks if the incoming scraped product title exactly matches a registered `expression_name` OR matches any string stored within the `aliases` text array. If a match is verified, it bypasses the LLM entirely, instantly updating `retail_listings` (0% VRAM overhead).
2. **Tier 2 (LLM Deductive Parsing):** If Tier 1 fails, the text string, merchant data, and an array of existing candidate product names are sent to the local LLM via **System Prompt A**. If the LLM successfully maps the item to an existing product with a confidence rating exceeding 85%, the system completes the binding and pushes the new variant name into the master record's `aliases` array.
3. **Tier 3 (Human-in-the-Loop Triage):** If the confidence score drops below 85%, the workflow writes the raw payload to an isolated staging queue. When an administrator manually hooks the item to a master entry in the Admin UI workspace, the custom text string is appended to the master `aliases` array, ensuring the system processes that string automatically on all subsequent nights.

---

## 5. Machine Learning Core Configurations (System Prompts)

### Prompt A: Technical Normalisation & Candidate Resolution

```text
[CONTEXT DEFINITION]
You are an expert spirit catalog automation tool running inside a data normalisation pipeline. Your task is to dissect raw web-scraped text payloads collected from Australian liquor merchants and transform them into a standardized schema object.

[METRIC & SPELLING LAW]
- Enforce the metric system strictly. All capacities must be parsed into absolute integers representing millilitres (ml) (e.g., "700ml" -> 700, "70 CL" -> 700, "1.75 Litre" -> 1750, "700ml Pack of 2" -> 1400).
- Extract the Alcohol By Volume (ABV) as a numeric float value. Strip any percentage symbols (e.g., "46.3% ALC/VOL" -> 46.30).
- Utilise Australian English text layouts where applicable (e.g., normalise, flavour).

[THE MULTI-SKU BUNDLE CONSTRAINT]
If the raw payload mentions "gift pack", "bundle", "multi-pack", "case", or "3-pack", you must treat this as an independent retail SKU configuration. It must never be mapped to a standard single-bottle master catalog entry.

[CANDIDATE MATRIX EVALUATION]
You will be provided with a reference list of existing database profiles format: [{"id": "UUID", "name": "Text", "aliases": ["Text"]}]
- Evaluate if the incoming merchant string refers to an item already present.
- If it is a match, output the matching UUID in the 'matched_master_id' field.
- If it represents an entirely new product entry, output null.

[OUTPUT FORMAT REGULATION]
Output strictly a valid, raw, un-nested JSON schema block. Do not add explanations, conversational tokens, or markdown wrappers.

{
  "brand_producer": "Normalized Brand Title Text",
  "expression_name": "Normalized Expression Specifics Text",
  "origin_country": "Country Text or 'Unknown'",
  "size_ml": 700,
  "abv_percentage": 40.00,
  "age_statement": "Age Text or 'NAS'",
  "is_bundle": true/false,
  "matched_master_id": "UUID_STRING_OR_NULL",
  "confidence_score": 0.00
}

```

### Prompt B: Content Up-Cycling & Verification

```text
[CONTEXT DEFINITION]
You are a concise, analytical spirit commentator. Your job is to update and polish the narrative overview field for a Master Product profile.

[EVALUATION CHECKLIST CRITERIA]
To achieve a production-ready verification state, the profile text must satisfy the following checks:
1. Has_Flavour_Notes: True/False (Must identify concrete nose, palate, or structural finish elements).
2. Has_Age_Statement: True/False (Must explicitly extract maturity indicators or record it as a Non-Age-Statement product).
3. Composition Bounds: The text length must be greater than 30 words and less than 60 words.

[UP-CYCLING INCREMENT MATRIX]
Analyze the 'current_description' string and compare it with the incoming 'scraped_merchant_text'. Merge any new production facts, distillery history, or verified tasting points into a single polished summary block. Increment the revision tracker. If the target checklist requirements are met, flag the system to freeze the asset.

[OUTPUT FORMAT REGULATION]
Output strictly a valid JSON string. Do not wrap in markdown fences.
{
  "updated_description": "Polished text string meeting all criteria",
  "has_flavour_notes": true/false,
  "has_age_statement": true/false,
  "word_count": 0,
  "trigger_lock_state": true/false
}

```

---

## 6. Functional Execution Edge Cases

### A. The Launch Avalanche Mitigation (Cold-Start Rule)

* **Problem:** Bootstrapping 26 stores simultaneously on Day 1 will overwhelm a 12GB VRAM local workstation, forcing hundreds of items into manual triage.
* **Resolution:** Implement an **Anchor Seeding Loop**. The execution runtime must be restricted to **Strategy A (Shopify JSON)** for your top two highly structured merchants (e.g., *Spirits of France* and *Barrel and Batch*) during the initialization phase. This populates a solid baseline catalog of ~75% of active Australian rum products. When you enable remaining merchants, an orchestration fallback rule will automatically approve new master profiles without manual HIL intervention if the LLM's independent validation confidence score registers above 85%.

### B. Obscure Cart-Level Modifiers

* **Problem:** Merchant-wide promotions (e.g., "15% off all items when you spend $150") are completely invisible when parsing individual product card components.
* **Resolution:** The parsing wrapper checks for high-level promotional layout modules or site-wide banner classes. If identified, the exact phrase is captured as a global string and injected down into the `special_notes` table property of every individual listing collected during that run. The frontend UI layers then parse and display this text as a promotional badge, protecting the core numeric sorting engines from calculation errors.

---

## 7. Security Layer & Row-Level Security (Supabase RLS)

Because the frontend application communicates directly with Supabase via public JSON service endpoints, your data tables must be protected against malicious write, update, or deletion operations.

```sql
-- 1. Enforce Row-Level Security across all relational entities
ALTER TABLE master_products ENABLE ROW LEVEL SECURITY;
ALTER TABLE retail_listings ENABLE ROW LEVEL SECURITY;
ALTER TABLE price_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE site_registry ENABLE ROW LEVEL SECURITY;

-- 2. Establish Public Read-Only Policies (Anon Access Keys)
CREATE POLICY "Allow public read access to catalog" 
ON master_products FOR SELECT USING (true);

CREATE POLICY "Allow public read access to active listings" 
ON retail_listings FOR SELECT USING (true);

CREATE POLICY "Allow public read access to price trends" 
ON price_history FOR SELECT USING (true);

-- 3. Restrict Admin write paths (Requires service_role authentication keys)
CREATE POLICY "Restrict product mutations to service role" 
ON master_products FOR ALL TO service_role USING (true);

CREATE POLICY "Restrict listing mutations to service role" 
ON retail_listings FOR ALL TO service_role USING (true);

CREATE POLICY "Restrict registry mutations to service role" 
ON site_registry FOR ALL TO service_role USING (true);

```

---

## 8. Error Quarantine & Logging Architecture

To prevent failed scrapes or malformed LLM data outputs from disappearing when n8n's internal execution logs expire, anomalies are written to a persistent triage log.

```sql
-- Table 5: Ingestion Operational Error Logs
CREATE TABLE scraping_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    store_name VARCHAR(155) NOT NULL,
    error_type VARCHAR(100) NOT NULL, -- e.g., 'LLM_Parsing_Violation', 'HTTP_Timeout'
    raw_payload TEXT,
    resolved BOOLEAN DEFAULT FALSE,
    recorded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_scraping_logs_status ON scraping_logs(resolved) WHERE resolved = FALSE;

```

---

## 9. Structural Schema Updates (Stock State Transitions & Decay)

To handle the "Back in Stock" calculation alongside its 72-hour visibility decay window, update the core tables with explicit state timestamps.

```sql
-- Append state tracking columns to relational layers
ALTER TABLE retail_listings ADD COLUMN oos_since TIMESTAMP WITH TIME ZONE;
ALTER TABLE master_products ADD COLUMN restocked_at TIMESTAMP WITH TIME ZONE;

-- Refine the automated mutation trigger function to track timestamps
CREATE OR REPLACE FUNCTION archive_price_mutation()
RETURNS TRIGGER AS $$
BEGIN
    -- Detect Transition: Active to Out of Stock
    IF (NEW.in_stock = FALSE AND (OLD.in_stock = TRUE OR OLD.in_stock IS NULL)) THEN
        NEW.oos_since := NOW();
    END IF;

    -- Detect Transition: Out of Stock to Available (Restock Event)
    IF (NEW.in_stock = TRUE AND OLD.in_stock = FALSE) THEN
        NEW.oos_since := NULL;
        -- Update the Master Product record to flag a global catalog restock event
        UPDATE master_products 
        SET restocked_at = NOW() 
        WHERE id = NEW.master_product_id;
    END IF;

    -- Append entries to historical logs upon any price or stock volatility
    IF (TG_OP = 'INSERT') OR (OLD.current_price IS DISTINCT FROM NEW.current_price) OR (OLD.in_stock IS DISTINCT FROM NEW.in_stock) THEN
        INSERT INTO price_history (retail_listing_id, price, is_special, recorded_at)
        VALUES (
            NEW.id, 
            NEW.current_price, 
            (NEW.special_notes IS NOT NULL AND NEW.special_notes <> ''), 
            NOW()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

```

### Frontend Badge Execution Rules (For Stitch & Antigravity)

* **The "Back in Stock" Condition:** The frontend component renders the badge **only** if `master_products.restocked_at` is populated **AND** `NOW() - master_products.restocked_at` is less than or equal to **72 hours**. Once the 72-hour window lapses, the badge decays and disappears from the grid card automatically.

---

## 10. Politeness Throttle & Anti-Bot Infrastructure

To protect smaller boutique Australian distilleries and independent rum club servers from being overwhelmed by multi-threaded extraction sweeps, n8n must strictly enforce sequential execution throttles.

* **The Wait Configuration:** Every workflow routing through `Strategy_HTML_DOM` or `Strategy_Headless_JS` must wrap its target URL array inside a sequential **Split-In-Batches loop** (concurrency = 1).
* **The Delay Element:** Immediately following the data extraction node for a URL, an n8n **Wait Node** must execute a mandatory, randomised sleep timer between **2000ms and 5000ms** before iterating to the next item in the cluster array. This breaks up predictable patterns and prevents domestic IP blacklisting.

### C. Request Header Spoofing Rules (Anti-Fingerprinting)

Every direct HTTP request executed within `Strategy_JSON_Feed` and `Strategy_HTML_DOM` workflows must suppress default automation footprints by explicitly defining a standardised browser header object.

```json
{
  "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36",
  "Accept": "application/json, text/html, application/xhtml+xml, */*",
  "Accept-Language": "en-AU,en;q=0.9",
  "Cache-Control": "no-cache",
  "Pragma": "no-cache"
}

```

* **Dynamic Referer Rotation:** The n8n loop must dynamically set the `Referer` header to match the root origin domain of the specific site record being crawled during that cycle step.

---

## 11. Stale Inventory Cleanup & Variant Filtering

### A. Automated Stale Registry Sweep

To prevent deleted merchant pages from remaining active as "Ghost Listings", every orchestration pipeline must execute a post-scrape cleanup query. The moment an ingestion queue loop terminates for a specific `store_name`, n8n triggers an immediate RPC or SQL update command via the Supabase REST API gateway.

The 2-hour interval acts as a safety buffer covering the duration of the active scrape session, ensuring only items missed during the current run are marked out of stock.

```sql
-- Force obsolete or missing listings to an out-of-stock state
UPDATE retail_listings
SET in_stock = FALSE, oos_since = NOW()
WHERE store_name = 'TARGET_STORE_NAME'
AND last_scraped_at < NOW() - INTERVAL '2 hours';

```

### B. Shopify Nested Variant Isolation

When processing payloads within `Strategy_JSON_Feed`, the ingestion script must not assume a 1:1 relationship between a top-level JSON product object and a single bottle size.

* The workflow must map a nested iterative loop over the `product.variants` array.
* The script must evaluate the variant's capacity metrics (via fields like `variant.title`, `variant.sku`, or `variant.grams`) to ensure pricing is strictly bound to the corresponding volumetric Master Product (e.g., isolating 700ml options and discarding sample packs, 200ml tasters, or merchandise).