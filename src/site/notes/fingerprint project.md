---
{"dg-publish":true,"permalink":"/fingerprint-project/","tags":["CloudFlare","QC","gardenEntry"]}
---



# Thông tin về fingerprint, cách hoạt động...
[Xem thêm tại đây](https://chatgpt.com/share/69406fb5-77c4-800b-9609-7dbd8bee0e19)
# Các lưu ý khi làm ladi page
- Những trang web cần xử lý spam cần có script header trong trang chính và trang thank-you
- Trang chính cũng điền script ở phía dưới
- Khi khách điền form thành công -> chuyển về trang thank-you
- trang thank-you cần có pixel, KHÔNG bật "Trang cảm ơn (thank you page"
![Ảnh màn hình 2025-12-16 lúc 03.34.08.png|350](/img/user/%E1%BA%A2nh%20m%C3%A0n%20h%C3%ACnh%202025-12-16%20l%C3%BAc%2003.34.08.png) 
# Hướng dẫn cách chặn khách
- Kiểm tra IP khách, thời gian khách điền form trong google sheet
- Trong cloudflare KV, và check fingerprint-list để tìm fingerprint chuẩn của người spam
![Ảnh màn hình 2025-12-16 lúc 03.39.21 1.png|700](/img/user/%E1%BA%A2nh%20m%C3%A0n%20h%C3%ACnh%202025-12-16%20l%C3%BAc%2003.39.21%201.png)
- Copy giá trị finger print của khách rồi chuyển sang KV blocked-fingerprint và điền:

| Key                   | Value   |
| --------------------- | ------- |
| fingerprint cần block | blocked |
- Sau khi điền xong thì bấm <span style="background:#fff88f">Add entry</span>
# Code v1
>(không hiệu quả - bị spam fingerprint-list quá nhiều)
 code này điền vào header của trang chính, mục đích dùng để tính toán fingerprint -> gửi fingerprint về KV cloudflare -> nếu trong danh sách blocked-fingerprint thì sẽ chuyển về trang domain-của-bạn.com/spam
## Script web:
```javascript
<!-- script của han -->
<script>
document.documentElement.style.display = "none";
</script>

<script src="https://cdn.jsdelivr.net/npm/@fingerprintjs/fingerprintjs@3/dist/fp.min.js"></script>

<script>
(async () => {
  try {
    const fp = await FingerprintJS.load();
    const result = await fp.get();

    const res = await fetch(
      "https://fp-guard.khanhbuinguyenduc.workers.dev/fp-log",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          fingerprint: result.visitorId,
          path: location.pathname,
          origin: location.origin,
        }),
      }
    );

    const data = await res.json();

    if (data.blocked && data.redirect) {
      location.href = data.redirect;
      return;
    }

    document.documentElement.style.display = "";
  } catch (e) {
    // lỗi fingerprint → cho vào (tránh block nhầm)
    document.documentElement.style.display = "";
  }
})();
</script>
```

## Worker code:
> Phần này là code javascript trong worker của cloudflare.
```javascript
const corsHeaders = {

"Access-Control-Allow-Origin": "*",

"Access-Control-Allow-Methods": "POST, OPTIONS",

"Access-Control-Allow-Headers": "Content-Type",

"Access-Control-Max-Age": "86400",

};

  

function formatTimeVN(ms) {

return new Intl.DateTimeFormat("vi-VN", {

timeZone: "Asia/Ho_Chi_Minh",

year: "numeric",

month: "2-digit",

day: "2-digit",

hour: "2-digit",

minute: "2-digit",

second: "2-digit",

hour12: false,

})

.format(new Date(ms))

.replace(",", "");

}

  

export default {

async fetch(request, env) {

const url = new URL(request.url);

  

/* ================= CORS PREFLIGHT ================= */

if (request.method === "OPTIONS") {

return new Response(null, {

status: 204,

headers: corsHeaders,

});

}

  

/* ================= FP LOG API ================= */

if (url.pathname === "/fp-log" && request.method === "POST") {

let data;

try {

data = await request.json();

} catch {

return new Response("Invalid JSON", {

status: 400,

headers: corsHeaders,

});

}

  

const { fingerprint, path = "/", origin } = data;

  

if (!fingerprint || !origin) {

return new Response("Missing fingerprint or origin", {

status: 400,

headers: corsHeaders,

});

}

  

// Chuẩn hóa origin (chỉ lấy https://domain)

let siteOrigin;

try {

siteOrigin = new URL(origin).origin;

} catch {

return new Response("Invalid origin", {

status: 400,

headers: corsHeaders,

});

}

  

const ip =

request.headers.get("cf-connecting-ip") ||

request.headers.get("x-forwarded-for")?.split(",")[0] ||

"unknown";

  

const timestamp = Date.now();

  

/* ========== CHECK BLOCK ========== */

const blockInfo = await env.FP_BLOCKED.get(fingerprint);

if (blockInfo) {

return new Response(

JSON.stringify({

blocked: true,

redirect: `${siteOrigin}/spam`,

}),

{

status: 403,

headers: {

...corsHeaders,

"Content-Type": "application/json",

},

}

);

}

  

/* ========== SAVE LOG ========== */

const logKey = `${fingerprint}:${timestamp}`;

const logValue = {

fingerprint,

ip,

timestamp,

time_readable: formatTimeVN(timestamp),

origin: siteOrigin,

path,

};

  

await env.FP_LIST.put(logKey, JSON.stringify(logValue), {

expirationTtl: 60 * 60 * 24 * 30, // 30 ngày

});

  

return new Response(

JSON.stringify({ blocked: false }),

{

headers: {

...corsHeaders,

"Content-Type": "application/json",

},

}

);

}

  

return new Response("Not Found", {

status: 404,

headers: corsHeaders,

});

},

};
```

# Code v2
> Đã chỉnh sửa để chỉ lưu fingerprint khi khách điền form thành công
> Khi khách vào trang chính sẽ tính toán fingerprint để xem fingerprint có nằm trong danh sách blocked-fingerprint không, nếu có thì sẽ chuyển về trang có tên-miền.com/spam
> Trong trường hợp khách điện form thành công thì sẽ chuyển về trang thank-you và fingerprint của khách sẽ được chuyển về fingerprint-list.
## Code worker cloudflare
```javascript
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type",
  "Access-Control-Max-Age": "86400",
};

function formatTimeVN(ms) {
  return new Intl.DateTimeFormat("vi-VN", {
    timeZone: "Asia/Ho_Chi_Minh",
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    hour12: false,
  })
    .format(new Date(ms))
    .replace(",", "");
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    /* ========== CORS PREFLIGHT ========== */
    if (request.method === "OPTIONS") {
      return new Response(null, { status: 204, headers: corsHeaders });
    }

    /* =====================================================
       1️⃣ CHECK BLOCK – KEY = fingerprint
    ===================================================== */
    if (url.pathname === "/fp-check" && request.method === "POST") {
      let data;
      try {
        data = await request.json();
      } catch {
        return new Response("Invalid JSON", { status: 400, headers: corsHeaders });
      }

      const { fingerprint, origin } = data;
      if (!fingerprint || !origin) {
        return new Response("Missing fingerprint or origin", {
          status: 400,
          headers: corsHeaders,
        });
      }

      const isBlocked = await env.FP_BLOCKED.get(fingerprint);

      return new Response(
        JSON.stringify({
          blocked: !!isBlocked,
          redirect: isBlocked ? `${origin}/spam` : null,
        }),
        {
          headers: {
            ...corsHeaders,
            "Content-Type": "application/json",
          },
        }
      );
    }

    /* =====================================================
       2️⃣ LOG FINGERPRINT – giữ nguyên
    ===================================================== */
    if (url.pathname === "/fp-log" && request.method === "POST") {
      let data;
      try {
        data = await request.json();
      } catch {
        return new Response("Invalid JSON", { status: 400, headers: corsHeaders });
      }

      const { fingerprint, path = "/", origin } = data;
      if (!fingerprint || !origin) {
        return new Response("Missing fingerprint or origin", {
          status: 400,
          headers: corsHeaders,
        });
      }

      let siteOrigin;
      try {
        siteOrigin = new URL(origin).origin;
      } catch {
        return new Response("Invalid origin", { status: 400, headers: corsHeaders });
      }

      const ip =
        request.headers.get("cf-connecting-ip") ||
        request.headers.get("x-forwarded-for")?.split(",")[0] ||
        "unknown";

      const timestamp = Date.now();

      const logKey = `${siteOrigin}:${fingerprint}:${timestamp}`;
      const logValue = {
        fingerprint,
        ip,
        origin: siteOrigin,
        path,
        timestamp,
        time_readable: formatTimeVN(timestamp),
      };

      await env.FP_LIST.put(logKey, JSON.stringify(logValue), {
        expirationTtl: 60 * 60 * 24 * 30,
      });

      return new Response(
        JSON.stringify({ ok: true }),
        {
          headers: {
            ...corsHeaders,
            "Content-Type": "application/json",
          },
        }
      );
    }

    return new Response("Not Found", {
      status: 404,
      headers: corsHeaders,
    });
  },
};

```
## Script web thank-you
```javascript
<script src="https://cdn.jsdelivr.net/npm/@fingerprintjs/fingerprintjs@3/dist/fp.min.js"></script>

<script>
(async () => {
  try {
    const fp = await FingerprintJS.load();
    const { visitorId } = await fp.get();

    fetch(
      "https://fp-guard.khanhbuinguyenduc.workers.dev/fp-log",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          fingerprint: visitorId,
          origin: location.origin,
          path: location.pathname,
        }),
      }
    );
  } catch {}
})();
</script>
```
## Script web chính
```javascript
<style>
body { visibility: hidden; }
</style>

<script src="https://cdn.jsdelivr.net/npm/@fingerprintjs/fingerprintjs@3/dist/fp.min.js"></script>

<script>
(async () => {
  try {
    const fp = await FingerprintJS.load();
    const { visitorId } = await fp.get();

    const res = await fetch(
      "https://fp-guard.khanhbuinguyenduc.workers.dev/fp-check",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          fingerprint: visitorId,
          origin: location.origin,
        }),
      }
    );

    const data = await res.json();

    if (data.blocked && data.redirect) {
      location.replace(data.redirect);
      return;
    }
  } catch (e) {
    // lỗi thì cho vào (tránh block nhầm)
  }

  document.body.style.visibility = "visible";
})();
</script>
```
# Nguồn
