---
name: deepweb
version: 2.0.0
description: Navigate login-walled websites with your real authenticated MCP Playwright session. Fan out across sites sequentially, compare results in a table, open checkout tabs ready to buy.
triggers:
  - deepweb
  - navigate to payment
  - check availability and book
allowed-tools:
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_click
  - mcp__playwright__browser_fill_form
  - mcp__playwright__browser_type
  - mcp__playwright__browser_tabs
  - mcp__playwright__browser_take_screenshot
  - mcp__playwright__browser_select_option
  - mcp__playwright__browser_press_key
  - mcp__playwright__browser_wait_for
  - Bash
  - AskUserQuestion
---

# /deepweb

Navigate any login-walled website to the payment step using MCP Playwright — your already-authenticated real browser session (cookies, fingerprint, logged-in state). Fan out across multiple sites, compare results in a table, open checkout tabs per selection.

**Rule: MCP Playwright is the ONLY navigation method. Never use $B, WebFetch, or `claude -p` subprocesses for navigation. They get 403'd on authenticated retail sites and add subprocess overhead with no benefit.**

## Usage

```
/deepweb [--sites URL1,URL2,...] [--dry-run] [--json] INTENT
```

- `--sites`    Comma-separated search URLs. Prompted if omitted.
- `--dry-run`  Navigate to payment page, extract summary, stop. Does not confirm purchase.
- `--json`     Emit structured JSON per site.
- `INTENT`     Natural language goal. Example: "powerful fan under 200€, delivery before July 2"

---

## Step 1: Parse invocation arguments

Read the user's message. Extract:
- **INTENT** — everything after the flags (the natural language goal)
- **SITES** — value of `--sites` flag (comma-separated URLs), or empty
- **DRY_RUN** — true if `--dry-run` present
- **JSON_OUT** — true if `--json` present

If `SITES` is empty, infer the best sites from INTENT (e.g., "fan" → French e-commerce: amazon.fr, fnac.com, darty.com, cdiscount.com, boulanger.com) and ask via AskUserQuestion which to check.

Split SITES on commas → `SITES_ARRAY`.

---

## Step 2: Navigate all sites using MCP Playwright

Navigate each site sequentially. For each site:

1. Call `mcp__playwright__browser_navigate` with the site URL.
2. Wait 500ms (cold load — pre-auth timing).
3. Run the **navigation loop** (up to 20 iterations).
4. Record the result JSON for this site.
5. Move to the next site (open a new tab if needed via `browser_evaluate: window.open('about:blank','_blank')`).

### Navigation loop (per site)

Each iteration:

**a) Snapshot:** Call `mcp__playwright__browser_snapshot` to read the accessibility tree.

**b) Classify page state:**
- `results_page` — search listing with multiple products
- `product_page` — single product detail
- `cart_page` — basket/cart
- `payment_page` — final checkout: price summary + confirm button visible
- `login_wall` — login/sign-in form (email+password, OAuth, "Connectez-vous")
- `unavailable` — out of stock, no results for query
- `blocked` — 403, CAPTCHA, access denied

**c) Act:**

**`results_page`:** Scan snapshot for products matching INTENT (price range, delivery date, product category). Extract up to 5 candidates with name, price, delivery date. Navigate to the best match or report "no match under constraints".

**`product_page`:** Extract name, price, delivery date. If matches INTENT, click "Ajouter au panier" / "Add to cart". If doesn't match, navigate back or try next result.

**`cart_page`:** Click "Passer la commande" / "Commander" / checkout CTA.

**`payment_page`:** Extract: product name, total price, delivery date, address shown, confirm button text. **STOP** — record result, proceed to next site. Do not click the confirm button.

**`login_wall`:** Run the **Login procedure** (below), then re-navigate to the original URL.

**`blocked`:** Record `{status: "blocked", note: "403/CAPTCHA"}`. Skip to next site.

**`unavailable`:** Record `{status: "unavailable"}`. Skip to next site.

### Login procedure

When a login wall is detected:

1. Extract domain from current URL.
2. Lookup credentials in Chrome Password Manager:

