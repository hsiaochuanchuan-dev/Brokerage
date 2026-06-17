# FunPage Project

## Data Source

- **File:** [OtherNote (Google Sheets)](https://docs.google.com/spreadsheets/d/1pljY-of-ICBP8WQVgg8l5ICSpL3F6rMs3B8XJXdP2t4/edit)
- **Sheet:** `dividend`
- **Range:** `A1:J18`
<!-- - **Access:** Read-only — do not write back to the spreadsheet -->

## Brokerage Ranking Task (scheduled, runs after market close on trading days)

### Output
- **Sheet:** `Brokerage` (gid=1267288898)
- **B1:** Query date (YYYY/MM/DD format)
- **Row 2:** Headers — 代號, 股價, 名稱, 原因, 操作重點
- **Rows 3–7:** Top **5** stocks only (NOT 10)

### Key Lessons Learned
1. **Top 5 only** — user confirmed: only show top 5 ranked stocks, rows 8–12 must remain empty.
2. **Enter names cell-by-cell** — never type multiple names in one go with Enter between them; each name must be entered individually using name-box navigation → type → Return to avoid concatenation with existing cell content.
3. **Overwrite existing cells** — use `double_click` on the cell + `ctrl+a` + `type` new content + `Return`. The name-box + Delete + type approach does NOT reliably clear existing content.
4. **Verify E column in formula bar** — after writing 操作重點, click each E cell and check the formula bar shows the full text (not just a tail fragment). If truncated, double_click + ctrl+a + retype.
5. **Stock prices via `mcp__workspace__web_fetch` + TWSE API (PRIMARY)** — `mcp__workspace__web_fetch` CAN reach TWSE directly. URL: `https://www.twse.com.tw/rwd/zh/afterTrading/STOCK_DAY?stockNo={CODE}&date={YYYYMMDD}&response=json`. Parse `data[-1][6]` for closing price. Chrome JS (Yahoo Finance) is fallback only.
6. **ETF filter** — exclude codes that are non-4-digit, start with `00`, or equal `0050`.
7. **Use `browser_batch` (not `computer-use`) to click/type in Chrome** — `mcp__computer-use__*` tools are tier "read" for browsers (no clicks/typing). Use `mcp__Claude_in_Chrome__browser_batch` with `{"name":"computer","input":{"action":"left_click/double_click/type/key","coordinate":[x,y],"tabId":N}}` instead. This is the only reliable way to click cells and type in Google Sheets.
8. **Chrome tab setup sequence** — always: `list_connected_browsers` → `select_browser(deviceId)` → `request_access(["Google Chrome"])` → `open_application("Google Chrome")` → `tabs_context_mcp(createIfEmpty:true)`. The `open_application` step is MANDATORY — without it, the extension stays in the sidepanel window which doesn't support tab groups ("Grouping is not supported by tabs in this window"). If `request_access` times out (no user present), skip it and try `open_application` anyway.
9. **D/E columns: NEVER use `type` for Chinese text — use clipboard** — Windows IME converts certain characters (e.g. 入 U+5165 → 兑, 搭 U+642D → 搜). For ALL Chinese text in D and E columns: `write_clipboard(text)` → name-box navigate → `F2` → `ctrl+a` → `ctrl+v` → `Return`.
10. **C column (名稱) overwrites require double_click** — `left_click + type` on a text cell APPENDs to existing content. Always use `double_click + ctrl+a + type + Return` for C3–C7 when overwriting.
11. **Column A/B bulk entry works** — clicking the first cell and typing all values separated by `\n` replaces content and advances row-by-row. Works reliably for numeric/code columns (A, B).
12. **Unicode escape caution** — when using `\uXXXX` escapes in `type` text, double-check codepoints. Better: avoid `type` entirely for Chinese D/E columns; use clipboard instead (lesson 9).
13. **Chrome `crypto.subtle.importKey` rejects PKCS#8 service account keys** — Chrome's WebCrypto validates RSA-CRT parameters strictly and throws `DataError` on import, even though Node.js accepts the same key. **Do NOT try to import the service account key in the browser.** Workaround: sign the JWT in the sandbox (Node.js `createSign`), inject the signed JWT string directly into `javascript_tool`, then use `fetch()` in the browser to exchange it for an access token at `https://oauth2.googleapis.com/token`.
14. **Sheets API via browser `fetch` with service account token** — once you have an access token (from lesson 13), use `fetch` inside `javascript_tool` to call the Sheets REST API directly. This is **far more reliable than any UI click/paste approach** for writing Chinese text. Pattern:
    ```javascript
    const h = {Authorization:`Bearer ${token}`,'Content-Type':'application/json'};
    const SID = '1pljY-of-ICBP8WQVgg8l5ICSpL3F6rMs3B8XJXdP2t4';
    fetch(`https://sheets.googleapis.com/v4/spreadsheets/${SID}/values/Brokerage!A3:E7?valueInputOption=RAW`,
      {method:'PUT', headers:h, body:JSON.stringify({values: rows})});
    ```
    Write all 5 rows in a **single batch PUT** to `A3:E7`. Clear stale rows with `POST` to `.../Brokerage!A8:E12:clear`.
15. **`navigator.clipboard.writeText` in `javascript_tool` does NOT work for Sheets paste** — even though the call returns `ok`, pressing `ctrl+v` inside a Google Sheets cell pastes nothing (or uses Sheets' internal clipboard). The only reliable approaches are: (a) Sheets REST API (lesson 14), or (b) the existing `write_clipboard` computer-use tool (requires user to grant `clipboardWrite`). Never rely on `navigator.clipboard` → Sheets paste.
16. **JWT signing workflow (no user present)** — complete recipe:
    ```bash
    # Step 1 (sandbox): sign JWT and print to stdout
    node -e "
      const {createSign}=require('crypto');
      const k=require('/path/to/key.json');
      function b64url(s){return Buffer.from(s).toString('base64').replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_');}
      const now=Math.floor(Date.now()/1000);
      const h=b64url(JSON.stringify({alg:'RS256',typ:'JWT'}));
      const p=b64url(JSON.stringify({iss:k.client_email,scope:'https://www.googleapis.com/auth/spreadsheets',aud:'https://oauth2.googleapis.com/token',iat:now,exp:now+3600}));
      const s=createSign('RSA-SHA256');s.update(h+'.'+p);
      process.stdout.write(h+'.'+p+'.'+s.sign(k.private_key,'base64').replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_'));
    "
    ```
    ```javascript
    // Step 2 (browser javascript_tool): exchange JWT for token, then call API
    const jwt = '<paste JWT from step 1>';
    fetch('https://oauth2.googleapis.com/token', {
      method:'POST', headers:{'Content-Type':'application/x-www-form-urlencoded'},
      body:`grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=${jwt}`
    }).then(r=>r.json()).then(d=>{ window._token = d.access_token; });
    ```
