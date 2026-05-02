# Amazon Cart Automation Report - 2026-05-02

## Goal

Add quantity 1 of the item shown in `IMG_1775.JPG` to the Amazon.com shopping cart, without checking out.

The photo showed an ARM & HAMMER Clean Burst liquid laundry detergent jug. The exact Amazon product family I identified was:

- `ARM & HAMMER Liquid Laundry Detergent, Clean Burst Fresh, 170 fl oz, 170 Loads`
- Amazon ASIN found from deal/reference data: `B0BSNXN4YX`
- Model/UPC reference from Slickdeals page: `033200975823` / `33200975823`

The exact 170 fl oz / 170-load variant was unavailable on Amazon at the time of the run, so I added the closest available same-scent variant from the same Amazon product family:

- `ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz`
- Quantity: `1`
- Cart subtotal shown by Amazon: `$15.98`
- Checkout was not opened or submitted.

## Tech Stack Used

The workflow used a small local automation stack:

- **PowerShell**: the command runner for file inspection, process checks, sleeps, and launching browser automation commands.
- **ImageMagick (`magick`)**: image identification and conversion. This was needed because `IMG_1775.JPG` was actually HEIC data behind a `.JPG` filename.
- **`uvx` with Python 3.13**: temporary package runner. It let me run `browser-use[cli]` without permanently installing the CLI into the environment.
- **Browser Use CLI (`browser-use`)**: the browser automation layer that opened Amazon, inspected the page state, clicked controls by element index, and preserved the named `amazon` browser session across commands.
- **Amazon.com web UI**: the actual shopping cart surface. No private Amazon API was used.
- **Web search/reference lookup**: used only to identify the exact historical Amazon ASIN/model for the 170 fl oz product after Amazon search results favored newer/nearby variants.

### What `browser-use[cli]` Is

`browser-use[cli]` is the Python package extra that exposes the `browser-use` command-line interface. In practical terms, it gives an agent-friendly command set for browser work:

```powershell
browser-use open https://example.com
browser-use state
browser-use click 5
browser-use type "text"
browser-use screenshot page.png
browser-use close
```

The important idea is the `state` command. Instead of forcing the agent to reason over the full raw DOM or maintain brittle CSS selectors, `browser-use state` returns a structured list of visible/clickable elements with numeric indices. The workflow then becomes:

1. Open a page.
2. Run `state`.
3. Pick the relevant element index.
4. Click or type using that index.
5. Re-run `state` after page changes.

In this run, examples included:

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon state
```

Then clicking the indexed controls Amazon exposed:

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon click 15323
uvx --python 3.13 "browser-use[cli]" --session amazon click 37141
```

Browser Use also supports a persistent browser daemon/session model. The named session (`--session amazon`) meant commands after the first open continued talking to the same browser context.

### Is Browser Use Open Source?

Yes. Browser Use's GitHub README says Browser Use is open source and free to use, and lists the project license as MIT. The project repository is:

- Browser Use GitHub: <https://github.com/browser-use/browser-use>
- Browser Use CLI docs: <https://docs.browser-use.com/open-source/browser-use-cli>

The open-source CLI/library is separate from Browser Use's optional hosted/cloud services. For this task, I used the CLI locally via `uvx`; I did not use a Browser Use Cloud browser.

### Is `browser-use[cli]` the Magic Behind This Workflow?

Mostly, yes, but not by itself.

The core browser-control "magic" was `browser-use[cli]`: it turned a live Amazon page into inspectable element indices, kept the browser session alive, and let me execute precise actions like selecting a product variant and clicking Add to Cart.

But the end-to-end workflow also depended on:

- ImageMagick to make the source image readable.
- Product reasoning to identify the detergent and avoid unrelated search results.
- External product reference lookup to find the exact ASIN for the older 170 fl oz variant.
- Human-level judgment when Amazon showed the exact variant was unavailable and the closest same-family, same-scent replacement was available.

So `browser-use[cli]` was the browser automation engine, not the whole workflow. It did not independently know what product was in the photo, decide substitution policy, or verify that checkout should be avoided. Those decisions came from the surrounding agent process.

## Problems Encountered

### 1. Image file extension did not match the real format

The file was named `IMG_1775.JPG`, but ImageMagick identified it as HEIC:

```powershell
magick identify -verbose '.\IMG_1775.JPG'
```

Key result:

```text
Format: HEIC (High Efficiency Image Format)
Mime type: image/heic
Geometry: 3024x4032
```

The app image viewer rejected the original file as an unsupported JPEG, and Windows `System.Drawing.Image.FromFile()` failed with an out-of-memory error because it was not actually a normal JPEG.

### 2. Image inspection needed a normalized copy

I used ImageMagick to create a PNG view copy while leaving the original untouched:

```powershell
magick '.\IMG_1775.JPG' -auto-orient -resize 1600x1600 '.\IMG_1775_view.png'
```

That made the product readable enough to identify:

- Brand: `ARM & HAMMER`
- Scent/product line: `Clean Burst`
- Label text: `Powerfully Clean Naturally Fresh`
- Load count visible: `170 loads`

I later generated a crop of the lower label area:

```powershell
magick '.\IMG_1775.JPG' -auto-orient -crop 1700x700+250+2300 -resize 2200x900 '.\IMG_1775_label_crop.png'
```

### 3. Browser automation CLI was not installed

The first intended browser tool was unavailable:

```powershell
browser-use doctor
```

Result:

```text
browser-use : The term 'browser-use' is not recognized ...
```

I checked the browser-use skill instructions and confirmed `uvx` was available, so I used `uvx` to run Browser Use without permanently installing it.

### 4. Browser Use needed Python 3.11+

