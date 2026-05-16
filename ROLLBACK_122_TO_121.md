## Rollback Plan: Build 122 -> Build 121

### Verified Pre-conditions (2026-05-15)
- OTA repo: github.com/GigaGoods/gigarep-ota (master)
- Build 121 commit: 690f3cc ("Build 121") — May 15 09:25 EDT
- Build 122 commit: 850df4f ("Build 122") — May 15 13:25 EDT (HEAD)
- Build 121 IPA size in git: 13,157,435 bytes (recoverable via git checkout)
- Build 122 IPA size on disk: 13,157,477 bytes (only 42-byte diff between builds)
- manifest.plist: identical between 121 and 122 (both bundle-version 1.50). No manifest edit required during rollback.
- IPA pulled via HTTPS from gigagoods.github.io (HTTP/2 200, etag 6a07573a-378). Cache-Control max-age=600 -> 10 min CDN TTL.

### Rollback Sequence (estimated 3–5 min from decision to first device update)

1. Restore build 121 IPA from git history (no need to download from elsewhere):
   ```
   cd /tmp/gigarep-ota
   git fetch origin master
   git checkout 690f3cc -- GIGARep.ipa
   # Sanity: must be 13,157,435 bytes
   stat -f %z GIGARep.ipa
   ```

2. Commit the rollback as a NEW commit (do NOT force-push; preserve history):
   ```
   git add GIGARep.ipa
   git commit -m "rollback: build 122 -> build 121 (IPA from 690f3cc)"
   git push origin master
   ```

3. Wait for GitHub Pages republish (typical 30–90s) and verify:
   ```
   curl -sI "https://gigagoods.github.io/gigarep-ota/GIGARep.ipa" | grep -i etag
   # New etag != 6a07573a-378 confirms CDN refreshed
   ```

4. (Optional) Invalidate CDN caches by adding a cache-busting query if reps already hit it:
   - manifest.plist URL is hardcoded in iOS app — cannot change without app update.
   - Cache-Control max-age=600 means CDN auto-refreshes within 10 min. No manual action needed.

5. Force re-install on the fleet — TWO PATHS:

   PATH A — Patches are working (foreground/silent-push register handlers do their job):
   - Send a silent APNs push to all 83 devices to wake them.
   - On wake, the new register handler will pull manifest.plist and download IPA. (However: since manifest bundle-version is still 1.50 for both 121 and 122, iOS may NOT re-prompt for install. This is a real risk — see PATH B.)

   PATH B — Mosyle MDM force-install (RECOMMENDED, deterministic):
   - Open Mosyle -> Apps -> GIGARep -> Force Update.
   - Mosyle will push InstallApplication MDM command to each device.
   - Estimated wall-clock for full fleet: 10–20 min depending on device check-in cadence.
   - This is the same path used to push build 122 originally, so the runbook is known.

### Rollback time budget
- Decision -> IPA restored + pushed: 1 min
- GitHub Pages republish: ~60s
- Mosyle queue + device fetch: 10–20 min
- Total: ~15–25 min until full fleet on 121

### Post-rollback verification
- Spot-check 3 reps via /admin tab: `/api/fleet/devices` -> expect build 121 in user-agent / version field
- Tail `/voice` webhook logs for 10 min -> expect 200s, no `application error`
- Confirm at least one inbound call completes end-to-end

### Hard constraints
- DO NOT push manifest.plist change in this rollback (no diff needed; pushing identical file is fine but unnecessary).
- DO NOT delete the build 122 commit from history. Keep it for forensic review.
- DO NOT restart gigarep-call-recorder during rollback — server is iOS-agnostic.
- Per CLAUDE.md HARD RULE: deploy is permission-gated. Chaz must give explicit go-ahead before steps 2 and 5.
