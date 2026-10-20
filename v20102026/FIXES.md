# Sync for Reddit: post-ReVanced fixes

## TLDR;

- **Random and Random NSFW work again.** Tapping either one showed "Error grabbing random subreddit" because Reddit removed `r/random` and `r/randnsfw` (404). The app now gets a random subreddit from `api.redditrand.com` instead, and doesn't send your Reddit login token to that site.
- Includes all fixes from v19102026 (gallery albums and Redgifs full view).
- **Released:** `sync.pro.patched-v20102026.apk`

Built on the `work\smali` from v19102026. Tested on a device (OnePlus CPH2573, Android 16).

---

## Fix 3: "Error grabbing random subreddit" on Random / Random NSFW

### Symptom
Tapping **Random** or **Random NSFW** (from the actions menu, the toolbar chip, or an `r/random` / `r/randnsfw` link) shows a progress dialog, then the toast "Error grabbing random subreddit", and nothing opens.

### Diagnosis
```
17:39:48.448 E Volley: Unexpected response code 404 for https://oauth.reddit.com/r/random.json
17:39:51.576 E Volley: Unexpected response code 404 for https://oauth.reddit.com/r/randnsfw.json
```
All entry points open the `e9.n1` dialog (`C3802n1` in jadx), which sends one request, `c8.d` (`C1240d`):

- `c(Context, boolean)` builds `https://oauth.reddit.com/` + `r/random.json` or `r/randnsfw.json`.
- `parseNetworkResponse` reads `data.children[0].data.subreddit` and the dialog opens that subreddit.

Reddit removed the special `r/random` and `r/randnsfw` subreddits, so both URLs return **404**, and the dialog's error listener shows the toast. Reddit no longer offers any "random subreddit" endpoint, and upstream ReVanced has no patch for this.

### Fix
Use the third-party `api.redditrand.com` service (the same one a popular old-reddit userscript uses). It returns a random subreddit path:

```
GET https://api.redditrand.com/reddit-runner/rand?nsfw=0  ->  {"url":"/r/chennaimeetups/"}
GET https://api.redditrand.com/reddit-runner/rand?nsfw=1  ->  {"url":"/r/<nsfw sub>/"}
```

**Watch out for this:** `c8.d` extends `i8.a`, whose `getHeaders()` adds `Authorization: bearer <Reddit OAuth token>` to every request (via `singleton.a.e(token)`). Pointing the request at another host without changing anything else would hand your Reddit token to that host. `c8.d` therefore overrides `getHeaders()` to send no headers.

The dialog, the filtered-subreddit retry and navigation stay as they are. NSFW results don't depend on your account's NSFW setting.

File: `smali_classes2\c8\d.smali`

**1. URL**, static method `c(Context, Z)`:

```diff
 .method static c(Landroid/content/Context;Z)Ljava/lang/String;
-    .locals 2
-    new-instance v0, Ljava/lang/StringBuffer;
-    invoke-direct {v0}, Ljava/lang/StringBuffer;-><init>()V
-    invoke-static {p0}, Lcom/laurencedawson/reddit_sync/singleton/SettingsSingleton;->g(Landroid/content/Context;)Ljava/lang/String;
-    move-result-object p0
-    invoke-virtual {v0, p0}, Ljava/lang/StringBuffer;->append(Ljava/lang/String;)Ljava/lang/StringBuffer;
-    if-eqz p1, :cond_0
-    const-string p0, "r/randnsfw.json"
-    invoke-virtual {v0, p0}, Ljava/lang/StringBuffer;->append(Ljava/lang/String;)Ljava/lang/StringBuffer;
-    goto :goto_0
-    :cond_0
-    const-string p0, "r/random.json"
-    invoke-virtual {v0, p0}, Ljava/lang/StringBuffer;->append(Ljava/lang/String;)Ljava/lang/StringBuffer;
-    :goto_0
-    invoke-virtual {v0}, Ljava/lang/StringBuffer;->toString()Ljava/lang/String;
-    move-result-object p0
-    return-object p0
+    .locals 1
+
+    # PATCH: Reddit removed r/random and r/randnsfw (404); use api.redditrand.com instead
+    if-eqz p1, :cond_0
+    const-string v0, "https://api.redditrand.com/reddit-runner/rand?nsfw=1"
+    return-object v0
+
+    :cond_0
+    const-string v0, "https://api.redditrand.com/reddit-runner/rand?nsfw=0"
+    return-object v0
 .end method
```

**2. Parser**, `parseNetworkResponse(NetworkResponse)` (the try/catch and `Response.success` / `ParseError` handling stay as they were):

```diff
-    invoke-virtual {v2, v0}, Lorg/json/JSONObject;->getJSONObject(Ljava/lang/String;)Lorg/json/JSONObject;   # "data"
-    ...getJSONArray("children") -> getJSONObject(0) -> getJSONObject("data")...
-    const-string v1, "subreddit"
-    invoke-virtual {v0, v1}, Lorg/json/JSONObject;->getString(Ljava/lang/String;)Ljava/lang/String;
+    # PATCH: redditrand returns {"url":"/r/<name>/"}; strip "/r/" and "/" to get the subreddit name
+    const-string v1, "url"
+    invoke-virtual {v2, v1}, Lorg/json/JSONObject;->getString(Ljava/lang/String;)Ljava/lang/String;
+    move-result-object v0
+    const-string v1, "/r/"
+    const-string v2, ""
+    invoke-virtual {v0, v1, v2}, Ljava/lang/String;->replace(Ljava/lang/CharSequence;Ljava/lang/CharSequence;)Ljava/lang/String;
+    move-result-object v0
+    const-string v1, "/"
+    invoke-virtual {v0, v1, v2}, Ljava/lang/String;->replace(Ljava/lang/CharSequence;Ljava/lang/CharSequence;)Ljava/lang/String;
+    move-result-object v0
```

**3. New method**, overrides `i8.a.getHeaders()`:

```diff
+.method public getHeaders()Ljava/util/Map;
+    .locals 1
+
+    # PATCH: third-party host; don't send the Reddit OAuth bearer token that i8/a adds
+    new-instance v0, Ljava/util/HashMap;
+    invoke-direct {v0}, Ljava/util/HashMap;-><init>()V
+    return-object v0
+.end method
```

### Caveat
Random now depends on a third-party site. If `api.redditrand.com` goes down or changes its response format, the toast "Error grabbing random subreddit" comes back. Check that URL first.

---

## Earlier fixes

Fix 1 (gallery albums, `j8\q.smali`) and Fix 2 (Redgifs full view, `e8\c.smali`) are unchanged. See [v19102026/FIXES.md](../v19102026/FIXES.md).

## Rebuild

```powershell
.\build.ps1            # -> out\patched.apk (aligned + debug-signed)
.\build.ps1 -Install   # also adb install -r
```
