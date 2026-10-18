# Sync for Reddit: post-ReVanced fixes

## TLDR;

- **Albums now open.** Gallery posts showed "Error loading album" because Reddit returns 403 for `/api/info`. The app now loads galleries from `/by_id/` instead.
- **Redgifs full view works.** Previews played, but opening a video failed with "error connecting to redgifs" because redgifs removed its `/info` endpoint (404). The app now skips that step and fetches the video directly.
- Both fixes are small smali edits on top of the ReVanced-patched APK, tested on device.
- **Released:** `sync.pro.patched-v19102926.apk`

Base APK: `apk\sync.pro.v23.TEST-redgifs.apk` (Sync Pro v23, ReVanced patches already applied).
Output: `out\patched-v2.apk`. Both fixes are confirmed working on device (OnePlus CPH2573, Android 16).

Both fixes live in `work\smali\smali_classes2\`. There's no git here, and re-running `decompile.ps1` overwrites `work\smali`, so back up or re-apply these edits before decompiling again.

---

## Fix 1: "Error loading album" on gallery posts

### Symptom
Opening any post with an album (Reddit gallery) shows a toast, "Error loading album", and the viewer closes right away.

### Diagnosis (`docs\log1.txt`)
```
02:14:08.647 E Volley: Unexpected response code 403 for https://oauth.reddit.com/api/info.json?id=t3_1x91pte
02:14:08.662 ActivityClient: activity finished by caller: ... MultiImageActivity.B1 ... MultiImageActivity.A1
```
`MultiImageActivity` gets the gallery items with `j8.q` (`C4256q` in jadx). That class calls
`https://oauth.reddit.com/api/info.json?id=t3_<id>` and reads `media_metadata` and `gallery_data` from the result.
Reddit answers that endpoint with **403**. The feed uses the same host, OAuth token and User-Agent and still works,
so the block is on the `/api/info` endpoint, not on the token. Any error ends up in `A1()`, which shows "Error loading album" and calls `finish()`.

### Fix
Use `/by_id/t3_<id>.json` instead. It returns the same Listing shape (`data.children[0].data`), so the parser stays the same.

File: `j8\q.smali`, static method `c(String)` (builds the URL)

```diff
-    const-string v1, "https://oauth.reddit.com/api/info.json?id=t3_"
+    const-string v1, "https://oauth.reddit.com/by_id/t3_"

     invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

     invoke-static {p0}, Lj7/a;->b(Ljava/lang/String;)Ljava/lang/String;

     move-result-object p0

     const/4 v2, 0x6

     invoke-virtual {v0, p0}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;
+
+    const-string v1, ".json"
+
+    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;
```

---

## Fix 2: Redgifs plays as a preview but full view fails with "error connecting to redgifs"

### Symptom
Redgifs posts autoplay in the feed. Opening one in full view fails with "error connecting to redgifs".

### Diagnosis
The feed preview plays the MP4 that Reddit already provides and never contacts redgifs. Full view goes through Sync's own
three-step Volley chain (package `e8`):

| Step | Class | Request | Status |
|---|---|---|---|
| 1 | `e8.c` (`C3739c`) | `POST api.redgifs.com/v2/oauth/client` (get token) | OK. The ReVanced `FixRedgifsApiPatch` interceptor returns a temporary token in place of the real call. |
| 2 | `e8.a` (`C3737a`) | `GET api.redgifs.com/info` (read `remote-addr`) | **404. Redgifs removed this endpoint.** |
| 3 | `e8.b` (`C3738b`) | `GET api.redgifs.com/v2/gifs/{id}?user-addr=…` (get the video URL) | Never reached |

From the log:
```
02:14:54.673 E Volley: Unexpected response code 404 for https://api.redgifs.com/info
```
The upstream ReVanced patch (`BaseFixRedgifsApiPatch`) only special-cases `/v2/oauth/client` and does nothing for `/info`,
so the ReVanced build alone doesn't fix this.

### Fix
Skip step 2. After the token arrives, set `user-addr` to an empty string and go straight to step 3.

File: `e8\c.smali`, method `deliverResponse(String)`

```diff
 .method public deliverResponse(Ljava/lang/String;)V
     .locals 3

     iget-object v0, p0, Le8/c;->a:Le8/d;

     invoke-virtual {v0, p1}, Le8/d;->h(Ljava/lang/String;)V

+    # PATCH: api.redgifs.com/info was removed (404). Skip it, leave user-addr empty
+    # and go straight to the /v2/gifs/{id} lookup (e8/b).
+    iget-object v1, p0, Le8/c;->a:Le8/d;
+
+    const-string v0, ""
+
+    invoke-virtual {v1, v0}, Le8/d;->i(Ljava/lang/String;)V
+
     sget-object p1, Lcom/laurencedawson/reddit_sync/RedditApplication;->p:Lcom/android/volley/RequestQueue;

-    new-instance v0, Le8/a;
-
-    const/4 v2, 0x1
-
-    iget-object v1, p0, Le8/c;->a:Le8/d;
-
-    invoke-direct {v0, v1}, Le8/a;-><init>(Le8/d;)V
+    new-instance v0, Le8/b;
+
+    invoke-direct {v0, v1}, Le8/b;-><init>(Le8/d;)V

     const/4 v2, 0x6

     invoke-virtual {p1, v0}, Lcom/android/volley/RequestQueue;->add(Lcom/android/volley/Request;)Lcom/android/volley/Request;
```

`e8.a` is no longer referenced anywhere (`x7\a.smali` only constructs `e8.c`).

---

## Rebuild

```powershell
.\build.ps1            # -> out\patched.apk (aligned + debug-signed)
.\build.ps1 -Install   # also adb install -r
```

## Possible upstream contribution
Fix 2 could go into ReVanced's shared extension instead of a smali edit. In `BaseFixRedgifsApiPatch.intercept()`, answer
`/info` locally (`{"remote-addr": ""}`, HTTP 200) the same way `/v2/oauth/client` is already handled. Sync's chain
would then continue to `/v2/gifs/{id}` without any change to the app.