This failed because the default `uvx` Python resolution used Python 3.10:

```powershell
uvx "browser-use[cli]" --help
```

The error said the current Python version did not satisfy `Python>=3.11`.

I fixed that by explicitly running Browser Use with Python 3.13:

```powershell
uvx --python 3.13 "browser-use[cli]" --help
```

### 5. Real Chrome profile launch timed out

I first tried to use the normal Chrome profile, so the Amazon cart would land in a logged-in browser context:

```powershell
uvx --python 3.13 "browser-use[cli]" --profile Default --headed open "https://www.amazon.com/s?k=Arm+%26+Hammer+Clean+Burst+170+loads+220+fl+oz+laundry+detergent"
```

That timed out waiting for the automation socket. I checked running Chrome processes and then switched to a clean headed Browser Use session named `amazon`.

### 6. Amazon exact variant was unavailable

Opening the exact ASIN page worked:

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon open "https://www.amazon.com/gp/product/B0BSNXN4YX"
```

But Amazon showed:

```text
Currently unavailable.
We don't know when or if this item will be back in stock.
```

The page also showed the `170 Fl Oz (Pack of 1)` size selected and unavailable. It offered available same-family Clean Burst sizes, including `205 Fl Oz (Pack of 1)`, in stock and sold by Amazon.com.

## Final Command Sequence That Added the Item

These are the browser commands that actually performed the Amazon cart add, after product identification and after discovering the exact 170 fl oz variant was unavailable.

### 1. Open Amazon search in a clean headed Browser Use session

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon --headed open "https://www.amazon.com/s?k=Arm+%26+Hammer+Clean+Burst+170+loads+220+fl+oz+laundry+detergent"
```

### 2. Inspect the Amazon search results

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon state
```

This showed several related Arm & Hammer products, including 205 fl oz, 200 fl oz, 174 fl oz, and 170 fl oz variants.

### 3. Open the exact known ASIN page

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon open "https://www.amazon.com/gp/product/B0BSNXN4YX"
```

### 4. Inspect the ASIN page

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon state
```

Important page state:

```text
ARM & HAMMER Liquid Laundry Detergent, Clean Burst Fresh, 170 fl oz, 170 Loads
Currently unavailable.
Size:
170 Fl Oz (Pack of 1)
```

The same page also exposed an available `205 Fl Oz (Pack of 1)` variant with an add-to-cart panel.

### 5. Select the available 205 fl oz variant

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon click 15323
```

After this click, the product page state showed:

```text
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
One-time purchase
$15.98
In Stock
Quantity:
Add to cart
Shipper / Seller
Amazon.com
```

### 6. Add the item to cart

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon click 37141
```

The clicked element was the product page's `Add to cart` control:

```text
input id=add-to-cart-button name=submit.add-to-cart title=Add to Shopping Cart type=submit value=Add to cart
```

### 7. Verify Amazon reported the item was added

```powershell
Start-Sleep -Seconds 3; uvx --python 3.13 "browser-use[cli]" --session amazon state
```

Key confirmation:

```text
1 item in cart
Added to cart
Scent:
Clean Burst
Size:
205 Fl Oz (Pack of 1)
Cart Subtotal:
$15.98
Proceed to checkout
(1 item)
```

I did not click `Proceed to checkout`.

### 8. Open the cart page for verification only

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon click 56751
```

This clicked `Go to Cart`, not checkout.

### 9. Verify the final cart line item and quantity

```powershell
Start-Sleep -Seconds 2; uvx --python 3.13 "browser-use[cli]" --session amazon state
```

Final cart confirmation:

```text
Subtotal (1 item):
$15.98

Shopping Cart
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
$15.98
In Stock
Scent:
Clean Burst
Size:
205 Fl Oz (Pack of 1)
```

## Complete Supporting Commands

These commands were used for diagnosis and setup, but were not the final cart-add path.

```powershell
Get-Content -LiteralPath "$HOME\.agents\skills\browser-use\SKILL.md" -TotalCount 220
```

```powershell
Get-Item -LiteralPath '.\IMG_1775.JPG' | Format-List FullName,Length
```

```powershell
Get-Command magick,ffmpeg,python -ErrorAction SilentlyContinue | Select-Object Name,Source
```

```powershell
magick identify -verbose '.\IMG_1775.JPG'
```

```powershell
magick '.\IMG_1775.JPG' -auto-orient -resize 1600x1600 '.\IMG_1775_view.png'
```

```powershell
browser-use doctor
```

```powershell
Get-Content -LiteralPath "$HOME\.agents\skills\agent-browser\SKILL.md" -TotalCount 260
```

```powershell
Get-Content -LiteralPath "$HOME\.codex\skills\playwright\SKILL.md" -TotalCount 260
```

```powershell
Get-Command uv,uvx,npx,node,chrome,msedge -ErrorAction SilentlyContinue | Select-Object Name,Source
```

```powershell
uvx "browser-use[cli]" --help
```

```powershell
uvx --python 3.13 "browser-use[cli]" --help
```

```powershell
uvx --python 3.13 "browser-use[cli]" --profile Default --headed open "https://www.amazon.com/s?k=Arm+%26+Hammer+Clean+Burst+170+loads+220+fl+oz+laundry+detergent"
```

```powershell
Get-Process chrome -ErrorAction SilentlyContinue | Select-Object Id,ProcessName,MainWindowTitle,Path | Select-Object -First 8
```

```powershell
magick '.\IMG_1775.JPG' -auto-orient -crop 1700x700+250+2300 -resize 2200x900 '.\IMG_1775_label_crop.png'
```

## Outcome

The cart ended with one item:

```text
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
Quantity: 1
Cart subtotal: $15.98
```

No checkout action was taken.
