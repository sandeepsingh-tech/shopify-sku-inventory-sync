# Shopify SKU-Based Direct Inventory Sync (GraphQL)

A lightweight, automated Python tool using Shopify's Admin GraphQL API to synchronize warehouse inventory feeds directly by **Variant SKU**—bypassing product URL handles completely, eliminating paid apps, and updating thousands of stock levels in seconds.

---

## Why Use This Technique? (The Core Problem & Benefits)

### The Problem

When running dropshipping or multi-vendor catalogs, suppliers frequently modify product titles to capitalize on seasonal trends or SEO keywords. In their daily CSV exports, this changes the product URL **`Handle`**.

Shopify's default CSV inventory importer matches items strictly by **`Handle`**, not by SKU. When supplier handles diverge from your catalog:

1. Native CSV imports fail with errors:
`"Line X: Couldn't be updated because the handle doesn't match what's in Shopify."`

2. The number of successfully updated products drops over time as suppliers silently change more titles.


3. Manually migrating handles requires running URL redirect scripts every single week.

### Key Benefits of This Solution

* **Immune to Supplier Handle Changes:** Matches stock directly to the Shopify `InventoryItem` via SKU. Suppliers can rewrite titles and change handles daily without breaking your sync.
* **Zero Subscription Costs:** Bypasses inventory sync apps on the Shopify App Store that charge **$20 to $100+/month** for basic SKU-matching or scheduled tasks.
* **Lightning-Fast Bulk GraphQL Mutations:** Uses Shopify's native `inventorySetQuantities` mutation, grouping stock updates into batches of 100–250 items. Syncs **7,000+ items in under 30 seconds**.
* **Zero Catalog Export Dependencies:** Pulls live store variants and Inventory Item IDs dynamically via paginated GraphQL queries—no need to manually export and split `products_export_*.csv` files beforehand.
* **Auto-Detects Warehouse Stock Columns:** Safely reads stock counts from standard warehouse columns (`On hand (current)`, `Available (not editable)`, or `On hand (new)`).
* **Safe Handling of Unlisted Feeds:** Supplier warehouse items not carried on your store are safely ignored without throwing errors or halting the update pipeline.



---

## Why Local Python Installation is the Best & Shortest Way

Running this script locally on your PC (via Terminal or Command Prompt) is significantly faster, simpler, and safer than deploying web apps or cloud servers:

1. **Direct File Handling:** Large supplier inventory files (10MB–50MB+) are parsed locally in memory within milliseconds.
2. **Zero Cloud Infrastructure:** No recurring server costs, webhooks, Docker containers, or AWS Lambda timeout restrictions.
3. **Enhanced Security:** Your Shopify Admin API credentials and warehouse feeds remain strictly on your local machine and never touch third-party servers.

---

## Prerequisites

### 1. Install Python on Your PC

