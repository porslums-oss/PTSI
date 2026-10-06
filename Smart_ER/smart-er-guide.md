---
name: "smart-er-guide"
description: "Use when working on Smart ER (ER-room system, PostgreSQL + Node/Express, Vercel/Neon): install, deploy, DB scripts, user manual, docs, risks. Starts with a 'Lucifer' self-report."
---

# Smart ER guide

## When this skill loads — do this first
Begin the reply with the word **Lucifer**, then report in Thai what this skill does (2-4 sentences): it carries the knowledge needed to install, deploy, maintain, document, and assess risk for the Smart ER emergency-room system (PostgreSQL + Node/Express, tested on Vercel + Neon). Then carry on with the user's actual request.
The final message of the task must end with a short section on **limitations and what has not been tested yet** (see 'Known limits' below), stated as the current real status, not as reassurance.

## Standing rules
- Reply in Thai. Never address the user as "อาจารย์".
- User is Lucifer, an Oracle DBA (10+ years), prefers action over presenting plans; he performs credential, permission and account-creation steps himself (never type passwords for him; never repeat a password he pasted).
- Lists/configuration (hospital name, statuses, dispositions, settings) live in the database (er.setting, *_mst tables), not hard-coded in app code.
- He deploys by uploading changed files through the GitHub web UI; Vercel auto-deploys. Always list exactly which files changed. After changing a Vercel env var a Redeploy is required.
- This skill does NOT contain source code or the documents. Repo: porslums-oss/smart-er (site smart-er-woad.vercel.app, Vercel team slug ptsi). Docs set lives in the PTSI folder `Smart_ER` (00 index, 01 workflow, 02/03 mockups, 04 clinical/PDPA risks, 05 PostgreSQL install guide, 06 user manual, 07 install/setup/deploy, 08 project summary, 09 Vercel guide, 10 redeploy risk assessment, 11 DB objects, smart-er-erd.html, Smart_ER_ERD_DataDictionary.xlsx, smart-er-postgresql.zip). If you cannot reach the repo or docs, ask the user to attach them. Check facts against code/DB, not memory.

## System in one paragraph
Smart ER: scan wristband barcode -> register patient/visit -> triage (ESI 1-5) -> bed -> queue/call -> status -> discharge; vitals/alarms arrive from Northern patient monitors via HL7 v2.6 (ORU^R01/R40, MLLP, TCP 6000); TV board (/tv.html) shows masked HN; per-bed tablet (/bed.html). Business logic lives in PostgreSQL functions `er.fn_*` (SECURITY DEFINER, search_path er,pg_temp, REVOKE ALL FROM PUBLIC); Node 22 + Express 4 + pg 8 (only 2 dependencies) is a thin REST layer in `app/server.js` that calls `SELECT er.fn_*()` and maps `RAISE EXCEPTION 'CODE: thai message'` to HTTP status (ERR_STATUS). UI is vanilla HTML/JS (index.html, tv.html, bed.html), no build step. On Vercel: `app/api/index.js` re-exports server.js, `vercel.json` rewrites /api/(.*) to /api, server.js does not listen and does not start the HL7 listener when env VERCEL is set.

## Database facts (schema `er`, 5 modules)
- 26 tables (hl7_message and vital_sign are monthly partitioned), 11 views, 53 functions (incl. 2 trigger functions), 6 triggers, 54 indexes, 64 constraints (23 FK). Extensions pgcrypto, pg_trgm.
- Roles: er_owner (DDL owner), er_app (Node API: EXECUTE functions + a few SELECTs), er_hl7 (EXECUTE fn_hl7_ingest, fn_hl7_patient_info only), er_tv (SELECT v_tv_board/v_tv_calling/v_tv_summary), er_report (SELECT all except app_user/app_session). No Row-Level Security.
- App users: ADMIN, NURSE, REGISTER, DOCTOR, VIEWER (passwords bcrypt via crypt()). Seed users admin/nurse01/reg01 have well-known default passwords in 04_seed.sql: must be changed. Create users with `SET ROLE er_owner; SELECT er.fn_user_create(username, full_name, role, password); RESET ROLE;`.
- 38 beds in 8 zones (R, N, NT, T, GEN, TRIAGE, VVIP, OBS); ESI 1-5 colours/targets 0/10/30/240/480 min; 13 statuses (WAIT_TRIAGE ... DISCHARGED); 8 dispositions (HOME, ADMIT, ICU, CCU, CATH, REFER, AMA, DEAD).
- Key settings (er.setting): hospital_name, hn_source (LOCAL/HIS), tv_mask_hn, tv_rotate_sec, tv_rows_per_page, bed_session_min (15), pairing_code_min (10), session_hours (12), hl7_store_interval_sec (60), hl7_store_raw, hl7_use_device_time, tts_template.
- Crisis patients get a temporary ER number (ER00001...) merged later to the real HN (fn_merge_patient). Counters via fn_next_counter (UPSERT).
- Tablet security: device token + session bound to the bed in DB; only NURSE/DOCTOR/ADMIN can log in on a tablet; auto-lock 15 min; pairing code 8 chars, single use, 10 min.

