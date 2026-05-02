# Amazon Cart Automation Report - 2026-05-02

## Goal

Add quantity 1 of the item shown in `IMG_1775.JPG` to the Amazon.com shopping cart, without checking out.

The photo showed an ARM & HAMMER Clean Burst liquid laundry detergent jug. The exact Amazon product family I identified was:

- `ARM & HAMMER Liquid Laundry Detergent, Clean Burst Fresh, 170 fl oz, 170 Loads`
- Amazon ASIN found from deal/reference data: `B0BSNXN4YX`
- Model/UPC reference from Slickdeals page: `033200975823` / `33200975823`

The exact 170 fl oz / 170-load variant was unavailable on Amazon at the time of the first run, so I used the closest available same-scent variant from the same Amazon product family:

- `ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz`
- Quantity: `1`
- Cart subtotal shown by Amazon: `$15.98`
- Checkout was not opened or submitted.

Important correction: the first round added the item to an isolated Browser Use browser session, not to the user's signed-in Chrome/Amazon session. A second round was required to add the item to the real logged-in Amazon cart.

## Tech Stack Used

The workflow used a small local automation stack:

- **PowerShell**: the command runner for file inspection, process checks, sleeps, and launching browser automation commands.
- **ImageMagick (`magick`)**: image identification and conversion. This was needed because `IMG_1775.JPG` was actually HEIC data behind a `.JPG` filename.
- **`uvx` with Python 3.13**: temporary package runner. It let me run `browser-use[cli]` without permanently installing the CLI into the environment.
- **Browser Use CLI (`browser-use`)**: the browser automation layer that opened Amazon, inspected the page state, clicked controls by element index, and preserved the named `amazon` browser session across commands.
- **Visible Chrome + Windows UI automation**: used in the second round to interact with the user's already signed-in Chrome window after Browser Use could not attach to it.
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

## First-Round Command Sequence

These are the browser commands that added the item in the first round, after product identification and after discovering the exact 170 fl oz variant was unavailable.

This sequence was technically successful inside the Browser Use session, but it did not update the user's real signed-in Amazon cart. Browser Use had launched and controlled its own temporary browser profile. Amazon treated that temporary profile as a separate signed-out browser.

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

### Why this did not show up in the real Amazon cart

The core mistake was session scope.

Browser Use created an isolated browser environment. That browser had its own:

- temporary Chrome user data directory
- cookies
- local storage
- Amazon cart state
- login state

The user's visible Chrome window had a different session:

- signed in to Amazon
- existing account cookies
- cart count visible in the real Amazon header

The first-round Browser Use state even showed Amazon as signed out:

```text
Hello, sign in
Account & Lists
```

So the first-round cart confirmation meant:

```text
The item exists in the Browser Use temporary session's Amazon cart.
```

It did not mean:

```text
The item exists in the user's signed-in Amazon account cart.
```

The user's screenshot after the first round showed the real Chrome/Amazon header still had `0` cart items. That was the evidence that the successful first-round automation had affected the wrong browser session.

## Second-Round Correction

The second round targeted the user's visible signed-in Chrome window instead of the isolated Browser Use browser.

The final corrected result was verified on the real `https://www.amazon.com/cart` page:

```text
Shopping Cart
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
Quantity: 1
Subtotal (1 item): $15.98
```

Checkout was not opened.

### 1. User reported the real cart was still empty

The user provided a screenshot of the signed-in Amazon header. It showed:

```text
Hello, az9713
Cart 0
```

That meant the first-round success was not enough. The target had to be the logged-in browser session, not the Browser Use session.

### 2. I checked whether Browser Use could attach to the real Chrome session

First I inspected Chrome processes to see how Chrome was running:

```powershell
Get-CimInstance Win32_Process -Filter "name='chrome.exe'" |
  Select-Object ProcessId,CommandLine |
  Format-List
```

Then I tried Browser Use's connect mode:

```powershell
uvx --python 3.13 "browser-use[cli]" --connect state
```

That failed with:

```text
Error: Could not discover a running Chrome instance with remote debugging enabled.
Enable remote debugging in Chrome (chrome://inspect, or launch with --remote-debugging-port=9222) and try again.
```

This was expected once the process list showed the user's real Chrome was not launched with a remote debugging port. Browser Use can attach to a real browser only when Chrome exposes a Chrome DevTools Protocol endpoint, for example with `--remote-debugging-port=9222`.

Because the existing signed-in Chrome did not expose that endpoint, Browser Use could not safely control it by `state` and `click`.

### 3. I tested the direct Amazon add-to-cart URL in the isolated session

I tried Amazon's legacy add-to-cart URL in the Browser Use session:

```powershell
uvx --python 3.13 "browser-use[cli]" --session amazon open "https://www.amazon.com/gp/aws/cart/add.html?ASIN.1=B0GQHXMDGT&Quantity.1=1"
Start-Sleep -Seconds 3
uvx --python 3.13 "browser-use[cli]" --session amazon state
```

That landed on Amazon sign-in:

```text
Sign in
Enter mobile number or email
```

This reinforced the diagnosis: the Browser Use session was isolated and not signed in.

### 4. I tested the same direct add-to-cart URL in the real Chrome window

Next I opened the direct add-to-cart URL in the existing Chrome application:

```powershell
Start-Process -FilePath 'chrome.exe' -ArgumentList 'https://www.amazon.com/gp/aws/cart/add.html?ASIN.1=B0GQHXMDGT&Quantity.1=1'
```

That did use the signed-in Chrome profile, but Amazon rendered:

```text
Cart is empty
```

