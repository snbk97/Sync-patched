# Sync: post-ReVanced fixes

## TLDR;
- **Comments work again.**
- **Thumbnail media preview in comments works again** (`-2` build). Thumbnail URLs arrived HTML-escaped (`&amp;`), which broke the signed `preview.redd.it` link (403). They're now unescaped.
- Includes all fixes from [v20102026](../v20102026/FIXES.md).
- **Released:** `sync.pro.patched-v21102026-2.apk` (latest, Fix 4 + Fix 5). The original `sync.pro.patched-v21102026.apk` (Fix 4 only) is kept.

## Fix 4: Comments fail to load (403)

### Symptom
Opening comments on any post, in any community, shows no comments, even while logged in. It started on 21/10/2026 having worked the day before.

### Diagnosis
logcat (filtered to the app PID) showed the comments request failing with 403 on every community:

```
E Volley : Unexpected response code 403 for
https://<api-host>/r/<sub>/comments/<id>.json?always_show_media=1&profile_img=true&limit=75
```


### Notes
- This relies on how the server matches paths and santization rules

## Fix 5: Thumbnail media preview missing in comments

### Symptom
With *Comment view → Post media preview size → Thumbnail*, the post header in comments shows an empty space next to the title instead of the thumbnail. *Cropped height* and *Full height* work.

### Diagnosis
Glide logged a 403 for the thumbnail:

```
W Glide : Load failed for https://preview.redd.it/<id>.jpeg?width=140&amp;height=140&amp;crop=1:1,smart&amp;auto=webp&amp;s=<sig> with size [240x240]
W Glide : w2.e(Forbidden, status code: 403)
```

Reddit now serves thumbnails as signed `preview.redd.it` URLs, and the JSON HTML-escapes them. The app unescapes `preview_url` (which Cropped and Full height use) but not `thumbnail`. The literal `&amp;` breaks the signature, so Reddit returns 403. The same URL with `&` returns 200.

### Fix
`smali_classes2\n7\c.smali`, `a(JSONObject)`: run the thumbnail through the app's own unescape helper `hc/p.a` before returning it.

```diff
     :cond_4
     const/4 v3, 0x4

+    # PATCH: unescape "&amp;" in signed preview.redd.it thumbnail URLs (403 otherwise)
+    invoke-static {p0}, Lhc/p;->a(Ljava/lang/String;)Ljava/lang/String;
+
+    move-result-object p0
+
     return-object p0
```

Posts already cached by the app keep the old escaped URL until they are refreshed.