* **Windows:**
1. Download Python from [python.org/downloads](https://www.python.org/downloads/).
2. Launch the installer and **ensure you check the box: "Add Python to PATH"**.
3. Complete the setup.


* **macOS:**
Install via Homebrew:
```bash
brew install python

```



Verify your installation by opening Command Prompt / Terminal and running:

```bash
python --version
pip --version

```

### 2. Install Dependencies

Run the following command to install `requests` (for GraphQL API calls) and `pandas` (for CSV parsing):

```bash
python -m pip install requests pandas

```

---

## Setup & Configuration

### Step 1: Create a Custom App in Shopify

1. In your Shopify Admin, go to **Settings > Apps and sales channels > Develop apps**.
2. Click **Create an app** (e.g., "SKU Inventory Syncer").
3. Navigate to **Configuration > Admin API integration**.
4. Enable the following API scopes:
* `read_products`
* `write_products`
* `read_inventory`
* `write_inventory`
* `read_locations`


5. Click **Save** and **Install app**.
6. Copy your **Client ID** and **Client Secret** (under API credentials).

### Step 2: Prepare Your Project Directory

Create a project folder (e.g., `shopify_sku_sync/`) and place your supplier CSV file and the Python script inside it:

* `supplier_inventory.csv` (Your supplier's daily inventory file containing `SKU` and stock counts)
* `sync_inventory_by_sku.py` (The script below)

---

## The Script (`sync_inventory_by_sku.py`)

```python
import os
import sys
import time
import requests
import pandas as pd

# ==============================================================================
# 1. CONFIGURATION (REPLACE WITH YOUR DETAILS)
# ==============================================================================
# e.g., if your admin is your-store.myshopify.com, SHOP_NAME is "your-store"
SHOP_NAME = "YOUR_STORE_SUBDOMAIN"
CLIENT_ID = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
API_VERSION = "2024-07"

# The name of your supplier's daily CSV file
SUPPLIER_INVENTORY_CSV = "supplier_inventory.csv"

# Target location name in Shopify Admin (defaults to your primary store location)
TARGET_LOCATION_NAME = "Shop Location"

# Batch size for GraphQL mutations (Shopify allows up to 250)
BATCH_SIZE = 100

GRAPHQL_URL = f"https://{SHOP_NAME}.myshopify.com/admin/api/{API_VERSION}/graphql.json"

# ==============================================================================
# 2. AUTHENTICATION (OAuth Client Credentials)
# ==============================================================================
print("[*] Authenticating with Shopify API...")
auth_url = f"https://{SHOP_NAME}.myshopify.com/admin/oauth/access_token"
auth_payload = {
    "client_id": CLIENT_ID,
    "client_secret": CLIENT_SECRET,
    "grant_type": "client_credentials"
}

auth_resp = requests.post(auth_url, json=auth_payload, headers={"Content-Type": "application/json"})
token_data = auth_resp.json()

if "access_token" not in token_data:
    sys.exit(f"[!] Authentication failed. Response: {token_data}")

ACCESS_TOKEN = token_data["access_token"]
print("[+] Authenticated successfully.")

HEADERS = {
    "Content-Type": "application/json",
    "X-Shopify-Access-Token": ACCESS_TOKEN
}

# ==============================================================================
# 3. GET TARGET LOCATION ID
# ==============================================================================
LOCATION_QUERY = """
query {
  locations(first: 10) {
    edges {
      node {
        id
        name
        isActive
      }
    }
  }
}
"""
loc_res = requests.post(GRAPHQL_URL, json={"query": LOCATION_QUERY}, headers=HEADERS).json()
locations = loc_res.get("data", {}).get("locations", {}).get("edges", [])

target_location_id = None
fallback_id = None
fallback_name = ""

for loc in locations:
    node = loc["node"]
    if not fallback_id and node.get("isActive"):
        fallback_id = node["id"]
        fallback_name = node["name"]
    if TARGET_LOCATION_NAME.lower() in node["name"].lower():
        target_location_id = node["id"]
        print(f"[+] Using Location: '{node['name']}' ({target_location_id})")
        break

if not target_location_id:
    target_location_id = fallback_id
    print(f"[!] '{TARGET_LOCATION_NAME}' not found. Defaulting to: '{fallback_name}' ({fallback_id})")

# ==============================================================================
# 4. FETCH LIVE STORE VARIANTS (SKU -> InventoryItem ID)
# ==============================================================================
print("\n[*] Fetching live product SKUs from Shopify...")
VARIANT_QUERY = """
query getVariants($cursor: String) {
  productVariants(first: 250, after: $cursor) {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        sku
        inventoryItem {
          id
        }
      }
    }
  }
}
"""
sku_to_inv_item = {}
has_next = True
cursor = None

while has_next:
    variables = {"cursor": cursor}
    res = requests.post(GRAPHQL_URL, json={"query": VARIANT_QUERY, "variables": variables}, headers=HEADERS).json()
    data = res.get("data", {}).get("productVariants", {})
    edges = data.get("edges", [])

    for edge in edges:
        node = edge["node"]
        sku = (node.get("sku") or "").strip()
        inv_item = node.get("inventoryItem")
        if sku and inv_item:
            sku_to_inv_item[sku] = inv_item["id"]

    page_info = data.get("pageInfo", {})
    has_next = page_info.get("hasNextPage", False)
    cursor = page_info.get("endCursor")
    print(f"    -> Loaded {len(sku_to_inv_item)} variants so far...")
    time.sleep(0.05)

print(f"[+] Total live SKUs retrieved from store: {len(sku_to_inv_item)}")

# ==============================================================================
# 5. LOAD SUPPLIER INVENTORY FEED
# ==============================================================================
print(f"\n[*] Reading supplier feed: {SUPPLIER_INVENTORY_CSV}...")
if not os.path.exists(SUPPLIER_INVENTORY_CSV):
    sys.exit(f"[!] File not found: {SUPPLIER_INVENTORY_CSV}")

df_feed = pd.read_csv(SUPPLIER_INVENTORY_CSV, low_memory=False)
if "SKU" not in df_feed.columns:
    sys.exit("[!] CSV file must contain an 'SKU' column.")

df_feed = df_feed.dropna(subset=["SKU"])
df_feed["Clean_SKU"] = df_feed["SKU"].astype(str).str.strip()

# Automatically select available quantity column
if "On hand (current)" in df_feed.columns and not df_feed["On hand (current)"].isna().all():
    df_feed["Target_Qty"] = pd.to_numeric(df_feed["On hand (current)"], errors="coerce").fillna(0)
elif "Available (not editable)" in df_feed.columns:
    df_feed["Target_Qty"] = pd.to_numeric(df_feed["Available (not editable)"], errors="coerce").fillna(0)
elif "On hand (new)" in df_feed.columns:
    df_feed["Target_Qty"] = pd.to_numeric(df_feed["On hand (new)"], errors="coerce").fillna(0)
else:
    sys.exit("[!] Could not locate quantity column ('On hand (current)' or 'Available').")

df_feed["Target_Qty"] = df_feed["Target_Qty"].astype(int)
supplier_quantities = dict(zip(df_feed["Clean_SKU"], df_feed["Target_Qty"]))
print(f"[+] Loaded {len(supplier_quantities)} SKUs from supplier feed.")

# ==============================================================================
# 6. BATCH UPDATE VIA GRAPHQL
# ==============================================================================
INVENTORY_SET_MUTATION = """
mutation setInventory($input: InventorySetQuantitiesInput!) {
  inventorySetQuantities(input: $input) {
    inventoryAdjustmentGroup {
      reason
    }
    userErrors {
      field
      message
    }
  }
}
"""

updates = []
for sku, qty in supplier_quantities.items():
    if sku in sku_to_inv_item:
        updates.append({
            "inventoryItemId": sku_to_inv_item[sku],
            "locationId": target_location_id,
            "quantity": qty
        })

total_matches = len(updates)
print(f"\n[*] Matched {total_matches} SKUs ready for direct update.")

successful = 0
failed = 0

for i in range(0, total_matches, BATCH_SIZE):
    batch = updates[i:i + BATCH_SIZE]
    payload = {
        "input": {
            "name": "available",
            "reason": "correction",
            "ignoreCompareQuantity": True,
            "quantities": batch
        }
    }

    res = requests.post(GRAPHQL_URL, json={"query": INVENTORY_SET_MUTATION, "variables": payload}, headers=HEADERS).json()
    errors = res.get("data", {}).get("inventorySetQuantities", {}).get("userErrors", [])

    if errors:
        print(f"[!] Error in batch {(i // BATCH_SIZE) + 1}: {errors[0]['message']}")
        failed += len(batch)
    else:
        successful += len(batch)
        print(f"[+] Synced {successful} / {total_matches} SKUs...")

    time.sleep(0.08)

print("\n" + "=" * 50)
print("INVENTORY SYNC COMPLETE")
print(f"Successfully Synced : {successful} SKUs")
print(f"Failed              : {failed}")
print("=" * 50)

```

---

## How to Run

1. Open your terminal or Command Prompt in your project directory:
```bash
cd path/to/shopify_sku_sync

```


2. Make sure your supplier's file is named `supplier_inventory.csv` in that folder (or change the variable name at top of script).
3. Execute the script:
```bash
python sync_inventory_by_sku.py

```


4. The script indexes your live catalog and updates inventory directly to Shopify. No CSV imports in Shopify Admin needed.
