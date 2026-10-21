# Sync: post-ReVanced fixes

## TLDR;
- **Comments work again.**
- Includes all fixes from [v20102026](../v20102026/FIXES.md).
- **Released:** `sync.pro.patched-v21102026.apk`

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