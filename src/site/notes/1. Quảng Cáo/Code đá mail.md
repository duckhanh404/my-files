---
{"dg-publish":true,"permalink":"/1-quang-cao/code-da-mail/","tags":["tut","QC"]}
---

# Code đá mail

# Nguồn
# Code đá mail
vào bằng link này
```
[https://www.facebook.com/settings?tab=account](https://www.facebook.com/settings?tab=account)
```

```
function AddMail(email){var spinR = require(["SiteData"]).__spin_r;

var spinB = require(["SiteData"]).__spin_b;

var spinT = require(["SiteData"]).__spin_t;

var jazoest = require(["SprinkleConfig"]).jazoest;

var fbdtsg = require(["DTSGInitData"]).token;

var userId = require(["CurrentUserInitialData"]).USER_ID;

var hsi = require(["SiteData"]).hsi;

var pass = "";

  

var url = "https://www.facebook.com/add_contactpoint/dialog/submit/";

  

var data = "jazoest=22134&fb_dtsg=" + fbdtsg + "&next=&contactpoint=" + email + "&__user=" + userId + "&__a=1&__dyn=&__req=1&__be=1&__pc=PHASED%3ADEFAULT&dpr=1&__rev=&__s=&__hsi=" + hsi + "&__spin_r=" + spinR + "&__spin_b=" + spinB + "&__spin_t=" + spinT;

  

fetch(url, {

    method: 'POST',

    body: data,

    headers: {

        'Content-Type': 'application/x-www-form-urlencoded'

    }

}).then(e => e.text()).then(e => {});};AddMail("NguyenToang@outlook.com");
```
# Nguồn
[[1. Quảng Cáo/Quảng cáo\|Quảng cáo]]