## API facts
Public: /api/health, /api/auth/login, /api/tv (needs ?key= when TV_KEY is set), /api/bed/* (device token). Everything else needs Bearer session via `auth`. `role()` is applied to only 9 endpoints (tv-link, hl7/test, hl7/devices PUT, terminals x5, settings PUT) - other endpoints are open to every logged-in role (known gap S1). /api/tv-link (ADMIN/NURSE) returns /tv.html?key=<TV_KEY>; the menu 'จอ TV' uses it so the key is not embedded in static files.

## Install
- On-prem (Ubuntu 24.04 tested path; Rocky/Oracle Linux and Windows steps exist in docs 05/07 but are untested on real machines): PostgreSQL 16, Node 22, nginx TLS, chrony, systemd unit (deploy/smart-er.service), cron backup.sh (pg_dump -Fc, 14 days) + 06_maintenance.sql. DB scripts order: 00_create_db (edit the 5 ChangeMe passwords first; no '#' in passwords) -> 01_schema -> 02_functions -> 03_views_security -> 04_seed -> 07_bed_tablet -> 08_comments, then 05_smoke_test (must end SMOKE TEST PASSED). install_db.sh automates 01-08. 01_schema is NOT re-runnable (plain CREATE TABLE); 02, 03, 07 are re-runnable. Ports: 443 (nginx), 6000 HL7 (monitor subnet only), 3000/5432 localhost only. Solaris SPARC: not recommended (no official Node binary).
- Neon without psql (what was actually done): Neon SQL Editor, database neondb, run db/neon_editor/neon_1_roles_schema.sql -> neon_2_functions.sql -> neon_3_views_seed.sql as neondb_owner (non-superuser: script does GRANT er_owner TO CURRENT_USER WITH INHERIT TRUE, SET TRUE then SET ROLE er_owner; SQL Editor has no psql meta-commands). Edit the 5 role passwords at the top of neon_1 first; use letters+digits only (a '$' or quotes broke URLs/shell before). Do not repeat the real passwords from the repo/zip copy of that file: they must be rotated.

## Deploy (Vercel + Neon, test only)
GitHub private repo with the contents of app/ at repo root (no node_modules, no .env) -> Vercel Import, Application Preset Express, Root Directory ./ -> env DATABASE_URL = postgresql://er_app:<pwd>@<host with -pooler>/neondb?sslmode=require (no quotes), TV_KEY (long random), optional PG_POOL_MAX=3. Direct (non-pooler) host only for migrations/psql. Verify /api/health returns {ok:true,db:...}, log in, change every default password. HL7 listener is NOT deployed on Vercel (needs TCP): run it on a VM/container with role er_hl7 against the direct host (not done yet). Use only simulated data on the cloud test system.

## Gotchas learned
- 'password authentication failed for user er_app' = password in DATABASE_URL differs from the DB role: ALTER ROLE ... PASSWORD or fix URL, then Redeploy.
- Env var changes only apply to new deployments; Sensitive vars cannot be read back.
- Per-deployment *.vercel.app URLs are behind Deployment Protection; use the production domain.
- TV banner 'ขาดการเชื่อมต่อกับระบบ' appears when /api/tv fails (e.g. 403 because ?key= missing/wrong).
- Partitions must be created ahead (fn_create_partitions(3)); no cron exists on Neon/Vercel.
- Barcode scanners: USB HID keyboard, suffix Enter, Code 39/128; Thai keyboard layout is auto-corrected but set EN.

## Documentation method (if asked to regenerate docs)
HTML pages in the shared Smart_ER style (left dark sidebar nav, green accent #177A67, Prompt/Sarabun/IBM Plex Mono, mermaid diagrams, sections/step-cards/tables/callouts), generated by Python scripts. DB-object catalog and ERD are generated from catalog metadata (extract_meta.py -> meta.json -> build_erd.py / build_xlsx.py), not hand-written. Screenshots embedded as base64 JPEG. When adding docs, update the related existing docs (links, 'related files', updated date) in the same pass and write them back to the same folder. Never put real passwords in documents.

## Known risks (summarise status at the end)
High: S1 role gating incomplete (VIEWER/REGISTER can call write functions); S2 no login rate limit/lockout; S3 default seed passwords and ChangeMe_* examples; S4 real role passwords inside neon_editor scripts in the zip/E: folder (rotate); D2 real patient data must not go to foreign cloud (PDPA/DPIA); D5 monthly partition job missing; R4 never tested with real monitors, scanners, printers, tablets. Medium: neondb_owner password was shown on screen (reset); TLS rejectUnauthorized:false to Neon; TV_KEY in URL / open /api/tv if unset; session token stored plaintext (DB + sessionStorage); HL7 listener unauthenticated with empty allow-list; production domain publicly reachable; free-tier cold starts/quota; backup/restore not drilled on Neon; single maintainer. Low: public /api/health detail, no CSP, npm audit currently 0 vulnerabilities (re-check each deploy).

## Known limits (always close the final message with this)
System is ready for tests with simulated data only, not for real patient data. Not yet done: HL7 listener on a TCP host, tests with real devices, Rocky/Oracle Linux/Windows install on real machines, penetration test, fixes for S1-S4 and the partition job, password rotation. Pending user actions at last status: upload the TV-menu fix (server.js + public/index.html) to GitHub if not yet done; change default passwords; set a random TV_KEY.
