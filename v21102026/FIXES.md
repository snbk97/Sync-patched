# Sync for Reddit: post-ReVanced fixes

## TLDR;
- **Comments work again.** Opening any post's comments showed nothing / an error because Reddit now returns `403` for the comments API when it is called with Sync's installed-app OAuth bearer token. The app now authenticates comment requests with the logged-in **web session cookie** (`reddit_session`), which Reddit still accepts. The cookie is captured per-user during the normal OAuth login, so nothing is hardcoded and the APK is safe to share.
- Includes all fixes from [v20102026](../v20102026/FIXES.md).
- **Released:** `sync.pro.patched-v21102026.apk`

## Fix 4: Comments fail to load (403)

### Symptom
Opening comments on any post, in any subreddit, shows no comments (or an error), even while logged in. It started on 21/10/2026 having worked the day before.

### Diagnosis
logcat (filtered to the app PID) showed the comments request failing with 403, on every subreddit:

```
E Volley : Unexpected response code 403 for
https://oauth.reddit.com/r/<sub>/comments/<id>.json?always_show_media=1&profile_img=true&limit=75
```

Changing the URL shape did not help (dropping `/r/<sub>/`, dropping `.json` — both still 403). Comparing against a working browser request (HAR export + reproducing it with `curl`) isolated the real cause:

- The exact same URL returns **200** from a browser that is **logged in on reddit.com**.
- It is **not** auth via the app token and **not** TLS fingerprinting — the discriminator is the **`reddit_session` web session cookie**:

  | Sent | Result |
  |---|---|
  | `reddit_session` cookie | **200** |
  | `token_v2` cookie | 403 |
  | `reddit_session` as `Bearer` header | 401 |
  | Sync's app OAuth bearer token | 403 |
  | `reddit_session` cookie **+** any `Authorization` bearer | 403 |

So Reddit moved the comments endpoint to require a logged-in web session cookie and to reject the installed-app OAuth bearer token Sync sends. The request must carry `Cookie: reddit_session=…` and **no** `Authorization` header (mirroring the browser). The `200` body is the same `Listing` JSON Sync already parses, so only the authentication had to change — no parser work.

`reddit_session` is a signed JWT with a fixed ~6-month expiry (`sub` = the account). It cannot be refreshed by a token; it is only reissued at (re-)login. So when it lapses (~6 months) the user re-logs in, which re-captures a fresh one.

### Fix
Two smali edits. The cookie is captured per-user from each user's own OAuth login, so there is no secret in the APK.

**1. `smali_classes2\h8\a$b.smali`** — the OAuth login `WebViewClient`. On the successful-login callback (`:cond_1`, after a valid `code` is received), read the `reddit_session` web cookie from the login WebView's `CookieManager` and store it in the existing `OAuthLoginHelper` prefs under key `reddit_cookie`. `.locals` was raised `3 → 6` so the capture uses scratch registers (v3–v5) without clobbering the method's boolean return register (v0) — doing so caused a `VerifyError` and crash on the first attempt.

```smali
    :cond_1
    # PATCH: capture the reddit_session web cookie from the login WebView into prefs,
    # so comment requests (j8/g.getHeaders) authenticate per-user without a hardcoded cookie.
    invoke-static {}, Landroid/webkit/CookieManager;->getInstance()Landroid/webkit/CookieManager;
    move-result-object v3
    const-string v4, "https://www.reddit.com"
    invoke-virtual {v3, v4}, Landroid/webkit/CookieManager;->getCookie(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v3
    const-string v4, "OAuthLoginHelper"
    invoke-static {v4}, Le7/z;->e(Ljava/lang/String;)Landroid/content/SharedPreferences;
    move-result-object v4
    invoke-interface {v4}, Landroid/content/SharedPreferences;->edit()Landroid/content/SharedPreferences$Editor;
    move-result-object v4
    const-string v5, "reddit_cookie"
    invoke-interface {v4, v5, v3}, Landroid/content/SharedPreferences$Editor;->putString(Ljava/lang/String;Ljava/lang/String;)Landroid/content/SharedPreferences$Editor;
    move-result-object v4
    invoke-interface {v4}, Landroid/content/SharedPreferences$Editor;->apply()V
    :try_start_0
    ...
```

**2. `smali_classes2\j8\g.smali`** — the comments request (`j8.g`). Override `getHeaders()` to send only the stored cookie and no bearer. It reads `reddit_cookie` from the `OAuthLoginHelper` prefs and returns a single `Cookie` header.

```smali
.method public getHeaders()Ljava/util/Map;
    .locals 3
    ...
    const-string v0, "OAuthLoginHelper"
    invoke-static {v0}, Le7/z;->e(Ljava/lang/String;)Landroid/content/SharedPreferences;
    move-result-object v0
    const-string v1, "reddit_cookie"
    const-string v2, ""
    invoke-interface {v0, v1, v2}, Landroid/content/SharedPreferences;->getString(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v0
    new-instance v1, Ljava/util/HashMap;
    invoke-direct {v1}, Ljava/util/HashMap;-><init>()V
    const-string v2, "Cookie"
    invoke-virtual {v1, v2, v0}, Ljava/util/HashMap;->put(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
    return-object v1
.end method
```

### Notes
- **Re-login required once** after installing (the cookie store starts empty; the first OAuth login fills it). Each user logs in with their own account, so the APK is shareable.
- **Session expiry (~6 months):** when `reddit_session` lapses, comments 403 again until the user logs in again. A graceful "session expired → re-login" prompt is not yet implemented.
- **Load more comments** (the separate `morechildren` request) was tested and works.