```bash
python3 - <<'PYEOF'
import subprocess, hashlib, sqlite3, shutil, os, sys, tempfile

def get_all_chrome_keys():
    r = subprocess.run(
        ['secret-tool','search','--all','xdg:schema','chrome_libsecret_os_crypt_password_v2'],
        capture_output=True, text=True
    )
    return [l.split('secret = ')[1].strip() for l in r.stdout.splitlines() if l.strip().startswith('secret = ')]

def try_decrypt(aes_key, ct):
    from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
    from cryptography.hazmat.backends import default_backend
    iv = b' ' * 16
    cipher = Cipher(algorithms.AES(aes_key), modes.CBC(iv), backend=default_backend())
    dec = cipher.decryptor()
    raw = dec.update(ct) + dec.finalize()
    pad = raw[-1]
    if not (1 <= pad <= 16): return None
    try: return raw[:-pad].decode('utf-8')
    except: return None

domain = os.environ.get('DOMAIN','')
fd, tmp_db = tempfile.mkstemp(suffix='.db', prefix='dw-')
os.close(fd)
try:
    shutil.copy(os.path.expanduser('~/.config/google-chrome/Default/Login Data'), tmp_db)
    os.chmod(tmp_db, 0o600)
    keys = get_all_chrome_keys()
    conn = sqlite3.connect(tmp_db); c = conn.cursor()
    c.execute("SELECT username_value, password_value FROM logins WHERE origin_url LIKE ? ORDER BY date_last_used DESC LIMIT 1",(f'%{domain}%',))
    row = c.fetchone(); conn.close()
    if not row or not row[1]: sys.exit(1)
    enc = bytes(row[1])
    if enc[:3] != b'v11': sys.exit(1)
    for k in keys:
        aes_key = hashlib.pbkdf2_hmac('sha1',k.encode(),b'saltysalt',1,dklen=16)
        pw = try_decrypt(aes_key, enc[3:])
        if pw is not None: print(f'{row[0]}\t{pw}'); sys.exit(0)
    sys.exit(1)
except SystemExit: raise
except: sys.exit(1)
finally:
    try: os.unlink(tmp_db)
    except OSError: pass
PYEOF
```

3. If credentials found:
   - Read snapshot to find email/username field selector
   - Fill username: `browser_fill_form` or `browser_evaluate`
   - Submit if needed (multi-step auth: email first, then password page)
   - Fill password, submit
   - If 2FA detected in snapshot, ask user for OTP code via AskUserQuestion
   - Continue navigation loop

4. If no credentials found:
   - Tell user: "Please log in to [domain] in your real Chrome browser, then press Enter."
   - Wait for user input, then re-navigate to original URL and continue

### Safety gate (required before any purchase confirmation)

Before clicking any button matching: "Place Order", "Commander", "Passer la commande", "Confirmer la commande", "Pay Now", "Payer maintenant" — **ALWAYS** ask:

> "About to click '[button text]' on [site] for [price]. This places the order. Confirm? [y/N]"

Do not proceed without explicit user confirmation.

### Timing

- Pre-auth / cold page load: wait 500ms after `browser_navigate`
- Post-auth actions (cart, checkout, delivery selection): wait 100ms between steps
- No other artificial delays needed

---

## Step 3: Comparison table

After navigating all sites, display:

```
## Results

| # | Site | Product | Price | Delivery | Status |
|---|------|---------|-------|----------|--------|
| 1 | amazon.fr | Rowenta Turbo Silence SF7552 | 149,99€ | 1 juil | ✓ found |
| 2 | fnac.com | DREO DR-HAF003S | 89€ | 2 juil | ✓ found |
| 3 | darty.com | — | — | — | ✗ blocked |
| 4 | cdiscount.com | Cecotec EnergySilence 1030 | 109€ | 3 juil | ✓ found |
```

Column rules:
- **Site** — domain only (no https://)
- **Product** — name + key spec, max 40 chars
- **Price** — total including shipping
- **Delivery** — best date found (human-readable, not epoch)
- **Status** — `✓ found`, `✓ at checkout`, `✗ unavailable`, `✗ blocked`, `⚠ error`

---

## Step 4: Select results and open checkout tabs

Ask via AskUserQuestion which results to open as checkout-ready tabs (multi-select, skip unavailable/blocked rows). If user selects none, stop here.

For each selected result, use MCP Playwright to:
1. Open a new tab: `browser_evaluate: window.open(url, '_blank')`
2. Navigate to product page
3. Add to cart
4. Proceed through checkout (address, delivery selection)
5. Reach final payment summary page — **STOP before clicking confirm**

Show final checkout tab summary:

```
## Checkout Tabs — Ready to Buy

| Tab | Site | Product | Total | Delivery | Confirm button |
|-----|------|---------|-------|----------|----------------|
| 1 | amazon.fr | Rowenta Turbo Silence SF7552 | 149,99€ | 1 juil | "Commander" |
| 2 | cdiscount.com | Cecotec EnergySilence 1030 | 109€ | 3 juil | "Passer la commande" |

All tabs are open. Click the confirm button in the tab of your choice.
⚠ No orders have been placed yet. You are in full control.
```

---

## Architecture notes

- **Why only MCP Playwright**: the session is already authenticated with real cookies and fingerprint. $B and WebFetch are unauthenticated headless sessions — they get 403'd by French retail (Fnac, Darty, BHV, Boulanger, Cdiscount all block them). MCP Playwright has zero 403 risk for sites where you're already logged in.
- **Sequential is fine**: each site navigation takes 5–15 seconds. 6 sites = ~60–90 seconds total. The old parallel bash approach was slower in practice because of subprocess startup, `claude -p` JSON marshaling, and 403 failures requiring retries.
- **Chrome Password Manager**: handles auto-login for sites not yet authenticated in the session. Tested on Amazon.fr (multi-step + WhatsApp 2FA), Cdiscount (single-step).
- **Tab management**: each site gets its own tab. After Step 2, all result tabs are open. Step 4 navigates them to checkout.