So the old direct add-to-cart URL was not a reliable path for this item/session. It did not add the product to the real cart.

### 5. I opened the normal Amazon product page in real Chrome

I switched to the normal product-page path:

```powershell
Start-Process -FilePath 'chrome.exe' -ArgumentList 'https://www.amazon.com/dp/B0GQHXMDGT'
```

The visible Chrome page showed the correct product:

```text
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
$15.98
In Stock
```

This confirmed the target item and offer were available in the signed-in browser.

### 6. The Add to Cart button was initially hard to reach

The page loaded with the product title and image visible, but the buy box was partly off to the right of the screenshot/viewport. The `Add to cart` button was not safely visible at first.

I tried several UI positioning actions:

```powershell
[System.Windows.Forms.SendKeys]::SendWait('^f')
[System.Windows.Forms.SendKeys]::SendWait('Add to cart')
[System.Windows.Forms.SendKeys]::SendWait('{ESC}')
```

That did not bring the button into view reliably.

I then paged and zoomed the Chrome page:

```powershell
[System.Windows.Forms.SendKeys]::SendWait('{PGDN}')
[System.Windows.Forms.SendKeys]::SendWait('^{HOME}')
[System.Windows.Forms.SendKeys]::SendWait('^-')
[System.Windows.Forms.SendKeys]::SendWait('^-')
```

Zooming out exposed the buy box on the right side of the page:

```text
One-time purchase
$15.98
In Stock
Quantity: 1
Add to cart
Buy Now
```

The visible UI now contained the correct control. The next problem was reliably clicking it.

### 7. Coordinate clicking was unreliable

I attempted to click the visible yellow Add to Cart button by screen coordinate:

```powershell
[MouseOps]::SetCursorPos(1494,844)
[MouseOps]::mouse_event(0x0002,0,0,0,[UIntPtr]::Zero)
[MouseOps]::mouse_event(0x0004,0,0,0,[UIntPtr]::Zero)
```

That clicked too far right and opened the Windows notification/calendar sidebar instead of the Amazon button.

I closed the panel, zoomed further out, and tried again with a safer coordinate:

```powershell
[MouseOps]::SetCursorPos(1455,730)
[MouseOps]::mouse_event(0x0002,0,0,0,[UIntPtr]::Zero)
[MouseOps]::mouse_event(0x0004,0,0,0,[UIntPtr]::Zero)
```

That still did not successfully submit the add-to-cart action. One issue was focus: the Codex app came to the foreground during one interaction, so a click aimed at Chrome was not guaranteed to land in the browser.

To reduce focus ambiguity, I then focused Chrome by its real window handle:

```powershell
$chrome = Get-Process chrome |
  Where-Object { $_.MainWindowTitle -like 'Amazon.com: ARM & HAMMER*' } |
  Select-Object -First 1

[WinMouse]::ShowWindow($chrome.MainWindowHandle,3) | Out-Null
[WinMouse]::SetForegroundWindow($chrome.MainWindowHandle) | Out-Null
```

Even with Chrome focused, coordinate clicking remained brittle. The page was zoomed, the browser window was near the screen edge, and the button was close to the right boundary. A DOM-level click inside the real Chrome page was more reliable.

### 8. The first JavaScript bookmarklet attempt was stripped by Chrome

Because Browser Use could not attach to the real Chrome profile, I used the visible Chrome address bar to run a bookmarklet-style command against the already-open Amazon product page:

```javascript
javascript:document.getElementById('add-to-cart-button').click()
```

The first attempt pasted the whole command into the address bar. Chrome stripped the `javascript:` prefix and treated the rest as a Google search query:

```text
document.getElementById('add-to-cart-button').click()
```

That did not click the button.

This is a Chrome safety behavior: pasted `javascript:` URLs are often sanitized so users do not accidentally execute pasted code.

### 9. I worked around Chrome's paste-stripping behavior

The workaround was to type the `java` prefix manually, then paste only the remaining part:

```powershell
[System.Windows.Forms.SendKeys]::SendWait('^l')
[System.Windows.Forms.SendKeys]::SendWait('java')
Set-Clipboard -Value "script:document.getElementById('add-to-cart-button').click()"
[System.Windows.Forms.SendKeys]::SendWait('^v')
[System.Windows.Forms.SendKeys]::SendWait('{ENTER}')
```

This reconstructed the full address bar command as:

```javascript
javascript:document.getElementById('add-to-cart-button').click()
```

Because it ran inside the real signed-in Chrome tab, it clicked Amazon's actual product-page Add to Cart button in the user's session.

Amazon then navigated to a cart interstitial showing:

```text
Added to cart
Scent: Clean Burst
Size: 205 Fl Oz (Pack of 1)
```

That was the first successful evidence that the real signed-in browser cart had been updated.

### 10. I verified the real cart page

Finally I opened the real cart page in Chrome:

```powershell
Start-Process -FilePath 'chrome.exe' -ArgumentList 'https://www.amazon.com/cart'
```

The cart page showed:

```text
Shopping Cart
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
In Stock
Scent: Clean Burst
Size: 205 Fl Oz (Pack of 1)
Quantity: 1
Subtotal (1 item): $15.98
```

I stopped there and did not click `Proceed to checkout`.

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

After the first round, only the isolated Browser Use cart contained the item. After the second round, the user's real signed-in Amazon cart contained one item:

```text
ARM & HAMMER Liquid Laundry Detergent, Powerfully Clean, Clean Burst Scent, More Loads, 205 Loads, 205 Fl Oz
Quantity: 1
Cart subtotal: $15.98
```

No checkout action was taken.
