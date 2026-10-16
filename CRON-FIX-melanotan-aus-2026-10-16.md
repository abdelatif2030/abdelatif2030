# WP-Cron recovery on melanotan-aus.com (Cloudways), 2026-10-16

Scope: fixing the stalled WordPress scheduler on the live store. Nothing in this document covers email-provider setup (SES / SendGrid), which is a separate, blocked topic (see `HANDOFF-fluentcrm-ses-project.md`, section 0).
No passwords, API keys or tokens are stored here. All times are UTC unless noted (the user's local time was UTC+3).

---

## 1. Summary

| | |
|---|---|
| Site | https://melanotan-aus.com (WordPress 7.1.3, WooCommerce 11.2.1, PHP 8.2.34, nginx) |
| Host | Cloudways server **Melanotan-Aus-Server** (id 1605291), live app **Melanotan-Aus** (app id 6305766, folder `psukegvwxg`) |
| App path | `/home/1605291.cloudwaysapps.com/psukegvwxg/public_html` |
| Problem | WP-Cron had effectively stopped about **Sep 24**. An Oct 10 export showed **214 of 390** events overdue (oldest Sep 23, some by 17 days). On Oct 16 about **123** were overdue. Scheduled backups (UpdraftPlus), WooCommerce's Action Scheduler queue, Jetpack sync, plugin housekeeping and FluentCRM's own tasks were not running. |
| Root cause | A real server cron job did not exist in the crontab (the Basic-tab job never took effect). `wp-cron.php` invoked from cron started, exited 0 and processed no events (output was only a Redis `NOAUTH` message). Running events through **WP-CLI** worked. |
| Fix | One line in the Cloudways **Advanced** crontab that runs `wp cron event run --due-now` every 5 minutes, with `flock` (no overlapping runs), `timeout 600` (a hung event cannot jam the queue) and a timestamp in a log file. |
| Result (11:30 UTC) | Runs logged at 11:20, 11:25, 11:30 (`Success: Executed a total of 8 cron events.`). Overdue events fell from 123 to **3** (only hooks that are always due between ticks). UpdraftPlus backup completed. Disk 42%, load ~0.8. |

---

## 2. Final configuration

Cloudways > Applications > **Melanotan-Aus** > **Cron Job Management** > **Advanced** tab. Keep exactly these lines (and a trailing blank line):

```
PATH=/var/cw/php_versions/8.2/bin:/usr/local/bin:/usr/bin:/bin
WP_CLI_PHP=/usr/bin/php8.2
*/5 * * * * flock -n /home/1605291.cloudwaysapps.com/psukegvwxg/tmp/wp-cron2.lock -c "date -u; cd /home/1605291.cloudwaysapps.com/psukegvwxg/public_html && timeout 600 /usr/local/bin/wp cron event run --due-now" >> /home/1605291.cloudwaysapps.com/psukegvwxg/tmp/wp-cron.log 2>&1
```

What each part does:
- `flock -n .../wp-cron2.lock`: if a previous run still holds the lock, this tick is skipped instead of starting a second copy.
- `date -u`: stamps every run in the log.
- `timeout 600`: kills a single run after 10 minutes so one hung event cannot block later ticks.
- `/usr/local/bin/wp cron event run --due-now`: runs every due event directly (no HTTP loopback, no dependency on page loads).
- Output goes to `/home/1605291.cloudwaysapps.com/psukegvwxg/tmp/wp-cron.log` (small; remove or rotate it later).

Notes:
- **Do not use the Basic tab form** for this. In the Basic form the Command box already has the prefix `/psukegvwxg/public_html/` and Type PHP, so only `wp-cron.php` is entered; that job did not end up in the real crontab.
- Because the job runs every 5 minutes, the every-minute hooks (FluentCRM `fluentcrm_scheduled_minute_tasks`, Action Scheduler `action_scheduler_run_queue`, FlyingPress watchdog) are processed once per tick. Change `*/5` to `*` only when FluentCRM sending starts.
- `DISABLE_WP_CRON` has **not** been added to `wp-config.php`. Optional: add `define( 'DISABLE_WP_CRON', true );` above `/* That's all, stop editing! */` after a stable day (download a copy of `wp-config.php` first).

---

## 3. Timeline of what was done

1. **Backup first.** A manual Cloudways backup (app level) was taken before any change. UpdraftPlus's own automatic backup was found to be incomplete later.
2. **Site Health baseline (~09:20).** "A scheduled event has failed" (`wp_scheduled_auto_draft_delete`), 13 inactive plugins, 10 pending updates, PHP 8.2.34, autoloaded options 874 KB. WP Crontrol (1.22.0) was already installed and showed **287 events, 205 paused, 215 duplicated**. FluentCRM and Pro 3.2.6 were already active; FluentSMTP installed but inactive; WP Mail SMTP active.
3. **First cron job (Basic tab, ~09:21).** Cloudways > app 6305766 > Cron Job Management > Add New Cron Job: schedule every 5 minutes, Type PHP, Command `wp-cron.php`. (First attempt failed validation because a full path was typed after the auto-added prefix.) After 10+ minutes the overdue events had **not** cleared.
4. **SSH diagnostics (Cloudways > Server > Master Credentials > Launch SSH Terminal):**
   - path exists, CLI PHP 8.2.34, `wp` at `/usr/local/bin/wp`;
   - `grep -i CRON wp-config.php`: nothing (no `DISABLE_WP_CRON`);
   - `wp cron test`: "WP-Cron spawning is working as expected";
   - `doing_cron` lock not set;
   - master user's crontab empty; no `wp-cron` process; no PHP error log at the expected path;
   - **123 events due now**;
   - two harmless core events were run by hand (`wp_privacy_personal_data_cleanup_requests`, `recovery_mode_clean_expired_keys`) and rescheduled correctly, proving WordPress can run and reschedule events.
5. **Customer-email risk review before running the backlog.** The due list contained customer-facing hooks (`wlr_expire_email`, `wlr_change_expire_status`, `sendEmail`, `active_cron_to_*`, `ph_live_tracking_status_action_cron`, `ivole_send_reminder`). In WP Crontrol (Tools > Cron Events) these were **paused**. Identification:
   - `sendEmail` = WooCommerce Advanced Reporting & Statistics Pro, `OrderProcessorAdmin::sendEmail()`, a weekly admin report.
   - `active_cron_to_check_order_and_send_email` / `active_cron_to_find_order_where_paid_off_9d` = defined in the inactive `enfold-child` theme's `functions.php`, action shown as "None" (no callback).
   - `ivole_send_reminder` = CusRev (Customer Reviews for WooCommerce) review-request emails; **205 were already paused** before the work began.
   - Action Scheduler pending queue (checked with `wp db query` on `actionscheduler_actions`): only housekeeping (WooCommerce, WPForms, Stripe cache), 2 `airwallex_check_pending_transactions`, 1 FluentCRM `every_minute_tasks`, and admin-only items. Nothing customer-facing.
6. **Partial drain (09:31 to 10:25).** Some events cleared (Sep 24 batch, then Sep 28 items) but slowly. Manual one-off runs: `wp cron event run action_scheduler_run_queue` and `wp cron event run jetpack_sync_cron jetpack_sync_full_cron jetpack_clean_nonces wc_admin_process_orders_milestone`. UpdraftPlus files showed activity until 09:47 only: the backup was **incomplete** (database + plugins zip present, `updraft_backup_resume` events overdue).
7. **Does the Cloudways job fire?** A probe event (`wp cron event schedule cron_probe now`) and a "recurring hook" test (Jetpack hooks showing a short time ahead after a manual run) both showed "now": the job was **not** firing.
8. **The crontab was empty.** Opening the **Advanced** tab showed only the two environment lines (`PATH=...`, `WP_CLI_PHP=...`) and **no cron line**. The Basic-tab job was not in the real crontab.
9. **Advanced line v1 (~10:41).** Added `*/5 * * * * (date -u; /usr/bin/php8.2 .../public_html/wp-cron.php; echo "exit $?") >> .../tmp/wp-cron.log 2>&1`. The log then showed runs at 10:45, 10:50, 10:55, 11:00 each ending `exit 0` with only `[FlyingPress get]: NOAUTH Authentication required.`, yet the overdue hooks stayed "now". **Conclusion: cron fires, but `wp-cron.php` from CLI processes nothing.** (The Redis `NOAUTH` message points at the object cache; not fixed here.)
10. **Advanced line v2 (~11:02).** Replaced with `flock + wp cron event run --due-now` (no timeout). First run started 11:05.
    - It executed about 47 overdue `ivole_send_reminder` events in about 5 ms each (see section 5), plus housekeeping, and `updraft_backup_resume` (34.7 s).
    - It then **hung on a later event**: process `php /usr/local/bin/wp cron event run --due-now` (PID 1118379) stayed in sleep state for 15+ minutes and held the `flock`, so every later tick was skipped. It could not be killed from the master login (`kill: Operation not permitted`; it belongs to the app user; Cloudways support can terminate it).
11. **Advanced line v3 (~11:16), the final one.** New lock file name `wp-cron2.lock` (the stuck process keeps the old lock), plus `timeout 600` and `date -u`. The old line was **replaced, not kept beside it**.
12. **Verification.**
    - 11:20 run: `Success: Executed a total of 23 cron events.`
    - 11:30 check: log timestamps at 11:20, 11:25, 11:30; last run `Executed a total of 8 cron events`; overdue (excluding paused review reminders) = **3** (`fluentcrm_scheduled_five_minute_tasks`, `fluentcrm_scheduled_minute_tasks`, `hippoo_bi_daily_aggregate`); no `wp cron event` process left running; no `updraft_backup_resume` pending (backup finished); `updraft_backup_database` next in ~15 h, `updraft_backup` in ~2 d 15 h.
    - Server health at 11:20: disk 59 GB / 24 GB used (42%), load 0.78, 16 GB RAM (10 GB available), uptime 72 days, `wp-content/updraft` 4.9 GB.
    - Site Health at 11:32: the "scheduled event has failed" item became the milder "a scheduled event is late" (the late check flags events more than ~10 min past due; expected to clear).

---

## 4. Verification and monitoring commands (all read-only)

Run in the Cloudways SSH terminal (Server > Master Credentials > Launch SSH Terminal).

```
cd /home/1605291.cloudwaysapps.com/psukegvwxg/public_html
date -u
# runs so far and the last result
grep -E "^[A-Z][a-z]{2} [A-Z][a-z]{2} +[0-9]+ [0-9:]+ UTC" /home/1605291.cloudwaysapps.com/psukegvwxg/tmp/wp-cron.log | tail -6
tail -4 /home/1605291.cloudwaysapps.com/psukegvwxg/tmp/wp-cron.log
# is a run in progress (or hung)?
ps aux | grep "[w]p cron event" | head -3
# overdue events, excluding the paused review reminders
wp cron event list --fields=hook,next_run_relative --format=csv 2>/dev/null | grep -v ivole | grep -c ",now"
wp cron event list --fields=hook,next_run_relative --format=csv 2>/dev/null | grep -v ivole | grep ",now"
# recurring hooks should show a short time ahead (under 5 minutes) most of the time
wp cron event list --fields=hook,next_run_relative --format=csv 2>/dev/null | grep -E "^jetpack_sync_cron|^fluentcrm_scheduled_five_minute_tasks|^action_scheduler_run_queue|^updraft_backup"
# disk, load, memory
df -h /home | head -3; du -sh wp-content/updraft; uptime; free -m | head -2
```

Healthy means: a new timestamp every 5 minutes, `Success: Executed a total of N cron events.`, overdue of 0 to 3, no stuck process older than 10 minutes.

If something goes wrong:
- No new log lines: open Cloudways > Cron Job Management > Advanced and confirm the three lines (plus blank line) are still there and saved.
- A stuck `wp cron event run` process: with the `timeout 600` it ends on its own within 10 minutes; if it ever does not, Cloudways support must terminate it (give them the PID and "owner = app user").
- Rollback: delete the `*/5 ...` line in the Advanced box and save (WordPress then falls back to its normal page-load scheduler). Remove `.../tmp/wp-cron2.lock` and `.../tmp/wp-cron.log` if desired.

---

## 5. What "paused" really does (important)

WP Crontrol's **Pause this hook** removes the actions attached to a hook. The event still fires on schedule, but nothing happens. Observed in the log:
- Paused recurring hooks (`wlr_expire_email`, `wlr_change_expire_status`, `ph_live_tracking_status_action_cron`) show "Executed ... in ~0.012s" each tick: a harmless no-op, and they keep rescheduling.
- Paused **one-off** events are **consumed and deleted** when they come due. About **47** overdue `ivole_send_reminder` (CusRev review requests, Oct 12 to 15) were discarded at about 11:05 UTC (each ran in ~5 ms, far too fast to be a real email). The remaining **158** will be discarded as their dates arrive (3 at 12:00 UTC on Oct 16, then daily).
- **Decision for the client:** whether those review requests should still be sent. Do **not** resume `ivole_send_reminder` without her say-so (it would email every queued customer at once).
- Paused on purpose (leave paused): `ivole_send_reminder`, `wlr_expire_email`, `wlr_change_expire_status`, `ph_live_tracking_status_action_cron`, `active_cron_to_check_order_and_send_email`.
- Not confirmed paused but harmless: `sendEmail` (weekly admin report), `active_cron_to_find_order_where_paid_off_9d` (no callback); both ran once before the pauses.

---

## 6. Side findings (not part of the cron fix)

- **Redis "NOAUTH Authentication required."** appears in every WP-CLI call (`[FlyingPress get]`). `wp_using_ext_object_cache()` returns true, so an object-cache drop-in is active, yet one call is unauthenticated. For Cloudways/client to look at (Redis credentials are in the app's Access Details; do not copy them anywhere).
- **FluentCRM state:** no automations (funnels table empty), no funnel subscribers, **no queued emails**. The campaigns table holds two sequence shells marked "(DRAFT, do not start)" ("Welcome Series" with 3 emails, "Abandoned Cart" with 1 email) and their 4 "published" sequence emails. Content not reviewed.
- **Orders:** HPOS off; 2 orders in "Pending payment" on Oct 16 (Airwallex status checks had been overdue). Worth a look by the client.
- **Site Health items for the client:** autoloaded options 874 KB (critical), PHP 8.2.34 (8.3 recommended), 13 inactive plugins, ~10 pending plugin updates (not touched), outdated WooCommerce template overrides, PHP opcode cache full (hit rate ~59%, Cloudways can raise it), 13,099 WordPress users (bot-looking sign-ups seen in Drip too).
- **Security:** the Site Health **Info** tab contains a Jetpack blog token. Do not paste the Info tab into chats or notes. The client may want to disconnect and reconnect Jetpack at the end. Also tell her to change every password shared during the work (AWS, Drip, WordPress, Cloudflare, Cloudways) and remove your access.

---

## 7. Mistakes and corrections made along the way (so they are not repeated)

- The Basic-tab cron form does not accept a full `php /home/...` command; the box already adds `/psukegvwxg/public_html/` and the PHP type. An earlier note assumed `/home/master/applications/...` paths; the real path on this server is `/home/1605291.cloudwaysapps.com/psukegvwxg/public_html`.
- An empty `crontab -l` for the master user did not mean the job was missing; the Cloudways job lives in the app's Advanced editor. The real signal was the empty Advanced box plus no log file.
- `wp-cron.php` from CLI exiting 0 does **not** mean events were processed; check the overdue count and the recurring hooks' next-run times.
- `flock -n` without a `timeout` lets one hung event block the cron forever. Always pair them.
- Paused hooks are not frozen: they run as no-ops and one-off events are discarded (section 5).
- Two `wp cron event run --due-now` attempts were started and cancelled with Ctrl+C early on, before the safety checks; they may have run part of the backlog.

---

## 8. Open items

1. Client: decide on the 158 queued review requests, the paused loyalty and live-tracking jobs, and the 2 pending orders; confirm no customer received an unexpected email around 11:05 UTC on Oct 16 (the 5 ms run times indicate none).
2. Cloudways: Redis NOAUTH; terminate PID 1118379 if it reappears; consider raising the PHP opcache memory.
3. Optional: change the job to every minute when FluentCRM sending starts; add `DISABLE_WP_CRON` after a stable day; remove `wp-cron.log` / `wp-cron2.lock` at handover.
4. UpdraftPlus: confirm in Settings > UpdraftPlus Backups > Existing backups that the Oct 16 backup set is complete (database + plugins + themes + uploads + others); keep the manual Cloudways backup until the site has been stable for a couple of days.
5. Handover: change shared passwords, remove your access, hand over this document.
