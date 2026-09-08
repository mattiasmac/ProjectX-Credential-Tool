# ProjectX Credential Extractor & Front-End API Reference

A tiny browser bookmarklet that captures the credentials the TopstepX / ProjectX web app already uses in your browser (API base URL, Bearer token, account ID), plus a reference for the front-end `userapi` endpoints that the web app calls to place, modify, cancel, and close trades.

It exists so external tools (trade copiers, risk managers, custom dashboards) can talk to the same API the trading UI uses, without a separate API key subscription.

> **Disclaimer.** This targets the *internal* front-end API (`userapi.<platform>.com`), not the public, documented ProjectX Gateway API. It is unofficial, undocumented, and can change without notice. Check your platform's terms of service before automating anything against it. You are responsible for anything your tooling sends to a live account. Nothing here is affiliated with or endorsed by TopstepX or ProjectX.

---

## Contents

1. [How it works](#how-it-works)
2. [Install the bookmarklet](#install-the-bookmarklet)
3. [Capture your credentials](#capture-your-credentials)
4. [Output format](#output-format)
5. [Bookmarklet source (readable)](#bookmarklet-source-readable)
6. [Front-end API reference](#front-end-api-reference)
   - [Base URL, auth, headers](#base-url-auth-and-headers)
   - [Symbol format](#symbol-format)
   - [Order types](#order-types)
   - [Endpoints](#endpoints)
7. [Gotchas we learned the hard way](#gotchas-we-learned-the-hard-way)
8. [Security notes](#security-notes)

---

## How it works

The web app authenticates every API call with an `Authorization: Bearer <JWT>` header and includes your `accountId` in the request body of trading calls. The bookmarklet:

1. Injects a small floating panel into the page.
2. Monkey-patches `XMLHttpRequest.prototype.open/setRequestHeader/send` and `window.fetch`.
3. Watches for any request to a `userapi.*` host (`userapi.topstepx.com`, `userapi.projectx.com`, or any `*.projectx.com` subdomain).
4. Pulls the token from the `Authorization` header, the base URL from the request origin, and the `accountId` from the JSON body.
5. Shows the three values with copy buttons and a "download settings file" button.
6. Restores the original XHR/fetch functions when you close the panel.

Nothing leaves your browser. There is no server, no telemetry, no storage. The values only exist in memory on the page until you copy or download them.

Why a bookmarklet instead of a Chrome extension? Content Security Policy and Manifest V3 restrictions make intercepting page-level network calls from an extension awkward. A bookmarklet runs in the page context and just works.

---

## Install the bookmarklet

1. Show your bookmarks bar
   - Windows/Linux: `Ctrl + Shift + B`
   - Mac: `Cmd + Shift + B`
2. Create a new bookmark (right-click the bar → *Add page* / *New bookmark*).
3. Name it anything (e.g. `PX Creds`).
4. Paste the **entire** contents of [`bookmarklet.min.js`](./bookmarklet.min.js) into the URL field. It must start with `javascript:`. (The `%60` sequences are URL-encoded backticks — leave them; the browser decodes them when it runs the bookmark.)
5. Save.

If you host an install page, put the bookmarklet in an `<a href="javascript:...">` link so users can drag it straight to the bookmarks bar.

---

## Capture your credentials

1. Log in to TopstepX (or any ProjectX-based platform) and open the trading interface.
2. Click the bookmarklet. A panel appears in the top-right corner. Drag it anywhere.
3. **Place any order.** A limit order far away from the market is the safest — it will never fill and you can cancel it right after.
4. The panel switches to "Credentials Captured" and shows:
   - **API URL** – e.g. `https://userapi.topstepx.com`
   - **API Key (Bearer Token)** – the JWT, hidden by default (click *Show*)
   - **Account ID** – the numeric trading account ID, hidden by default
5. Use *Copy* on each field, or *Download Settings File* to save all three.
6. Cancel the test order. Close the panel with ✕ or press `Esc` — this removes the panel, un-patches XHR/fetch, and detaches all event listeners. Clicking the bookmarklet again while a panel is open closes the old one first.

Tips:

- The token is tied to your login session. When the web app logs you out or the session expires, capture again.
- If you have multiple accounts, the ID captured is whichever account the test order was placed on. Switch accounts and place another test order to capture a different ID.
- Placing an order is required because the account ID only appears in trading request bodies. Just loading the page will usually capture the token and URL but not the account.

---

## Output format

The downloaded file is plain text, one key per line:

```
API_URL=https://userapi.topstepx.com
API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....
ACCOUNT_ID=1234567
```

Filename: `projectx_credentials_YYYY-MM-DD.txt`.

---

## Bookmarklet source (readable)

<p align="center">
  <img src="https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/credential_tool.png" alt="The credential tool panel after a successful capture: API URL, masked Bearer token, masked account ID, download button, and a log of the intercepted calls" width="460">
</p>

[`bookmarklet.min.js`](./bookmarklet.min.js) is the shipped, minified build (plain black-and-white UI, no branding). The listing below is the same logic unminified so you can read it before trusting it. If you edit it, minify with any JS minifier, prefix with `javascript:`, and URL-encode backticks as `%60`.

```javascript
(function () {
  // If the bookmarklet is clicked twice, close the existing panel cleanly (restores hooks) first
  if (typeof window.pxClose === 'function') window.pxClose();
  else document.getElementById('px-monitor')?.remove();

  // ---------- UI ----------
  const panel = document.createElement('div');
  panel.id = 'px-monitor';
  panel.style.cssText =
    'position:fixed;top:10px;right:10px;width:450px;background:#fff;border:3px solid #444;' +
    'border-radius:10px;padding:20px;z-index:999999;font-family:Arial,sans-serif;' +
    'box-shadow:0 5px 15px rgba(0,0,0,.3);cursor:move;';
  panel.innerHTML = `
    <h3 style="margin:0 0 15px 0;">Credential Extractor</h3>
    <div id="px-status" style="padding:10px;background:#fff3cd;border-radius:5px;margin-bottom:15px;color:#856404;">
      ⏳ Monitoring... place an order
    </div>
    <div id="px-results" style="display:none;background:#f8f9fa;padding:15px;border-radius:5px;margin-bottom:15px;">
      <label>API URL</label>
      <div style="display:flex;gap:5px;margin-bottom:12px;">
        <input id="px-url" type="text" readonly style="flex:1;font-family:monospace;">
        <button onclick="pxCopy('url')">Copy</button>
      </div>
      <label>API Key (Bearer Token)</label>
      <div style="display:flex;gap:5px;margin-bottom:12px;">
        <input id="px-key" type="password" readonly style="flex:1;font-family:monospace;">
        <button onclick="pxToggle('key')">Show</button>
        <button onclick="pxCopy('key')">Copy</button>
      </div>
      <label>Account ID</label>
      <div style="display:flex;gap:5px;margin-bottom:15px;">
        <input id="px-account" type="password" readonly style="flex:1;font-family:monospace;">
        <button onclick="pxToggle('account')">Show</button>
        <button onclick="pxCopy('account')">Copy</button>
      </div>
      <button onclick="pxDownload()" style="width:100%;padding:12px;">💾 Download Settings File</button>
    </div>
    <div id="px-log" style="font-size:11px;color:#666;background:#f8f9fa;padding:10px;max-height:150px;overflow-y:auto;font-family:monospace;"></div>
    <button id="px-close" type="button" aria-label="Close" title="Close (Esc)" style="position:absolute;top:10px;right:10px;">✕</button>`;
  document.body.appendChild(panel);

  // Simple drag support (skip when clicking inputs/buttons/log)
  let dragging = false, offX = 0, offY = 0, curX = 0, curY = 0;
  panel.addEventListener('mousedown', e => {
    if (['INPUT', 'BUTTON'].includes(e.target.tagName) || e.target.id === 'px-log') return;
    dragging = true; offX = e.clientX - curX; offY = e.clientY - curY;
  });
  document.addEventListener('mousemove', e => {
    if (!dragging) return;
    curX = e.clientX - offX; curY = e.clientY - offY;
    panel.style.transform = `translate(${curX}px, ${curY}px)`;
  });
  const onUp = () => dragging = false;
  document.addEventListener('mouseup', onUp);
  const onKey = e => { if (e.key === 'Escape' && typeof window.pxClose === 'function') window.pxClose(); };
  document.addEventListener('keydown', onKey);

  // ---------- State ----------
  window.pxData = { url: '', key: '', id: '' };

  const isApiUrl = u =>
    u && (u.includes('userapi.topstepx.com') || u.includes('userapi.projectx.com') || u.includes('.projectx.com'));

  function log(msg) {
    const el = document.getElementById('px-log');
    if (!el) return;
    el.innerHTML += `[${new Date().toLocaleTimeString()}] ${msg}<br>`;
    el.scrollTop = el.scrollHeight;
  }

  function setOrigin(u) {
    try { window.pxData.url = new URL(u).origin; }
    catch { window.pxData.url = 'https://userapi.projectx.com'; }
  }

  function grabToken(header) {
    if (header && header.startsWith('Bearer ')) {
      window.pxData.key = header.substring(7);
      log('Got API key');
      showResults();
    }
  }

  function grabAccountFromBody(body) {
    if (!body || typeof body !== 'string') return;
    try {
      const j = JSON.parse(body);
      if (j.accountId) { window.pxData.id = String(j.accountId); log('Account ID: ' + window.pxData.id); showResults(); }
    } catch { /* not JSON */ }
  }

  function showResults() {
    if (!window.pxData.key || !window.pxData.id) return;
    if (!window.pxData.url) window.pxData.url = 'https://userapi.projectx.com';
    const s = document.getElementById('px-status');
    s.innerHTML = '✅ <strong>Credentials captured</strong>';
    s.style.background = '#d4edda'; s.style.color = '#155724';
    document.getElementById('px-results').style.display = 'block';
    document.getElementById('px-url').value = window.pxData.url;
    document.getElementById('px-key').value = window.pxData.key;
    document.getElementById('px-account').value = window.pxData.id;
  }

  // ---------- Hook XMLHttpRequest ----------
  const XP = XMLHttpRequest.prototype;
  const origOpen = XP.open, origSetHeader = XP.setRequestHeader, origSend = XP.send;

  XP.open = function (method, url) { this._url = url; return origOpen.apply(this, arguments); };
  XP.setRequestHeader = function (name, value) {
    if (name.toLowerCase() === 'authorization') grabToken(value);
    return origSetHeader.apply(this, arguments);
  };
  XP.send = function (body) {
    if (isApiUrl(this._url)) { log('XHR → ' + this._url); setOrigin(this._url); grabAccountFromBody(body); }
    return origSend.apply(this, arguments);
  };

  // ---------- Hook fetch ----------
  const origFetch = window.fetch;
  window.fetch = function (...args) {
    const [url, opts] = args;
    if (typeof url === 'string' && isApiUrl(url)) {
      log('fetch → ' + url); setOrigin(url);
      if (opts && opts.headers) {
        const h = opts.headers instanceof Headers
          ? opts.headers.get('Authorization')
          : (opts.headers.Authorization || opts.headers.authorization);
        grabToken(h || '');
      }
      if (opts && opts.body) grabAccountFromBody(opts.body);
    }
    return origFetch.apply(this, args);
  };

  // ---------- Buttons ----------
  window.pxClose = function () {
    document.removeEventListener('mouseup', onUp);
    document.removeEventListener('keydown', onKey);
    document.getElementById('px-monitor')?.remove();
    XP.open = origOpen; XP.setRequestHeader = origSetHeader; XP.send = origSend;
    window.fetch = origFetch; window.pxData = null; window.pxClose = null;
  };
  // Wire the ✕ button with a real listener (inline onclick could be blocked by the page's CSP,
  // which is why the panel sometimes failed to close in earlier versions)
  document.getElementById('px-close').addEventListener('click', e => { e.preventDefault(); e.stopPropagation(); window.pxClose(); });
  window.pxCopy = function (which) {
    const v = { url: window.pxData.url, key: window.pxData.key, account: window.pxData.id }[which];
    navigator.clipboard.writeText(v).then(() => log('Copied ' + which));
  };
  window.pxToggle = function (which) {
    const el = document.getElementById('px-' + which);
    el.type = el.type === 'password' ? 'text' : 'password';
  };
  window.pxDownload = function () {
    const text = `API_URL=${window.pxData.url}\nAPI_KEY=${window.pxData.key}\nACCOUNT_ID=${window.pxData.id}`;
    const a = document.createElement('a');
    a.href = URL.createObjectURL(new Blob([text], { type: 'text/plain' }));
    a.download = `projectx_credentials_${new Date().toISOString().slice(0, 10)}.txt`;
    document.body.appendChild(a); a.click(); a.remove();
  };

  log('Monitor ready — place an order');
})();
```

---

## Front-end API reference

Everything below was captured from the web app's network traffic and its bundled API client (`usrApi.ts`), then confirmed with live orders. Field names are exact; treat them as case-sensitive.

### Base URL, auth, and headers

| Item | Value |
|---|---|
| Base URL | `https://userapi.topstepx.com` (TopstepX) or `https://userapi.<platform>.com` — use whatever the extractor captured |
| Auth | `Authorization: Bearer <token>` |
| Content type | `Content-Type: application/json` on POST/PUT |
| Accept | `Accept: application/json` |

Some endpoints (notably the DELETE position/order calls) were more reliable when we also mirrored the browser's headers:

```
Origin: https://www.topstepx.com
Referer: https://www.topstepx.com/
X-App-Type: px-desktop
X-App-Version: 1.19.1
```

The version string will drift; copy whatever your browser currently sends.

### Symbol format

The front-end API uses CQG-style symbol IDs with an `F.US.` prefix, **not** the `CON.F.US.ENQ.Z25` contract IDs from the public Gateway API. The front-end resolves the active contract month for you.

| Product | Mini | Micro |
|---|---|---|
| S&P 500 | `F.US.EP` | `F.US.MES` |
| Nasdaq-100 | `F.US.ENQ` | `F.US.MNQ` |
| Dow | `F.US.YM` | `F.US.MYM` |
| Russell 2000 | `F.US.RTY` | `F.US.M2K` |
| Crude oil | `F.US.CL` | `F.US.MCLE` |
| Gold | `F.US.GC` | `F.US.MGC` |
| Silver | `F.US.SI` | `F.US.MSI` |
| Euro FX | `F.US.EC` | `F.US.M6E` |
| British pound | `F.US.BP` | `F.US.M6B` |
| Japanese yen | `F.US.JY` | `F.US.MJY` |
| Bitcoin | `F.US.BTC` | `F.US.MBT` |
| Ether | `F.US.ETH` | `F.US.MET` |
| 10-yr note | `F.US.TY` | — |
| Natural gas | `F.US.NG` | — |

Note the ones that don't follow the "add an M" pattern: ES→`EP`, NQ→`ENQ`, 6E→`EC`, 6B→`BP`, 6J→`JY`, MCL→`MCLE`. When in doubt, place a manual order in the web UI and read `symbolId` from the request body.

### Order types

Direction is **not** part of the type. The sign of `positionSize` is the direction: positive = buy, negative = sell.

| `type` | Meaning | Price field used |
|---|---|---|
| `1` | Limit | `limitPrice` |
| `2` | Market | none |
| `4` | Stop (stop-market) | `stopPrice` |
| `6` | Join bid | none |
| `7` | Join ask | none |

Stop-limit orders exist in the UI but we submit them as type `4` (stop) and haven't mapped a distinct type for them.

### Endpoints

Every path below is relative to the base URL.

#### Place an order — `POST /Order`

Used for **new positions and adding to positions**. Do not use it to flat a position (see [Gotchas](#gotchas-we-learned-the-hard-way)).

```json
{
  "accountId": 1234567,
  "symbolId": "F.US.MNQ",
  "type": 1,
  "limitPrice": 20150.25,
  "stopPrice": null,
  "positionSize": -2,
  "trailDistance": null,
  "positionEffect": 1,
  "customTag": "my-tag-001"
}
```

- Market order: `type: 2`, both prices `null`.
- Stop order: `type: 4`, `limitPrice: null`, `stopPrice` set.
- `positionEffect: 1` is what the web UI sends for opening trades.
- `customTag` is free text you get back in order/position records — handy for tracking your own IDs.

The response contains the new order record; keep its `id` if you plan to cancel or modify it later.

#### Cancel one order — `DELETE /Order/cancel/{accountId}/id/{orderId}`

No body.

#### Cancel all orders on an account — `DELETE /Order/cancel/{accountId}/all`

No body. Cancels every working order on the account, all symbols.

#### Close a position for one symbol — `DELETE /Position/close/{accountId}/symbol/{symbolId}`

Example: `DELETE /Position/close/1234567/symbol/F.US.MNQ`. Flattens that symbol at market. This is the correct way to exit; it does not create an opposing position.

#### Close all positions (flatten) — `DELETE /Position/close/{accountId}`

Flattens every open position on the account at market. Pair it with `/Order/cancel/{accountId}/all` for a true "flatten and cancel" panic button.

#### List open positions — `GET /Position?accountId={accountId}`

Returns an array of position objects. The fields you'll care about:

```json
{
  "id": 987654321,
  "accountId": 1234567,
  "symbolId": "F.US.MNQ",
  "positionSize": -2,
  "averagePrice": 20148.50,
  "stopLoss": null,
  "takeProfit": null
}
```

**The position ID is in `id`, not `positionId`.** You need it for the stop/target call below. Polling this every couple of seconds is a reasonable way to keep a symbol → position ID map current.

#### Attach or edit a bracket (stop + target) — `POST /Order/editStopLossAccount`

This is the "OCO" primitive. It attaches a stop-loss and/or take-profit to an existing position; the platform manages the one-cancels-other behaviour. Calling it again with new prices **modifies** the existing stop/target — one call moves a stop to breakeven or trails it.

```json
{
  "positionId": 987654321,
  "stopLoss": 20200.00,
  "takeProfit": 20050.00,
  "tradingAccountId": 1234567
}
```

- Pass `null` for either price to leave that side unset.
- The account field here is `tradingAccountId`, not `accountId`.
- Because it's keyed on the position, the stop and target are automatically removed when the position closes.

#### Other endpoints seen in the client (not yet exercised by us)

These appear in the bundled API client and are listed for completeness. Payloads unverified.

| Method / path | Likely purpose |
|---|---|
| `POST /Order/editStopLoss` | Older single-account variant of `editStopLossAccount` |
| `PUT /Order/edit/stopLimit/{orderId}` | Modify a working stop-limit's prices |
| `PUT /Order/edit/trailingDistance/{orderId}` | Modify a trailing stop's distance |
| `POST /Order/postWithContract` | Place an order using an explicit contract rather than the front-month symbol |
| `POST /Position/sltp` | Alternate stop/target entry point |
| `POST /Position/reverse` / `/Position/reverseByContract` | Flip a position |
| `GET /Position/all/account/{accountId}` | Positions incl. history |
| `GET /Trade/range`, `/Trade/id/{id}` | Fill history |
| `GET /TradingAccount` | List accounts (useful for multi-account support) |
| `GET /TradeClock`, `/TradeLimit`, `/TradingAccount/personalLimits` | Session and risk-limit info |

---

## Gotchas we learned the hard way

1. **Exits must use `/Position/close/...`, never `/Order`.** Sending an opposite-side order through `/Order` to flatten will, depending on timing, open a new position in the other direction instead of closing.
2. **Direction lives in the sign of `positionSize`.** `type` is the same for buys and sells. Sending a positive size with a "sell" mindset gives you a long.
3. **Position ID comes back as `id`.** Reading `positionId` from the position list gives you nothing.
4. **`editStopLossAccount` wants `tradingAccountId`**, while `/Order` wants `accountId`. Same number, different key.
5. **Capture on `userapi.*`, not `chartapi.*`.** Chart data requests also carry a Bearer token, but the account ID only appears on trading calls. Early versions of the extractor watched the wrong host.
6. **Symbols are `F.US.XXX`**, and a few of them (`EP`, `ENQ`, `EC`, `BP`, `JY`, `MCLE`) are not what you'd guess.
7. **Tokens expire with the web session.** Build re-capture into your workflow rather than assuming a token is good for the day.
8. **Bracket orders arrive as normal orders from your platform.** If you are copying from a platform that submits stop/target as separate child orders, intercept them by name/type and route them to `editStopLossAccount` — don't forward them through `/Order`, or you'll double up.

---

## Security notes

- The Bearer token is a full session credential. Anyone holding it can trade your account until it expires. Treat the downloaded file like a password.
- Don't commit captured credential files. Add `*_credentials_*.txt` to `.gitignore`.
- The bookmarklet has no network component; you can read every line of it above. If you're distributing a minified version, publish the readable source next to it so users can verify.
- Hide the panel or use the masked fields when screen-sharing or recording.

---

## License

MIT. No warranty. See the disclaimer at the top.
