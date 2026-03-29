MASTER HANDOVER NOTE — DAILY LETTER / OSAP / DAILYCUT INFRASTRUCTURE SPLIT

Date: March 28, 2026
Status: Major split completed successfully

1. WHAT WE ACHIEVED

We successfully split the old multi-app backend into three independent runtime layers:

Current live architecture
Daily Letter
Domain: dailyletter.tv
Server: 54.164.1.136
Role: main publication backend only
OSAP
Domain: osap.ai
Server: 98.92.67.160
Role: OSAP standalone backend only
DailyCut
Domain: dailycut.ai
Server: 44.200.42.187
Role: DailyCut standalone backend only

This means:

OSAP is no longer served from the old Daily Letter EC2
DailyCut is no longer served from the old Daily Letter EC2
Daily Letter no longer has OSAP or DailyCut PM2 processes running locally
Each service now has its own EC2, own PM2, own nginx, own SSL, own domain
2. SSH COMMANDS — DO NOT LOSE THESE

Use the same SSH key for all three:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
Meaning
54.164.1.136 = old main Daily Letter EC2
98.92.67.160 = OSAP EC2
44.200.42.187 = DailyCut EC2
3. HOW WE WORK TOGETHER GOING FORWARD

The next person must follow this workflow exactly:

Required workflow
Always back up first
I send command
User runs command
User pastes raw output back
Then surgical instruction is given
Then verify
Then backup after success
Never do this
Never freestyle edits directly in production without a backup
Never assume a route, env var, or file path without checking it first
Never mass-rewrite before inspecting exact code
Never skip verification after PM2/nginx/env changes
Standard working pattern
inspect
backup
patch surgically
verify syntax
restart only target service
verify local curl
verify external curl
save PM2 state
4. HIGH-LEVEL STATUS OF EACH SERVER
A. DAILY LETTER SERVER
Host

54.164.1.136

SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
Current role

This server should now run Daily Letter only.

Confirmed PM2 state

Only these processes should remain:

dailyletter-main
dailyletter-publish-worker
Confirmed removed
osap-app removed
dailycut-app removed
Current nginx role

Should no longer actively proxy:

osap.ai
dailycut.ai
127.0.0.1:5002
127.0.0.1:5003

Only Daily Letter should be active here.

Daily Letter admin note

From prior work, the admin base path was moved from /admin to:

/ironclad-2025

That is the known admin route base currently remembered from prior work.

Immediate recommended cleanup still to run on old server

This removes leftover inactive nginx backup files and notes:

sudo rm -f /etc/nginx/conf.d/osap.ai.conf.bak*
sudo rm -f /etc/nginx/conf.d/_split-ready-notes.conf
sudo nginx -t
sudo systemctl reload nginx
Verify old server is clean
pm2 list
sudo grep -Rni "osap.ai\|dailycut.ai\|127.0.0.1:5002\|127.0.0.1:5003" /etc/nginx 2>/dev/null || true
B. OSAP SERVER
Host

98.92.67.160

SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
Current role

This server runs OSAP only.

Runtime
PM2 process: osap-app
App port: 5002
Nginx proxies osap.ai and www.osap.ai to 127.0.0.1:5002
SSL is active and working
Verified working checks

Internal:

curl -I http://127.0.0.1:5002
curl -I -H "Host: osap.ai" http://127.0.0.1

External:

curl -I https://osap.ai
curl -I https://www.osap.ai
OSAP code location on OSAP EC2
/home/ec2-user/osap-standalone
Main entry
/home/ec2-user/osap-standalone/apps/osap/server.js
Current OSAP env file
/home/ec2-user/osap-standalone/.env.osap
PM2 start pattern
cd /home/ec2-user/osap-standalone
ENV_FILE=/home/ec2-user/osap-standalone/.env.osap pm2 start apps/osap/server.js --name osap-app
pm2 save
PM2 boot persistence
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
pm2 save
Nginx / host routing check
curl -I -H "Host: osap.ai" http://127.0.0.1

If healthy, it should redirect HTTP to HTTPS or return OSAP depending on the final nginx block.

OSAP admin/backend note

I do not want to guess the admin entry for OSAP without checking. The next person should verify exact admin/auth paths by inspecting routes, middleware, and frontend links on the OSAP standalone server.

Use:

cd /home/ec2-user/osap-standalone
grep -Rni "admin\|login\|auth" apps osap shared 2>/dev/null | sed -n '1,240p'
C. DAILYCUT SERVER
Host

44.200.42.187

SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
Current role

This server runs DailyCut only.

Runtime
PM2 process: dailycut-app
App port: 5003
Nginx proxies dailycut.ai and www.dailycut.ai to 127.0.0.1:5003
SSL is active and working
Verified working checks

Internal:

curl -I http://127.0.0.1:5003
curl -I -H "Host: dailycut.ai" http://127.0.0.1

External:

curl -I https://dailycut.ai
curl -I https://www.dailycut.ai
DailyCut code location on DailyCut EC2
/home/ec2-user/dailycut-standalone
Main entry
/home/ec2-user/dailycut-standalone/apps/dailycut/server.js
Current env source used at boot

Right now DailyCut was started with:

ENV_FILE=/home/ec2-user/dailyletter-backend/apps/dailycut/.env

That means DailyCut is running independently on its own EC2, but its env path still points into the copied dailyletter-backend tree on that DailyCut server.

Important DailyCut note

DailyCut is live and functioning, but its server.js still prints:

⚠️ apps/dailycut/server.js is not fully activated yet.
⚠️ Remaining unresolved dependencies must be extracted before standalone cutover.

That message is hardcoded in the file. It does not stop runtime, but it means DailyCut’s codebase still thinks extraction is unfinished. Infrastructure split is done; codebase hardening is not fully done.

PM2 start pattern
cd /home/ec2-user/dailycut-standalone
ENV_FILE=/home/ec2-user/dailyletter-backend/apps/dailycut/.env pm2 start apps/dailycut/server.js --name dailycut-app
pm2 save
PM2 boot persistence
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
pm2 save
DailyCut admin/backend note

I should not invent the admin route. Next person should verify exact admin/login entry by inspecting DailyCut standalone code:

cd /home/ec2-user/dailycut-standalone
grep -Rni "admin\|login\|2fa\|cookie" apps dailycut shared 2>/dev/null | sed -n '1,260p'
5. KNOWN BACKUPS / KNOWN SNAPSHOT HISTORY

Below are the known backups and states referenced during this work.

Earlier known backup set from handover

These were explicitly referenced earlier:

/home/ec2-user/backups/standalone-runtime-verified-2026-03-28-061347/server.js
/home/ec2-user/backups/standalone-runtime-verified-2026-03-28-061347/server.osap.js
/home/ec2-user/backups/standalone-runtime-verified-2026-03-28-061347/server.dailycut.js
/home/ec2-user/backups/standalone-runtime-verified-2026-03-28-061347/env.dailyletter.bak
/home/ec2-user/backups/standalone-runtime-verified-2026-03-28-061347/env.osap.bak
Other known OSAP-related backups seen during this process

On old backend:

/home/ec2-user/backups/osap.server.js.pre-ask-osap-homepage-2026-03-28-165246
/home/ec2-user/backups/osap.server.js.pre-hero-proof-upgrade-2026-03-28-170320
/home/ec2-user/backups/osap.server.js.pre-homepage-upgrade-2026-03-28-163342
/home/ec2-user/backups/osap.server.js.pre-premium-homepage-2026-03-28-164609
/home/ec2-user/backups/osap.server.js.pre-public-homepage-2026-03-28-164051
Known env pre-clean backup seen
/home/ec2-user/backups/env.osap.pre-clean-<timestamp>.bak

I do not have a complete guaranteed list of every backup now present on every server. So below are the commands to create fresh clean backups immediately for each system.

6. CREATE FRESH BACKUPS NOW — DO THIS FOR EACH SERVER
A. Daily Letter fresh backup

SSH in:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136

Run:

mkdir -p /home/ec2-user/backups/dailyletter-$(date +%Y%m%d-%H%M%S)

cp /home/ec2-user/dailyletter-backend/server.js /home/ec2-user/backups/dailyletter-$(date +%Y%m%d-%H%M%S)/server.js 2>/dev/null || true
cp /home/ec2-user/dailyletter-backend/.env /home/ec2-user/backups/dailyletter-$(date +%Y%m%d-%H%M%S)/env.bak 2>/dev/null || true
pm2 save
pm2 list > /home/ec2-user/backups/dailyletter-$(date +%Y%m%d-%H%M%S)/pm2-list.txt

tar -czf /home/ec2-user/backups/dailyletter-backend-$(date +%Y%m%d-%H%M%S).tar.gz \
  /home/ec2-user/dailyletter-backend \
  /etc/nginx/conf.d

Better version with one timestamp:

STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p /home/ec2-user/backups/dailyletter-$STAMP
cp /home/ec2-user/dailyletter-backend/.env /home/ec2-user/backups/dailyletter-$STAMP/env.bak 2>/dev/null || true
pm2 list > /home/ec2-user/backups/dailyletter-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d /home/ec2-user/backups/dailyletter-$STAMP/nginx-conf.d
tar -czf /home/ec2-user/backups/dailyletter-$STAMP.tar.gz /home/ec2-user/dailyletter-backend /home/ec2-user/backups/dailyletter-$STAMP
B. OSAP fresh backup

SSH in:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160

Run:

STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p /home/ec2-user/backups/osap-$STAMP
cp /home/ec2-user/osap-standalone/.env.osap /home/ec2-user/backups/osap-$STAMP/env.osap.bak 2>/dev/null || true
pm2 list > /home/ec2-user/backups/osap-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d /home/ec2-user/backups/osap-$STAMP/nginx-conf.d
tar -czf /home/ec2-user/backups/osap-$STAMP.tar.gz /home/ec2-user/osap-standalone /home/ec2-user/backups/osap-$STAMP

Optional specific code backups:

STAMP=$(date +%Y%m%d-%H%M%S)
cp /home/ec2-user/osap-standalone/apps/osap/server.js /home/ec2-user/backups/osap.server.$STAMP.js
cp /home/ec2-user/osap-standalone/.env.osap /home/ec2-user/backups/osap.env.$STAMP.bak
C. DailyCut fresh backup

SSH in:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187

Run:

STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p /home/ec2-user/backups/dailycut-$STAMP
cp /home/ec2-user/dailycut-standalone/package.json /home/ec2-user/backups/dailycut-$STAMP/package.json.bak 2>/dev/null || true
cp /home/ec2-user/dailycut-standalone/apps/dailycut/server.js /home/ec2-user/backups/dailycut-$STAMP/server.js.bak 2>/dev/null || true
pm2 list > /home/ec2-user/backups/dailycut-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d /home/ec2-user/backups/dailycut-$STAMP/nginx-conf.d
tar -czf /home/ec2-user/backups/dailycut-$STAMP.tar.gz /home/ec2-user/dailycut-standalone /home/ec2-user/backups/dailycut-$STAMP

Optional env backup if you create a dedicated env file later:

STAMP=$(date +%Y%m%d-%H%M%S)
cp /home/ec2-user/dailyletter-backend/apps/dailycut/.env /home/ec2-user/backups/dailycut.env.$STAMP.bak 2>/dev/null || true
7. HOW THE THREE BACKENDS RELATE TO EACH OTHER

Even though they are now separated, they still conceptually work together.

Daily Letter

This is the public publishing and main platform layer.

OSAP

This is the political memory / analysis / search / contradiction / archive intelligence layer.

DailyCut

This is the video clipping / rendering / subtitle / editing / automation layer.

Conceptual relationship
Daily Letter uses OSAP for intelligence, structured retrieval, political memory, analysis, archive, search, contradiction and story-related support
Daily Letter uses DailyCut for media generation, clipping, subtitles, render workflows, video processing
DailyCut may use OSAP for transcript intelligence or editorial augmentation, depending on route-level integrations
All three can be independent operationally while still connected at the API level
What independence means now

Each service can:

restart alone
deploy alone
fail alone without taking all three down
scale alone
own its own domain and SSL
run its own PM2 and nginx lifecycle
What is NOT yet fully finished

Full independence requires that all internal cross-service calls stop depending on:

old localhost assumptions
copied monolith-only files
shared env paths that only make sense inside the old backend tree
8. WHAT THE NEXT PERSON MUST DO NEXT

This is the most important section.

PRIORITY 1 — HARDEN DAILYCUT INTO A TRUE STANDALONE

Infrastructure split is complete, but DailyCut still has extraction residue.

Required next work
Create a dedicated env file for DailyCut standalone
Stop pointing DailyCut to /home/ec2-user/dailyletter-backend/apps/dailycut/.env
Move only required env variables into a dedicated standalone env
Remove hardcoded “not fully activated yet” warnings from apps/dailycut/server.js
Audit any remaining requires that still depend on root monolith artifacts unnecessarily
Useful inspection commands

On DailyCut EC2:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone

grep -n "not fully activated yet\|Remaining unresolved dependencies" apps/dailycut/server.js
grep -Rni "process.env." apps dailycut shared *.js 2>/dev/null | sed -n '1,300p'
grep -Rni "dailyletter-backend\|localhost\|127.0.0.1" . 2>/dev/null | sed -n '1,300p'
Goal

DailyCut should boot with something like:

ENV_FILE=/home/ec2-user/dailycut-standalone/.env.dailycut

instead of referencing copied old backend env paths.

PRIORITY 2 — AUDIT OSAP FOR TRUE STANDALONE INTEGRITY

OSAP is operationally independent, but next person should verify no lingering monolith assumptions remain.

On OSAP EC2:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone

grep -Rni "dailyletter-backend\|localhost\|127.0.0.1" . 2>/dev/null | sed -n '1,300p'
grep -Rni "process.env." apps osap shared 2>/dev/null | sed -n '1,300p'
Goal

Ensure OSAP uses only:

its own local code
its own env file
explicit remote service URLs where cross-service communication is needed
PRIORITY 3 — DEFINE EXPLICIT SERVICE-TO-SERVICE URLS

The next person should standardize internal communication.

Example target pattern
Daily Letter knows:
OSAP_PUBLIC_BASE_URL=https://osap.ai
DAILYCUT_PUBLIC_BASE_URL=https://dailycut.ai
OSAP knows:
Daily Letter public/editorial API base if needed
DailyCut base if needed
DailyCut knows:
Daily Letter base if needed
OSAP base if needed
Find where old assumptions live

On all servers:

grep -Rni "osap.ai\|dailycut.ai\|dailyletter.tv\|127.0.0.1\|localhost" /home/ec2-user 2>/dev/null | sed -n '1,400p'
Goal

All cross-service traffic should use:

stable domain names
or explicitly configured internal base URLs
not hidden route assumptions
PRIORITY 4 — VERIFY ADMIN ENTRY POINTS FOR EACH PRODUCT

Do not guess these. Inspect and document them.

Daily Letter

Known base:

/ironclad-2025
OSAP

Inspect:

cd /home/ec2-user/osap-standalone
grep -Rni "admin\|login\|cookie\|jwt\|2fa" apps osap shared 2>/dev/null | sed -n '1,260p'
DailyCut

Inspect:

cd /home/ec2-user/dailycut-standalone
grep -Rni "admin\|login\|cookie\|jwt\|2fa" apps dailycut shared 2>/dev/null | sed -n '1,260p'

Goal:

document exact admin URLs
document auth cookie names
document required env vars
document 2FA path if present
PRIORITY 5 — REMOVE OLD BACKEND COPY FROM DAILYCUT EC2 ONCE SAFE

Right now DailyCut EC2 still contains the copied dailyletter-backend tree because it was used during extraction.

That is acceptable short-term, but long-term the next person should:

confirm DailyCut standalone no longer depends on it
back it up
remove it
Check if still referenced
cd /home/ec2-user/dailycut-standalone
grep -Rni "/home/ec2-user/dailyletter-backend" . 2>/dev/null

Only remove after this returns clean or after all references are migrated.

9. QUICK HEALTH CHECK COMMANDS FOR EACH SERVICE
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
pm2 list
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
pm2 list
curl -I http://127.0.0.1:5002
curl -I -H "Host: osap.ai" http://127.0.0.1
curl -I https://osap.ai
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
pm2 list
curl -I http://127.0.0.1:5003
curl -I -H "Host: dailycut.ai" http://127.0.0.1
curl -I https://dailycut.ai
10. SIMPLE STATUS SUMMARY FOR THE NEXT PERSON
Completed
OSAP split to its own EC2
DailyCut split to its own EC2
Daily Letter old server cleaned of OSAP and DailyCut PM2 processes
old nginx routes for OSAP and DailyCut removed from old Daily Letter server
DNS working for osap.ai and dailycut.ai
SSL working for both standalone products
PM2 startup persistence configured on new OSAP and DailyCut servers
Not fully completed
DailyCut still needs full codebase hardening into a true standalone with its own dedicated env file and removal of extraction warning
OSAP should be audited for remaining monolith assumptions
service-to-service URLs should be made explicit and documented
admin entry points for OSAP and DailyCut should be explicitly verified and documented
old copied backend on DailyCut EC2 should eventually be removed after dependency audit
11. FINAL PICKUP POINT

If a new person picks this up, this is the order they should follow:

SSH into each machine and run fresh backups
Clean old nginx backup-note files on Daily Letter server
Verify all three services are healthy
Harden DailyCut env and remove old backend dependency
Audit OSAP for lingering monolith assumptions
Standardize cross-service URLs
Verify and document admin routes for all three
Only then start feature work

If you want, I can turn this into a cleaner operations runbook version next, with copy-paste sections labeled exactly:
Daily Letter Runbook, OSAP Runbook, and DailyCut Runbook.

yes please do

Good. This is the clean operations runbook version — structured so someone can operate, debug, deploy, and recover without guessing.

🧠 SYSTEM RUNBOOK — DAILY LETTER / OSAP / DAILYCUT
0. SYSTEM MAP (READ FIRST)
Architecture
Service	Domain	Server	Port	PM2 Name
Daily Letter	dailyletter.tv	54.164.1.136	5001	dailyletter-main
OSAP	osap.ai	98.92.67.160	5002	osap-app
DailyCut	dailycut.ai	44.200.42.187	5003	dailycut-app
1. SSH ACCESS (NEVER FORGET)
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
2. GLOBAL OPERATING RULES
ALWAYS DO THIS FIRST
# before any change
backup → inspect → patch → verify → restart → verify again
NEVER DO THIS
never edit production files blindly
never restart all PM2 apps
never assume env vars
never skip curl verification
3. DAILY LETTER RUNBOOK
Location
/home/ec2-user/dailyletter-backend
PM2 control
pm2 list
pm2 restart dailyletter-main
pm2 logs dailyletter-main --lines 200
Verify backend
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
Admin route
/ironclad-2025
Nginx reload
sudo nginx -t
sudo systemctl reload nginx
Backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailyletter-$STAMP

cp .env ~/backups/dailyletter-$STAMP/env.bak
pm2 list > ~/backups/dailyletter-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailyletter-$STAMP/nginx

tar -czf ~/backups/dailyletter-$STAMP.tar.gz \
  ~/dailyletter-backend \
  ~/backups/dailyletter-$STAMP
4. OSAP RUNBOOK
Location
/home/ec2-user/osap-standalone
Start manually
ENV_FILE=/home/ec2-user/osap-standalone/.env.osap node apps/osap/server.js
PM2 control
pm2 list
pm2 restart osap-app
pm2 logs osap-app --lines 200
Verify
curl -I http://127.0.0.1:5002
curl -I -H "Host: osap.ai" http://127.0.0.1
curl -I https://osap.ai
Nginx config
/etc/nginx/conf.d/osap.ai.conf
SSL renewal check
sudo certbot renew --dry-run
Backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/osap-$STAMP

cp .env.osap ~/backups/osap-$STAMP/env.bak
pm2 list > ~/backups/osap-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/osap-$STAMP/nginx

tar -czf ~/backups/osap-$STAMP.tar.gz \
  ~/osap-standalone \
  ~/backups/osap-$STAMP
5. DAILYCUT RUNBOOK
Location
/home/ec2-user/dailycut-standalone
Start manually
ENV_FILE=/home/ec2-user/dailyletter-backend/apps/dailycut/.env node apps/dailycut/server.js

⚠️ Temporary: still referencing old env (must be fixed later)

PM2 control
pm2 list
pm2 restart dailycut-app
pm2 logs dailycut-app --lines 200
Verify
curl -I http://127.0.0.1:5003
curl -I -H "Host: dailycut.ai" http://127.0.0.1
curl -I https://dailycut.ai
Nginx config
/etc/nginx/conf.d/dailycut.ai.conf
Backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailycut-$STAMP

cp apps/dailycut/server.js ~/backups/dailycut-$STAMP/server.bak
pm2 list > ~/backups/dailycut-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailycut-$STAMP/nginx

tar -czf ~/backups/dailycut-$STAMP.tar.gz \
  ~/dailycut-standalone \
  ~/backups/dailycut-$STAMP
6. CROSS-SERVICE COMMUNICATION
CURRENT STATE

Services are physically independent but still need logical linking.

TARGET CONFIG
Daily Letter .env
OSAP_BASE_URL=https://osap.ai
DAILYCUT_BASE_URL=https://dailycut.ai
OSAP .env
DAILYLETTER_BASE_URL=https://dailyletter.tv
DailyCut .env (to be created)
DAILYLETTER_BASE_URL=https://dailyletter.tv
OSAP_BASE_URL=https://osap.ai
7. CRITICAL NEXT TASKS
1. Fix DailyCut env independence
create .env.dailycut
remove dependency on:
/home/ec2-user/dailyletter-backend/apps/dailycut/.env
2. Remove extraction warning in DailyCut

File:

apps/dailycut/server.js

Remove:

⚠️ not fully activated yet
3. Audit all services for hidden coupling

Run on each server:

grep -Rni "dailyletter-backend\|localhost\|127.0.0.1" . 2>/dev/null

Goal:

eliminate hidden dependencies
4. Remove copied monolith from DailyCut server

Check:

grep -Rni "dailyletter-backend" dailycut-standalone

If clean:

rm -rf /home/ec2-user/dailyletter-backend
5. Document admin routes

Run:

grep -Rni "admin\|login\|jwt\|cookie" .

For:

OSAP
DailyCut
8. HEALTH CHECK (RUN ANYTIME)
Daily Letter
curl -I https://dailyletter.tv
OSAP
curl -I https://osap.ai
DailyCut
curl -I https://dailycut.ai
9. DISASTER RECOVERY
If service fails
Step 1
pm2 logs <app-name> --lines 200
Step 2
pm2 restart <app-name>
Step 3
curl -I http://127.0.0.1:<port>
Step 4
sudo systemctl reload nginx
10. FINAL STATE

You now have:

true service separation
independent deployments
independent scaling paths
clean nginx routing
isolated PM2 processes
independent SSL

👉 This is no longer a monolith
👉 This is a modular backend system

11. PICKUP POINT FOR NEXT ENGINEER

Start here:

Run backups on all 3 servers
Fix DailyCut env independence
Audit OSAP for coupling
Standardize service URLs
Document admin endpoints
Remove remaining monolith remnants

If you want next, I can give you:

👉 Phase 2: performance + scaling plan (load balancing, queues, render workers, GPU paths, etc.)

yes
PHASE 2 RUNBOOK — PERFORMANCE, SCALING, AND FULL PRODUCT INDEPENDENCE

This is the next layer after the infrastructure split.

Phase 1 gave you:

separate EC2s
separate PM2
separate nginx
separate SSL
separate domains

Phase 2 is about making the system:

faster
safer
easier to scale
easier to recover
less coupled
more production-grade
1. CURRENT ARCHITECTURE
Live now
Daily Letter
Domain: dailyletter.tv
EC2: 54.164.1.136
PM2: dailyletter-main, dailyletter-publish-worker
Port: 5001
OSAP
Domain: osap.ai
EC2: 98.92.67.160
PM2: osap-app
Port: 5002
DailyCut
Domain: dailycut.ai
EC2: 44.200.42.187
PM2: dailycut-app
Port: 5003
2. PHASE 2 OBJECTIVES
Main goals
remove hidden coupling
standardize internal service URLs
move long-running work into queues
reduce risk of one product slowing another
improve observability
separate environments cleanly
prepare for horizontal scale later
3. PRIORITY ORDER

Do Phase 2 in this order:

Phase 2A
normalize env files
standardize service-to-service URLs
remove copied-monolith references
Phase 2B
add health endpoints and health checks
add structured restart/check procedures
add service-level logs and log rotation verification
Phase 2C
isolate background jobs
move heavy render/transcription tasks behind queue boundaries
keep web servers thin
Phase 2D
introduce dedicated worker processes
add optional autoscaling paths later
prep for database and storage abstraction
4. PHASE 2A — ENV + SERVICE URL NORMALIZATION

This is the first thing the next engineer should do.

A. Daily Letter should know explicit external services
Target Daily Letter env additions

On 54.164.1.136, Daily Letter should eventually contain explicit entries like:

OSAP_BASE_URL=https://osap.ai
DAILYCUT_BASE_URL=https://dailycut.ai
Why

So Daily Letter never assumes:

localhost
shared monolith routes
same-server availability
Inspect current references

SSH:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136

Run:

cd /home/ec2-user/dailyletter-backend
grep -Rni "osap.ai\|dailycut.ai\|127.0.0.1\|localhost\|5002\|5003" . 2>/dev/null | sed -n '1,300p'
B. OSAP should know Daily Letter and optionally DailyCut explicitly

On 98.92.67.160, target OSAP env additions may include:

DAILYLETTER_BASE_URL=https://dailyletter.tv
DAILYCUT_BASE_URL=https://dailycut.ai

SSH:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160

Run:

cd /home/ec2-user/osap-standalone
grep -Rni "dailyletter.tv\|dailycut.ai\|127.0.0.1\|localhost" . 2>/dev/null | sed -n '1,300p'
C. DailyCut must get its own dedicated env file

This is one of the biggest remaining issues.

Right now DailyCut still starts with:

ENV_FILE=/home/ec2-user/dailyletter-backend/apps/dailycut/.env

That is not fully standalone.

Target

Create:

/home/ec2-user/dailycut-standalone/.env.dailycut
DailyCut target env shape

At minimum, it should include whatever is actually required by apps/dailycut/server.js and transitive imports. A likely target shape is:

NODE_ENV=production
PORT=5003
PUBLIC_BASE_URL=https://dailycut.ai
JWT_SECRET=...
OPENAI_API_KEY=...

DAILYCUT_ADMIN_USER=...
DAILYCUT_ADMIN_PW=...
DAILYCUT_ADMIN_2FA_SECRET=...
DAILYCUT_ADMIN_COOKIE_NAME=...

MEDIA_TABLE=...
MEDIA_POSTS_TABLE=...
POSTS_TABLE=...
AGENT_JOBS_TABLE=...

RENDER_WEBHOOK_URL=...
DEFAULT_BURN_SUBTITLES=...
DEFAULT_GENERATE_VTT=...
DEFAULT_SUBTITLE_LANG=...
DEFAULT_RENDER_LOGO_URL=...
DEFAULT_RENDER_LOGO_POSITION=...
DAILYCUT_DEFAULT_LOGO_URL=...

DAILYLETTER_BASE_URL=https://dailyletter.tv
OSAP_BASE_URL=https://osap.ai
Inspect exact needed envs

SSH:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187

Run:

cd /home/ec2-user/dailycut-standalone
grep -Roh 'process\.env\.[A-Z0-9_]*' apps dailycut shared *.js 2>/dev/null | sort -u
Then identify current values from copied env
grep -nE '^(NODE_ENV|PORT|PUBLIC_BASE_URL|JWT_SECRET|OPENAI_API_KEY|DAILYCUT_ADMIN_USER|DAILYCUT_ADMIN_PW|DAILYCUT_ADMIN_2FA_SECRET|DAILYCUT_ADMIN_COOKIE_NAME|MEDIA_TABLE|MEDIA_POSTS_TABLE|POSTS_TABLE|AGENT_JOBS_TABLE|RENDER_WEBHOOK_URL|DEFAULT_BURN_SUBTITLES|DEFAULT_GENERATE_VTT|DEFAULT_SUBTITLE_LANG|DEFAULT_RENDER_LOGO_URL|DEFAULT_RENDER_LOGO_POSITION|DAILYCUT_DEFAULT_LOGO_URL)=' /home/ec2-user/dailyletter-backend/apps/dailycut/.env 2>/dev/null
5. PHASE 2B — HEALTH, OBSERVABILITY, AND SAFETY
A. Add or verify health endpoints

Each service should have a lightweight endpoint like:

/health
/ready
/status
Why

You want:

quick health checks
deploy verification
future load balancer readiness
less guesswork during outages
Inspect whether they exist
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -Rni "health\|ready\|status" server.js apps shared 2>/dev/null | sed -n '1,250p'
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Rni "health\|ready\|status" apps osap shared 2>/dev/null | sed -n '1,250p'
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "health\|ready\|status" apps dailycut shared 2>/dev/null | sed -n '1,250p'
Target

Every app should respond fast to:

curl -I https://<domain>/health
B. Standardize service health checks
Daily Letter
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
OSAP
curl -I http://127.0.0.1:5002
curl -I https://osap.ai
DailyCut
curl -I http://127.0.0.1:5003
curl -I https://dailycut.ai
Add explicit status command docs

Each server should have a local operator script or at least a runbook snippet:

pm2 list
pm2 logs <app-name> --lines 100
curl -I http://127.0.0.1:<port>
sudo nginx -t
sudo systemctl status nginx
C. Verify PM2 log rotation remains healthy

Each server currently has PM2 and likely pm2-logrotate where previously installed.

Check on each server
pm2 list
pm2 conf pm2-logrotate 2>/dev/null || true
Why

Render/transcription/video services can generate huge logs. Log rotation must remain active.

6. PHASE 2C — MAKE WEB SERVERS THIN

This matters especially for DailyCut and possibly OSAP.

Problem

Today the app server itself may be doing:

request handling
video work
transcript work
render orchestration
queue dispatch logic

That is okay temporarily, but not ideal long-term.

Goal

The web server should mostly:

accept request
validate request
enqueue job
return job id / status

The heavy work should happen in workers.

A. DailyCut heavy work candidates

Likely candidates to move behind queue/worker boundary:

rendering
trimming
subtitle generation
transcription
autocut
cropping
webhook-triggered media jobs
Inspect likely heavy paths
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "ffmpeg\|render\|transcrib\|subtitle\|autocut\|crop\|spawn(" apps dailycut 2>/dev/null | sed -n '1,350p'
Strategy
keep HTTP server light
push heavy work into queue-backed workers
store job state centrally
expose status APIs
B. OSAP heavy work candidates

Likely candidates:

deep analysis jobs
long archive ingestion
bulk contradiction scans
transcript/entity extraction
article/story planning jobs
Inspect
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Rni "openai\|pipeline\|archive\|story\|transcrib\|entity\|runRoutes\|job" apps osap shared 2>/dev/null | sed -n '1,350p'
7. PHASE 2D — DEDICATED WORKERS

Once queue boundaries are clearer, split workers from web processes.

Suggested future PM2 shape
Daily Letter server
dailyletter-main
dailyletter-publish-worker
OSAP server
osap-app
future: osap-worker
DailyCut server
dailycut-app
future: dailycut-render-worker
future: dailycut-transcription-worker
Why

If rendering spikes, DailyCut web UI/API still stays responsive.

If OSAP analysis spikes, OSAP search/web routes still stay responsive.

8. LOAD, SCALE, AND INSTANCE STRATEGY

Right now single-instance-per-product is fine.

Longer-term options:

A. Vertical scaling first

Before adding more servers, consider larger instance sizes if:

CPU spikes
render times degrade
memory pressure grows
PM2 restarts due to memory
Check
pm2 monit
free -h
df -h
top
B. Horizontal scaling later

Only do this when:

apps are stateless enough
job state is externalized
files are externalized
auth/session behavior is understood
health checks are stable
Good candidates later
multiple Daily Letter web nodes behind a balancer
separate DailyCut worker fleet
separate OSAP job worker nodes
9. NGINX AND EDGE HARDENING
A. Keep nginx simple

Nginx should:

terminate SSL
proxy to local PM2 port
redirect HTTP to HTTPS
enforce client body size
maybe set reasonable timeouts

Avoid business logic in nginx.

B. Verify cert renewals periodically

On OSAP and DailyCut:

sudo certbot renew --dry-run

On Daily Letter too if managed there:

sudo certbot renew --dry-run
C. Consider CDN / caching strategy later

Especially for:

static assets
rendered media
thumbnails
public read-heavy OSAP pages
article pages on Daily Letter
10. STORAGE AND FILE STRATEGY

Long-term, all large artifacts should live outside app servers.

You already have:

Backblaze B2
Hetzner set up
Cloudflare R2 / related storage experience from prior work
Target rule

EC2 instances should not become the permanent store for:

final renders
uploaded media
large transcript artifacts
backups only stored locally
Best practice
upload raw + processed assets to object storage
store metadata in DB
use short-lived local temp files only
11. DATABASE / TABLE OWNERSHIP

This is a major next step in independence.

Right now some services likely still share tables and naming inherited from the monolith.

Target

Document ownership clearly:

Daily Letter owns
publication/article/user/subscription/publication-related tables
OSAP owns
archive/intelligence/contradiction/entity/job-related tables
DailyCut owns
media/render/subtitle/clip/job-related tables
Why

True independence is not just separate EC2s.
It also means:

clear data ownership
fewer accidental cross-service writes
safer migrations
Inspect current table env usage
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Roh 'process\.env\.[A-Z0-9_]*TABLE[A-Z0-9_]*' apps dailycut shared *.js 2>/dev/null | sort -u
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Roh 'process\.env\.[A-Z0-9_]*TABLE[A-Z0-9_]*' apps osap shared *.js 2>/dev/null | sort -u
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -Roh 'process\.env\.[A-Z0-9_]*TABLE[A-Z0-9_]*' . 2>/dev/null | sort -u
12. ADMIN SURFACE HARDENING

You asked specifically for admin backend direction for each.

Daily Letter

Known:

admin path base is /ironclad-2025

Need next:

verify login path
verify cookies
verify 2FA
verify admin-only middleware boundaries

Inspect:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -Rni "ironclad-2025\|admin\|login\|2fa\|cookie\|jwt" . 2>/dev/null | sed -n '1,300p'
OSAP

Need:

exact admin/login path
cookie name
middleware ownership
whether it should have a separate admin host or stay under same domain

Inspect:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Rni "admin\|login\|2fa\|cookie\|jwt" apps osap shared 2>/dev/null | sed -n '1,300p'
DailyCut

Need:

exact admin/login path
2FA behavior
cookie names
whether admin should be path-based or subdomain-based later

Inspect:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "admin\|login\|2fa\|cookie\|jwt" apps dailycut shared 2>/dev/null | sed -n '1,300p'
13. DISASTER RECOVERY PROCEDURES

Every next engineer should know this cold.

If a web service goes down
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
pm2 logs dailyletter-main --lines 200
pm2 restart dailyletter-main
curl -I http://127.0.0.1:5001
sudo nginx -t
sudo systemctl reload nginx
curl -I https://dailyletter.tv
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
pm2 logs osap-app --lines 200
pm2 restart osap-app
curl -I http://127.0.0.1:5002
sudo nginx -t
sudo systemctl reload nginx
curl -I https://osap.ai
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
pm2 logs dailycut-app --lines 200
pm2 restart dailycut-app
curl -I http://127.0.0.1:5003
sudo nginx -t
sudo systemctl reload nginx
curl -I https://dailycut.ai
14. BACKUP POLICY GOING FORWARD

Before any meaningful change on any server:

Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailyletter-$STAMP
cp /home/ec2-user/dailyletter-backend/.env ~/backups/dailyletter-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/dailyletter-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailyletter-$STAMP/nginx-conf.d
tar -czf ~/backups/dailyletter-$STAMP.tar.gz /home/ec2-user/dailyletter-backend ~/backups/dailyletter-$STAMP
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/osap-$STAMP
cp /home/ec2-user/osap-standalone/.env.osap ~/backups/osap-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/osap-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/osap-$STAMP/nginx-conf.d
tar -czf ~/backups/osap-$STAMP.tar.gz /home/ec2-user/osap-standalone ~/backups/osap-$STAMP
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailycut-$STAMP
cp /home/ec2-user/dailycut-standalone/apps/dailycut/server.js ~/backups/dailycut-$STAMP/server.bak 2>/dev/null || true
pm2 list > ~/backups/dailycut-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailycut-$STAMP/nginx-conf.d
tar -czf ~/backups/dailycut-$STAMP.tar.gz /home/ec2-user/dailycut-standalone ~/backups/dailycut-$STAMP
15. WHAT “FULL INDEPENDENCE” MEANS FROM HERE

You are not done when each product has its own EC2.
You are done when each product also has:

its own env file
explicit external service URLs
no hidden monolith file paths
clear DB/table ownership
clear job ownership
clear admin boundary
recoverability without relying on another app’s server tree
Today’s reality
Daily Letter

Mostly independent

OSAP

Operationally independent, still should be audited for hidden coupling

DailyCut

Operationally independent, but not fully codebase-independent yet

16. PICKUP PLAN FOR THE NEXT ENGINEER

Use this order:

Step 1

Back up all 3 systems

Step 2

Clean remaining inactive nginx note/backup files on Daily Letter server

Step 3

Create dedicated .env.dailycut

Step 4

Retarget DailyCut PM2 to:

ENV_FILE=/home/ec2-user/dailycut-standalone/.env.dailycut
Step 5

Search for and remove remaining references to:

/home/ec2-user/dailyletter-backend
localhost
127.0.0.1 where not intended
old monolith assumptions
Step 6

Audit OSAP for same class of hidden coupling

Step 7

Document admin entrypoints precisely for all 3 systems

Step 8

Move heavy jobs further behind queue/worker boundaries

17. WHAT THIS WHOLE SPLIT NOW GIVES YOU

This matters strategically.

You now have:

Daily Letter as the publishing and audience layer
OSAP as the intelligence and political memory layer
DailyCut as the media generation and video processing layer

They can work together, but each can now:

evolve independently
be restarted independently
be deployed independently
be scaled independently
fail without fully taking down the others

That is a real systems milestone.

If you want, next I can write Phase 3: product-to-product API contract plan, which would define exactly how Daily Letter, OSAP, and DailyCut should talk to each other cleanly.

yes
PHASE 3 RUNBOOK — PRODUCT-TO-PRODUCT API CONTRACT PLAN

This is the layer that turns three separate servers into one coherent platform.

Phase 1 split the infrastructure.
Phase 2 hardens performance, envs, queues, and observability.
Phase 3 defines the contracts so Daily Letter, OSAP, and DailyCut communicate cleanly, predictably, and safely.

1. WHY THIS MATTERS

Right now the products are separated operationally, but the next risk is messy integration:

hidden route assumptions
one service calling another with ad hoc payloads
weak auth between products
duplicated logic
no standard timeouts/retries
hard-to-debug failures

Phase 3 fixes that by defining:

who calls whom
for what purpose
through which endpoints
with what payloads
with what auth
with what timeout/retry behavior
with what failure handling
2. THE ROLES OF EACH PRODUCT
Daily Letter

This is the public publishing layer.

It should own:

articles
publication workflows
editorial publishing
audience-facing pages
subscriptions / public content access
editorial UI

It should call the other systems when it needs:

intelligence or archive analysis from OSAP
media generation or clip/subtitle/render workflows from DailyCut
OSAP

This is the intelligence layer.

It should own:

archive
memory
contradiction logic
political entity analysis
research retrieval
story planning
transcript-based intelligence
political knowledge workflows

It should not become the public publishing layer.
It should expose clean APIs for intelligence services.

DailyCut

This is the media transformation layer.

It should own:

video trim
crop
subtitle generation
render workflows
media processing jobs
clip suggestions / autocut workflows
output artifact generation

It should not become the core archive/intelligence system or the main publication backend.

3. TARGET CALL GRAPH

This is the recommended product-to-product interaction map.

Primary relationships
Daily Letter → OSAP

Use for:

research lookup
contradiction check
archive retrieval
transcript/story planning
intelligence-assisted editorial workflows
Daily Letter → DailyCut

Use for:

video clip creation
subtitle generation
render requests
media transforms for article/video publishing
DailyCut → OSAP

Optional, only when useful for:

transcript enrichment
editorial understanding
segment intelligence
story-aware clip logic
OSAP → DailyCut

Optional, only when useful for:

create visual assets from OSAP-driven analysis
clip/transcript outputs connected to intelligence workflows
Avoid this unless necessary
Daily Letter should not proxy every request through itself
OSAP should not depend on Daily Letter for its own core functions
DailyCut should not depend on Daily Letter filesystem state
One service should not reach into another service’s database tables directly unless explicitly designed
4. REQUIRED ENV CONTRACTS

Every service should know the others by explicit base URL.

Daily Letter env

On 54.164.1.136:

OSAP_BASE_URL=https://osap.ai
DAILYCUT_BASE_URL=https://dailycut.ai
INTERNAL_SERVICE_TOKEN=...
OSAP env

On 98.92.67.160:

DAILYLETTER_BASE_URL=https://dailyletter.tv
DAILYCUT_BASE_URL=https://dailycut.ai
INTERNAL_SERVICE_TOKEN=...
DailyCut env

On 44.200.42.187:

DAILYLETTER_BASE_URL=https://dailyletter.tv
OSAP_BASE_URL=https://osap.ai
INTERNAL_SERVICE_TOKEN=...
Why INTERNAL_SERVICE_TOKEN

This gives a clean first step for machine-to-machine authentication without inventing a full auth platform.

Recommended pattern:

all server-to-server privileged requests include:
Authorization: Bearer <INTERNAL_SERVICE_TOKEN>

Or:

X-Internal-Token: <INTERNAL_SERVICE_TOKEN>

Same value on all three for now, or distinct service tokens later.

5. API DESIGN RULES

This is the contract discipline the next engineer should follow.

Rule 1

Public routes and internal routes should not be mixed casually.

Use naming like:

/api/public/...
/api/internal/...
/api/admin/...

If changing existing routes is too disruptive, start by documenting which existing routes are internal-only.

Rule 2

Asynchronous jobs should return job IDs, not block forever.

Bad pattern:

call DailyCut
wait 3 minutes for render in the request

Good pattern:

POST render request
DailyCut returns { jobId, status: "queued" }
caller polls status endpoint or receives webhook later
Rule 3

One service should not reach into another service’s filesystem.

Only communicate through:

HTTP API
queue
shared object storage if explicitly intended
webhook callback
Rule 4

Payloads should be versionable.

Use a convention like:

/api/internal/v1/...

Even if you do not version everything immediately, new internal contracts should aim for versionable structure.

Rule 5

Timeouts and retries must be explicit.

When one service calls another:

set request timeout
define retry rules
define fallback behavior

Do not let hangs become invisible.

6. RECOMMENDED INTERNAL API SURFACES

Below is a practical first-pass contract plan.

6A. DAILY LETTER → OSAP CONTRACTS

These are the most valuable integrations.

1. Research / archive lookup
Endpoint
POST /api/internal/v1/research/search
Purpose

Daily Letter asks OSAP for archive-backed research results.

Request
{
  "query": "Pete Hegseth promotion controversy",
  "context": "breaking-news drafting",
  "maxResults": 10
}
Response
{
  "results": [
    {
      "title": "Result title",
      "summary": "Short summary",
      "sourceType": "archive",
      "url": "https://...",
      "confidence": 0.91
    }
  ]
}
2. Contradiction analysis
Endpoint
POST /api/internal/v1/analysis/contradictions
Purpose

Daily Letter sends a statement/topic/person and asks OSAP for contradiction records.

Request
{
  "subject": "Pete Hegseth",
  "claim": "Supported merit-only military promotions",
  "timeRange": {
    "from": "2024-01-01",
    "to": "2026-03-28"
  }
}
Response
{
  "subject": "Pete Hegseth",
  "matches": [
    {
      "date": "2025-10-10",
      "statement": "...",
      "source": "...",
      "contradictionScore": 0.88
    }
  ]
}
3. Story planning / editorial assist
Endpoint
POST /api/internal/v1/story/plan
Purpose

Daily Letter asks OSAP to generate structured editorial angles from a transcript/article/topic.

Request
{
  "headline": "Draft headline",
  "sourceUrl": "https://...",
  "transcriptText": "...",
  "style": "investigative"
}
Response
{
  "anchorEvent": {...},
  "angles": [
    {
      "title": "Angle",
      "whyItMatters": "..."
    }
  ]
}
6B. DAILY LETTER → DAILYCUT CONTRACTS

These are critical for publication/media workflows.

1. Render request
Endpoint
POST /api/internal/v1/render/jobs
Purpose

Daily Letter asks DailyCut to render a media artifact.

Request
{
  "postId": "abc123",
  "inputVideoUrl": "https://...",
  "template": "youtube-landscape",
  "burnSubtitles": true,
  "logoUrl": "https://...",
  "callbackUrl": "https://dailyletter.tv/api/internal/v1/dailycut/callback"
}
Response
{
  "jobId": "render_123",
  "status": "queued"
}
2. Subtitle generation
Endpoint
POST /api/internal/v1/subtitles/jobs
Request
{
  "videoUrl": "https://...",
  "language": "en",
  "postId": "abc123",
  "callbackUrl": "https://dailyletter.tv/api/internal/v1/dailycut/callback"
}
Response
{
  "jobId": "sub_123",
  "status": "queued"
}
3. Clip generation / autocut
Endpoint
POST /api/internal/v1/autocut/jobs
Request
{
  "postId": "abc123",
  "videoUrl": "https://...",
  "strategy": "editorial-highlights",
  "callbackUrl": "https://dailyletter.tv/api/internal/v1/dailycut/callback"
}
Response
{
  "jobId": "autocut_123",
  "status": "queued"
}
4. Job status
Endpoint
GET /api/internal/v1/jobs/:jobId
Response
{
  "jobId": "render_123",
  "status": "processing",
  "progress": 42,
  "artifactUrl": null,
  "error": null
}
6C. DAILYCUT → DAILY LETTER CALLBACK CONTRACTS

DailyCut should notify Daily Letter when jobs complete.

Callback endpoint on Daily Letter
POST /api/internal/v1/dailycut/callback
Payload
{
  "jobId": "render_123",
  "jobType": "render",
  "status": "completed",
  "postId": "abc123",
  "artifactUrl": "https://...",
  "thumbnailUrl": "https://...",
  "metadata": {
    "durationSec": 61
  }
}
Error version
{
  "jobId": "render_123",
  "jobType": "render",
  "status": "failed",
  "postId": "abc123",
  "error": "ffmpeg exited with code 1"
}
6D. OSAP ↔ DAILYCUT OPTIONAL CONTRACTS

Use these only when a real workflow needs them.

1. Transcript enrichment

DailyCut asks OSAP for segment/story understanding from transcript text.

Endpoint
POST /api/internal/v1/transcript/enrich
Request
{
  "transcriptText": "...",
  "headline": "Raw video headline",
  "purpose": "autocut-ranking"
}
Response
{
  "entities": [...],
  "themes": [...],
  "recommendedMoments": [...]
}
2. Visual asset job from OSAP

OSAP asks DailyCut to generate visual video/short asset from OSAP-driven outputs.

Endpoint
POST /api/internal/v1/visual/jobs
7. AUTHENTICATION CONTRACT

This is one of the most important pieces.

Minimum viable approach

All internal endpoints require one shared internal token.

Request header
Authorization: Bearer <INTERNAL_SERVICE_TOKEN>
Validation logic

Each service should reject internal endpoints if token is missing or wrong:

401 Unauthorized
403 Forbidden if needed
Better future approach

Later you can move to:

per-service tokens
signed requests
service identity / IAM style auth
mTLS if you become very serious

But right now, a strong shared token is enough.

8. ERROR CONTRACTS

All services should use predictable error shapes.

Standard error response
{
  "ok": false,
  "error": {
    "code": "JOB_NOT_FOUND",
    "message": "No job exists for the supplied ID"
  }
}
Standard success response
{
  "ok": true,
  "data": {...}
}
Why

Without standard responses:

integrations get brittle
logging becomes inconsistent
fallback behavior gets messy
9. TIMEOUT AND RETRY POLICY

This is critical.

Daily Letter calling OSAP
timeout: 10–20 seconds for analysis/search
retry: 1 retry for transient 5xx only
fallback: show partial editorial workflow result or “OSAP unavailable”
Daily Letter calling DailyCut
timeout: short, 5–10 seconds for job creation
retry: 1 retry for job creation if idempotent
fallback: mark render request failed and allow manual retry
DailyCut callback to Daily Letter
timeout: 5–10 seconds
retry: exponential backoff on 5xx
persist callback attempts in job log
10. IDEMPOTENCY RULES

For job-creation APIs, duplicate requests should not create chaos.

Recommended pattern

Caller can send:

Idempotency-Key: <uuid>
Result

If the same request is retried, service returns the same job instead of creating duplicates.

This is especially useful for:

render jobs
subtitle jobs
autocut jobs
story plan jobs if expensive
11. WEBHOOK SECURITY

For callbacks, do not just trust the internet.

Minimum viable

Use the same internal auth token in callback header.

Better

Include:

timestamp
signature
replay protection window

Example headers:

X-Internal-Timestamp: 1711660000
X-Internal-Signature: sha256=...
12. DATA OWNERSHIP CONTRACTS

This is how you prevent accidental architectural drift.

Daily Letter owns
post/article metadata
editorial publishing state
audience/public page state
OSAP owns
archive records
contradiction records
research memory
intelligence jobs
entity analysis outputs
DailyCut owns
render jobs
subtitle jobs
clip jobs
media processing outputs
processing metadata
Rule

One service should not write directly into another service’s tables unless explicitly defined and documented.

Better:

service A sends request to service B
service B writes its own data
service B returns result or callback
13. RECOMMENDED INTERNAL ROUTE ORGANIZATION

Each service should expose a clear route layout.

Daily Letter
/api/public/...
/api/admin/...
/api/internal/...
OSAP
/api/public/...
/api/admin/...
/api/internal/...
DailyCut
/api/public/...
/api/admin/...
/api/internal/...

Even if you do not rename all legacy routes immediately, use this as the direction for new work.

14. OBSERVABILITY CONTRACT

Every cross-service request should be traceable.

Include a request ID

Headers:

X-Request-Id: <uuid>

If Daily Letter initiates something and calls OSAP or DailyCut:

pass the same X-Request-Id
log it on all sides
Why

This makes debugging possible across products.

15. FIRST IMPLEMENTATION PLAN FOR THE NEXT ENGINEER

Do this in order.

Step 1

Define env vars on all services:

OSAP_BASE_URL
DAILYCUT_BASE_URL
DAILYLETTER_BASE_URL
INTERNAL_SERVICE_TOKEN
Step 2

Add internal auth middleware to all new internal routes

Step 3

Document existing routes that already act as internal APIs

Step 4

Standardize Daily Letter → DailyCut job creation contract

Step 5

Standardize Daily Letter → OSAP story/research contract

Step 6

Add DailyCut → Daily Letter callback contract

Step 7

Add request ID propagation and consistent error shapes

16. INSPECTION COMMANDS FOR THE NEXT ENGINEER

These are the exact commands to start Phase 3 work.

On Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -Rni "osap.ai\|dailycut.ai\|fetch(\|axios\|http://\|https://" . 2>/dev/null | sed -n '1,350p'
On OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Rni "dailyletter.tv\|dailycut.ai\|fetch(\|axios\|http://\|https://" . 2>/dev/null | sed -n '1,350p'
On DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "dailyletter.tv\|osap.ai\|fetch(\|axios\|http://\|https://" . 2>/dev/null | sed -n '1,350p'
Inspect env usage for base URLs and auth
grep -Roh 'process\.env\.[A-Z0-9_]*' . 2>/dev/null | sort -u
17. WHAT SUCCESS LOOKS LIKE

Phase 3 is successful when:

Daily Letter talks to OSAP and DailyCut through explicit base URLs
DailyCut and OSAP do not rely on hidden monolith assumptions
internal routes require service auth
long jobs are asynchronous
callbacks are authenticated
request IDs let you trace flows across services
each product owns its own domain, runtime, and data contracts
18. STRATEGIC END STATE

When Phases 1–3 are done properly, you will have:

Daily Letter

The publishing system

OSAP

The intelligence engine

DailyCut

The media engine

Together they become:

one platform
three independent products
one architecture with explicit contracts instead of hidden coupling

That is how you build something durable.

If you want, next I can write Phase 4: security and admin hardening plan.

yes
PHASE 4 RUNBOOK — SECURITY AND ADMIN HARDENING PLAN

This phase is about making the three-product system safer, tighter, and harder to break.

Phase 1 split the infrastructure.
Phase 2 hardened runtime, envs, and performance direction.
Phase 3 defined API contracts.
Phase 4 hardens admin access, internal trust, secrets, and operational security.

1. PHASE 4 OBJECTIVES

Main goals:

lock down admin access
reduce blast radius if one app is compromised
standardize authentication and cookies
protect internal APIs
clean up secrets handling
tighten nginx and EC2 exposure
make recovery and auditing easier
2. CURRENT SECURITY REALITY

You now have:

3 separate EC2 instances
3 separate nginx layers
3 separate PM2 runtimes
3 separate public domains

That is already a security improvement because compromise in one service is less likely to directly take down the other two.

But there are still likely risks:

copied envs and legacy secrets
unclear admin endpoints
shared internal assumptions
possible overexposed routes
weak internal service auth
lingering files and backups with secrets in them
too much access from the public internet
3. ADMIN SURFACE HARDENING

This is one of the most important parts.

A. Daily Letter admin

Known route base:

/ironclad-2025

That is already better than /admin, but it should not be your only protection.

Required next checks

On Daily Letter server:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -Rni "ironclad-2025\|admin\|login\|2fa\|cookie\|jwt\|basic-auth" . 2>/dev/null | sed -n '1,350p'
Goal

Confirm:

exact login route
exact auth middleware
whether 2FA is enforced
cookie name
session/JWT expiry behavior
logout route
password reset exposure if any
B. OSAP admin

Need exact discovery, not guesswork.

Run:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -Rni "admin\|login\|2fa\|cookie\|jwt\|basic-auth" apps osap shared 2>/dev/null | sed -n '1,350p'
Goal

Document:

admin URL
admin auth method
cookie name
whether it shares JWT secret style with Daily Letter
whether there is separate admin-only middleware
C. DailyCut admin

Run:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "admin\|login\|2fa\|cookie\|jwt\|basic-auth" apps dailycut shared 2>/dev/null | sed -n '1,350p'
Goal

Document:

exact admin/login path
2FA flow
cookie names
JWT or session method
whether admin APIs are segregated cleanly
4. ADMIN ACCESS POLICY

This is the recommended target model.

Rule 1

Each product should have its own admin boundary.

Not:

one admin session accidentally valid everywhere

Prefer:

Daily Letter admin auth scoped to Daily Letter
OSAP admin auth scoped to OSAP
DailyCut admin auth scoped to DailyCut
Rule 2

Each product should have its own cookie name.

Examples:

Daily Letter: dl_admin
OSAP: osap_admin
DailyCut: dailycut_admin

You already saw:

OSAP_ADMIN_COOKIE_NAME=osap_admin
DailyCut likely has its own admin vars too

That is the right direction.

Rule 3

2FA should be required for admin if already supported.

Wherever 2FA is already in place, keep it enforced and verify it still works after the split.

Rule 4

Admin routes should not be discoverable by UI accidents.

That means:

avoid exposing admin links publicly
avoid frontends leaking admin endpoints in obvious ways if not intended
avoid default /admin when already moved
5. SECRET MANAGEMENT HARDENING

This is a major priority.

Current likely issue

Secrets may currently exist in:

.env
copied .env files
backup tarballs
historical backup files
shell history
older standalone copies

That is common, but must be tightened.

A. Inventory secrets on each server
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
grep -nE '^(JWT_SECRET|OPENAI_API_KEY|ADMIN_SECRET_KEY|OSAP_WEB_SEARCH_KEY|.*SECRET.*|.*KEY.*|.*TOKEN.*)=' .env 2>/dev/null
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
grep -nE '^(JWT_SECRET|OPENAI_API_KEY|OSAP_WEB_SEARCH_KEY|.*SECRET.*|.*KEY.*|.*TOKEN.*)=' .env.osap 2>/dev/null
DailyCut

After you create .env.dailycut:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -nE '^(JWT_SECRET|OPENAI_API_KEY|.*SECRET.*|.*KEY.*|.*TOKEN.*)=' .env.dailycut 2>/dev/null
B. Target rule

Each service should contain only the secrets it needs.

Not:

DailyCut carrying unrelated Daily Letter secrets
OSAP carrying unrelated DailyCut secrets
all apps sharing giant env files with excess privileges
C. Rotation policy

After env cleanup stabilizes, rotate high-value secrets in this order:

internal service token
admin secrets / 2FA recovery-sensitive material if needed
external API keys if overexposed historically
JWT secrets if auth model changes require it

Only rotate after you verify where each secret is actually used.

6. INTERNAL API SECURITY

This should be implemented as part of Phase 3/4 together.

Target

Every internal route should require a machine token.

Header
Authorization: Bearer <INTERNAL_SERVICE_TOKEN>
Why

Without this, anyone who discovers an internal route may be able to trigger expensive or privileged work.

Required next engineering step

On all three codebases, identify current internal routes and add middleware.

Search examples
Daily Letter
grep -Rni "/api/internal\|dailycut\|osap" /home/ec2-user/dailyletter-backend 2>/dev/null | sed -n '1,300p'
OSAP
grep -Rni "/api/internal\|router\." /home/ec2-user/osap-standalone 2>/dev/null | sed -n '1,300p'
DailyCut
grep -Rni "/api/internal\|router\." /home/ec2-user/dailycut-standalone 2>/dev/null | sed -n '1,300p'
7. NGINX HARDENING

nginx should stay simple but secure.

A. Maintain HTTPS redirect

Already working on OSAP and DailyCut.

Keep:

HTTP → HTTPS redirect
valid SSL certs
no accidental plain HTTP app exposure to the public
B. Consider basic request limits where appropriate

Especially on:

admin login routes
heavy POST endpoints
render initiation routes
search abuse surfaces

If not already handled at app layer, nginx can help with simple rate limiting later.

C. Hide unnecessary server details where possible

Not urgent, but later you may want to reduce obvious fingerprinting.

D. Check live nginx configs
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
sudo nginx -t
sudo ls -lah /etc/nginx/conf.d
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
sudo nginx -t
sudo ls -lah /etc/nginx/conf.d
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
sudo nginx -t
sudo ls -lah /etc/nginx/conf.d
8. FIREWALL / NETWORK EXPOSURE HARDENING

Very important.

Goal

Public internet should reach:

80
443
22 only if necessary and preferably restricted

The app ports:

5001
5002
5003

should not be publicly open in security groups unless there is a deliberate reason.

Check security group exposure

This must be reviewed in AWS console or via CLI if configured.

Desired state
22 restricted to your IP if practical
80 open
443 open
5001/5002/5003 not publicly exposed
Why

Those ports are meant for nginx local proxying, not direct internet traffic.

9. PM2 AND PROCESS HARDENING
A. Keep only required processes alive

You already cleaned old Daily Letter server of OSAP and DailyCut.

This is good security and operational hygiene.

Verify periodically
Daily Letter
pm2 list

Should show only:

dailyletter-main
dailyletter-publish-worker
OSAP

Should show only:

osap-app
DailyCut

Should show only:

dailycut-app

Unless you intentionally add workers later.

B. Watch for accidental process duplication

Never leave:

manual foreground process
PM2 process
old process on same port

At the same time.

Check
sudo lsof -i :5001
sudo lsof -i :5002
sudo lsof -i :5003
10. BACKUP SECURITY

Backups are necessary, but they are also sensitive.

Risk

Your backups may contain:

env files
API keys
JWT secrets
admin auth values
internal routing details
Policy
store backups in predictable location
name them clearly
do not casually copy them around
eventually move critical backups to encrypted storage or tightly controlled bucket
Find sensitive backups
Daily Letter
find /home/ec2-user/backups -maxdepth 3 -type f | sed -n '1,300p'

Do this similarly on OSAP and DailyCut servers.

11. LOGGING SECURITY

Logs can leak:

tokens
cookies
internal URLs
user PII
raw payloads
Goal

Avoid logging secrets or full auth headers.

Search suspicious logging
Daily Letter
grep -Rni "console\.log\|logger\." /home/ec2-user/dailyletter-backend 2>/dev/null | sed -n '1,350p'
OSAP
grep -Rni "console\.log\|logger\." /home/ec2-user/osap-standalone 2>/dev/null | sed -n '1,350p'
DailyCut
grep -Rni "console\.log\|logger\." /home/ec2-user/dailycut-standalone 2>/dev/null | sed -n '1,350p'
Audit for secrets being logged
grep -Rni "authorization\|cookie\|token\|secret\|jwt" /home/ec2-user/* 2>/dev/null | sed -n '1,400p'

This needs human review, not blind deletion.

12. ADMIN LOGIN HARDENING
Target features

For each admin surface:

strong password
2FA if supported
rate limiting on login
secure cookies
explicit logout
session expiry / token expiry
CSRF awareness if cookie-based admin actions exist
Check for rate limiting
DailyCut already showed express-rate-limit in use

That is promising.

Search it precisely:

ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "express-rate-limit\|rateLimit(" apps dailycut shared 2>/dev/null | sed -n '1,200p'

Do similarly for Daily Letter and OSAP.

13. COOKIE HARDENING

For admin cookies, confirm these are used if appropriate:

HttpOnly
Secure
SameSite=Lax or stricter where feasible
product-specific cookie names
limited path/domain scope where practical
Search
grep -Rni "cookie" /home/ec2-user/dailyletter-backend /home/ec2-user/osap-standalone /home/ec2-user/dailycut-standalone 2>/dev/null | sed -n '1,400p'
14. REMOVE LEGACY AND UNUSED FILES

This is both hygiene and security.

Daily Letter old server

You already removed active old routes for OSAP and DailyCut.

Still recommended:

remove old inactive nginx backup-note clutter
verify no obsolete public-facing configs remain
DailyCut server

Eventually remove copied dailyletter-backend tree after DailyCut is fully env-independent and no references remain.

Search for lingering old references
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
grep -Rni "/home/ec2-user/dailyletter-backend" . 2>/dev/null

Only after this is clean should the copied backend be removed.

15. ACCESS CONTROL POLICY FOR HUMANS

This is not just code. It is operator discipline.

Recommended rules
only one primary SSH key for authorized operator unless intentionally expanded
do not share the private key casually
if freelancer/dev needs access, create separate SSH key and remove later
prefer per-person access over shared credentials
keep a list of who has access to which box
16. MINIMUM INCIDENT RESPONSE PLAYBOOK

If something suspicious happens:

Step 1

Identify affected service:

Daily Letter
OSAP
DailyCut
Step 2

Check recent PM2 logs

pm2 logs <app-name> --lines 200
Step 3

Check nginx logs/system state

sudo systemctl status nginx
sudo nginx -t
Step 4

Check running listeners

sudo lsof -i :5001
sudo lsof -i :5002
sudo lsof -i :5003
Step 5

If compromise suspected

do not randomly delete evidence
back up configs and logs first
rotate relevant tokens/secrets
kill only clearly malicious or duplicate processes
restrict access if needed
17. PHASE 4 CHECKLIST FOR THE NEXT ENGINEER

This is the order they should follow.

Step 1

Inventory exact admin routes and auth methods for all 3 products

Step 2

Confirm product-specific cookies and 2FA behavior

Step 3

Create internal service auth middleware for all internal APIs

Step 4

Audit all env files and trim unnecessary secrets

Step 5

Plan secret rotation once env cleanup is complete

Step 6

Confirm security groups do not expose 5001/5002/5003 publicly

Step 7

Audit logs for secret leakage

Step 8

Remove stale files and monolith remnants once safe

18. WHAT SUCCESS LOOKS LIKE

Phase 4 is successful when:

admin surfaces are clearly documented and protected
internal APIs require service auth
each app has only the secrets it actually needs
app ports are not publicly exposed
cookies are scoped and hardened
logs do not leak secrets
old leftover files are removed
one service compromise has limited blast radius
19. STRATEGIC OUTCOME

When Phases 1–4 are done:

Daily Letter is the secure publishing layer
OSAP is the secure intelligence layer
DailyCut is the secure media layer

Together they remain connected, but each has:

separate infrastructure
separate runtime
separate admin boundary
explicit contracts
controlled trust

That is the foundation for something serious.

If you want, next I can write Phase 5: deployment and release management plan, so the next engineer knows exactly how to deploy changes to each service without breaking the others.

yes
PHASE 5 RUNBOOK — DEPLOYMENT AND RELEASE MANAGEMENT PLAN

This phase is about how to change the system safely without breaking the other products.

By now the platform is split into:

Daily Letter — publishing layer
OSAP — intelligence layer
DailyCut — media layer

Phase 5 defines:

how to deploy each one
how to verify each one
how to roll back each one
how to avoid cross-service breakage
1. DEPLOYMENT PHILOSOPHY
Rule 1

Deploy one service at a time.

Do not update:

Daily Letter
OSAP
DailyCut

all in one shot unless absolutely necessary.

Rule 2

Every deployment starts with:

inspect
backup
patch
syntax check
local verification
process restart
external verification
Rule 3

Never restart unrelated services.

Examples:

DailyCut deployment should not restart OSAP
OSAP deployment should not restart Daily Letter
Daily Letter deployment should not restart DailyCut
Rule 4

Always verify locally before externally.

Example order:

file diff / grep / syntax
curl -I http://127.0.0.1:<port>
then curl -I https://domain
2. DEPLOYMENT TARGETS
Service	Server	App Path	PM2 App	Port	Public Domain
Daily Letter	54.164.1.136	/home/ec2-user/dailyletter-backend	dailyletter-main	5001	https://dailyletter.tv
OSAP	98.92.67.160	/home/ec2-user/osap-standalone	osap-app	5002	https://osap.ai
DailyCut	44.200.42.187	/home/ec2-user/dailycut-standalone	dailycut-app	5003	https://dailycut.ai
3. SSH COMMANDS
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
4. STANDARD DEPLOYMENT TEMPLATE

This is the exact pattern every engineer should follow.

Step A — SSH into target server

Choose only the server you are working on.

Step B — backup

Use timestamped backup before touching files.

Step C — inspect current code

Use:

grep
sed -n
tail
cat
find

before editing.

Step D — patch surgically

Prefer:

python scripts
sed
exact file replacement
controlled heredoc edits

Do not blindly edit large files without checking exact insertion point.

Step E — verify syntax / startup assumptions

Depending on change:

node -c file.js for syntax if appropriate
nginx -t for nginx
pm2 list
grep after patch
local curl
Step F — restart only target process

Use:

pm2 restart <app-name>
Step G — verify locally

Use:

curl -I http://127.0.0.1:<port>
Step H — verify externally

Use:

curl -I https://<domain>
Step I — save PM2 if process config changed
pm2 save
5. DAILY LETTER DEPLOYMENT RUNBOOK
SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
App path
cd /home/ec2-user/dailyletter-backend
Pre-deploy backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailyletter-$STAMP
cp .env ~/backups/dailyletter-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/dailyletter-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailyletter-$STAMP/nginx-conf.d
tar -czf ~/backups/dailyletter-$STAMP.tar.gz /home/ec2-user/dailyletter-backend ~/backups/dailyletter-$STAMP
Restart only Daily Letter
pm2 restart dailyletter-main
Verify
pm2 list
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
If nginx changed
sudo nginx -t
sudo systemctl reload nginx
6. OSAP DEPLOYMENT RUNBOOK
SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
App path
cd /home/ec2-user/osap-standalone
Pre-deploy backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/osap-$STAMP
cp .env.osap ~/backups/osap-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/osap-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/osap-$STAMP/nginx-conf.d
tar -czf ~/backups/osap-$STAMP.tar.gz /home/ec2-user/osap-standalone ~/backups/osap-$STAMP
Restart only OSAP
pm2 restart osap-app
Verify
pm2 list
curl -I http://127.0.0.1:5002
curl -I -H "Host: osap.ai" http://127.0.0.1
curl -I https://osap.ai
If nginx changed
sudo nginx -t
sudo systemctl reload nginx
7. DAILYCUT DEPLOYMENT RUNBOOK
SSH
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
App path
cd /home/ec2-user/dailycut-standalone
Pre-deploy backup
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailycut-$STAMP
cp apps/dailycut/server.js ~/backups/dailycut-$STAMP/server.bak 2>/dev/null || true
pm2 list > ~/backups/dailycut-$STAMP/pm2-list.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailycut-$STAMP/nginx-conf.d
tar -czf ~/backups/dailycut-$STAMP.tar.gz /home/ec2-user/dailycut-standalone ~/backups/dailycut-$STAMP
Restart only DailyCut
pm2 restart dailycut-app
Verify
pm2 list
curl -I http://127.0.0.1:5003
curl -I -H "Host: dailycut.ai" http://127.0.0.1
curl -I https://dailycut.ai
If nginx changed
sudo nginx -t
sudo systemctl reload nginx
8. DEPLOYMENT TYPES

Every deployment usually falls into one of these categories.

Type A — Code-only deploy

Examples:

route logic change
helper change
middleware change
HTML/frontend-in-backend shell changes
Steps
backup
patch file
syntax check / grep
restart target PM2 app
local curl
external curl
Type B — Env change

Examples:

new API key
new base URL
new auth token
changed table name
Steps
backup env
inspect current env lines
change env surgically
restart only target PM2 app
verify local + external
verify feature using that env value
Type C — Nginx change

Examples:

new domain
new proxy target
redirect change
certbot change
Steps
backup nginx
patch nginx file
sudo nginx -t
sudo systemctl reload nginx
test localhost Host header
test public domain
Type D — Dependency change

Examples:

npm install
package version changes
new module added
Steps
backup app dir and package files
inspect package.json
install dependency
restart target app
inspect logs carefully
curl local
curl external
9. ROLLBACK PLAN

This must be simple and immediate.

Rule

If a deployment breaks and the fix is not instantly obvious:

stop changing more things
restore known good state
bring service back
investigate after stability returns
Daily Letter rollback

If file-level rollback:

cp /path/to/backup/file /home/ec2-user/dailyletter-backend/<target-file>
pm2 restart dailyletter-main
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv

If full rollback from tarball:

tar -xzf ~/backups/dailyletter-<STAMP>.tar.gz -C /

Only do full rollback carefully and knowingly.

OSAP rollback
cp /path/to/backup/file /home/ec2-user/osap-standalone/<target-file>
pm2 restart osap-app
curl -I http://127.0.0.1:5002
curl -I https://osap.ai
DailyCut rollback
cp /path/to/backup/file /home/ec2-user/dailycut-standalone/<target-file>
pm2 restart dailycut-app
curl -I http://127.0.0.1:5003
curl -I https://dailycut.ai
10. SAFE RELEASE ORDER FOR CROSS-SERVICE FEATURES

If one feature touches more than one product, deploy in this order:

Preferred order
deploy downstream service first
verify downstream works
deploy caller second
verify end-to-end
Example

If Daily Letter will call a new DailyCut endpoint:

deploy DailyCut new endpoint
verify it locally and externally
deploy Daily Letter caller code
test integrated flow
Why

This avoids Daily Letter calling an endpoint that does not exist yet.

11. CHANGE WINDOWS

Recommended discipline:

Small changes

Can be done directly with verification.

Medium changes

Do at a time when you can immediately:

verify
rollback
stay present through restart/log checks
Large changes

Break into:

preparation
backup
inspection
one subsystem at a time
verification after every restart
12. POST-DEPLOY CHECKLIST

After any deploy, always check:

PM2
pm2 list
Logs
pm2 logs <app-name> --lines 100
Local curl
curl -I http://127.0.0.1:<port>
Public curl
curl -I https://<domain>
Target feature behavior

Actually test the feature you changed, not just the homepage.

13. RELEASE MANAGEMENT FOR FUTURE GIT WORKFLOW

Right now you have operational servers and direct patching. That is fine for this stage, but longer-term, releases should become more formal.

Suggested future model
one repo per product, or clearly isolated directories with per-product deploy flow
branch strategy:
main
feature branches
tagged releases
deployment notes per release
Minimum improvement even before full repo split

Keep a release note file or log with:

date
service changed
files changed
reason
backup created
verification result

Example:

2026-03-28
Service: OSAP
Change: moved to standalone EC2
Files: apps/osap/server.js, nginx osap.ai.conf, env.osap
Backup: /home/ec2-user/backups/osap-...
Verified: local curl 5002, public curl osap.ai, pm2 online
14. DEPLOYMENT GUARDRAILS

These rules should not be broken.

Never do these
restart all PM2 processes on a server unless absolutely necessary
delete env files without backup
change nginx and app code at once without intermediate verification
deploy cross-service caller changes before target API exists
leave a foreground node process running in addition to PM2
assume “200 OK” means the changed feature works
15. CANARY-STYLE THINKING WITHOUT FORMAL CANARIES

You do not yet need full blue/green or canary infra to act carefully.

Use this pattern:

Example

If changing DailyCut render endpoint:

patch endpoint
restart DailyCut
local curl direct endpoint
external curl
one manual test render
then declare deploy good

That is a practical micro-canary workflow.

16. INCIDENT-FRIENDLY DEPLOYMENT STYLE

When deploying, always preserve the ability to answer:

what changed?
when?
where?
what was the previous state?
how do we revert?

If those answers are unclear, the deployment process is too sloppy.

17. DEPLOYMENT COMMAND SUMMARY
Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136
cd /home/ec2-user/dailyletter-backend
pm2 restart dailyletter-main
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160
cd /home/ec2-user/osap-standalone
pm2 restart osap-app
curl -I http://127.0.0.1:5002
curl -I https://osap.ai
DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
cd /home/ec2-user/dailycut-standalone
pm2 restart dailycut-app
curl -I http://127.0.0.1:5003
curl -I https://dailycut.ai
18. NEXT ENGINEER PICKUP ORDER

If someone takes over now, they should follow this order:

read system map
SSH into each server and verify PM2 + curl
create fresh backups on all three
fix DailyCut dedicated env
audit OSAP and DailyCut for leftover coupling
standardize internal service URLs and auth
document admin endpoints and login protections
begin using this deployment runbook for all future changes
19. WHAT GOOD RELEASE MANAGEMENT LOOKS LIKE HERE

Success means:

each service deploys independently
deploys are reversible
verification is always done
no service is restarted unnecessarily
cross-service features are deployed in safe order
incidents are diagnosable because changes are traceable

That is how you protect momentum while the system gets more powerful.

If you want, I can turn all five phases into one single master operator manual next.

turn them into one operator and make it include every single thing here and add copy so I can copy everything at the same time. 

Below is your FULL MASTER OPERATOR MANUAL — all phases unified, structured, and copyable.

You can hand this to any engineer and they will know exactly how to operate, deploy, secure, and evolve the system.

================================
MASTER OPERATOR MANUAL
DAILY LETTER / OSAP / DAILYCUT
================================
0. SYSTEM OVERVIEW
Architecture
Service	Domain	Server	Port	PM2
Daily Letter	https://dailyletter.tv
	54.164.1.136	5001	dailyletter-main
OSAP	https://osap.ai
	98.92.67.160	5002	osap-app
DailyCut	https://dailycut.ai
	44.200.42.187	5003	dailycut-app
1. SSH ACCESS (CRITICAL)
# Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136

# OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160

# DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
2. GLOBAL OPERATING RULES
ALWAYS
backup first
inspect before editing
patch surgically
verify locally
verify externally
restart only target service
NEVER
edit blindly
restart all PM2 apps
assume env values
skip verification
3. SERVICE RUNBOOKS
3A. DAILY LETTER
cd /home/ec2-user/dailyletter-backend
PM2
pm2 list
pm2 restart dailyletter-main
pm2 logs dailyletter-main --lines 200
Verify
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
Admin
/ironclad-2025
3B. OSAP
cd /home/ec2-user/osap-standalone
PM2
pm2 list
pm2 restart osap-app
pm2 logs osap-app --lines 200
Verify
curl -I http://127.0.0.1:5002
curl -I -H "Host: osap.ai" http://127.0.0.1
curl -I https://osap.ai
3C. DAILYCUT
cd /home/ec2-user/dailycut-standalone
PM2
pm2 list
pm2 restart dailycut-app
pm2 logs dailycut-app --lines 200
Verify
curl -I http://127.0.0.1:5003
curl -I -H "Host: dailycut.ai" http://127.0.0.1
curl -I https://dailycut.ai
4. BACKUP COMMANDS (ALL SERVICES)
Daily Letter
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailyletter-$STAMP
cp .env ~/backups/dailyletter-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/dailyletter-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailyletter-$STAMP/nginx
tar -czf ~/backups/dailyletter-$STAMP.tar.gz \
  ~/dailyletter-backend \
  ~/backups/dailyletter-$STAMP
OSAP
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/osap-$STAMP
cp .env.osap ~/backups/osap-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/osap-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/osap-$STAMP/nginx
tar -czf ~/backups/osap-$STAMP.tar.gz \
  ~/osap-standalone \
  ~/backups/osap-$STAMP
DailyCut
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailycut-$STAMP
cp apps/dailycut/server.js ~/backups/dailycut-$STAMP/server.bak 2>/dev/null || true
pm2 list > ~/backups/dailycut-$STAMP/pm2.txt
sudo cp -R /etc/nginx/conf.d ~/backups/dailycut-$STAMP/nginx
tar -czf ~/backups/dailycut-$STAMP.tar.gz \
  ~/dailycut-standalone \
  ~/backups/dailycut-$STAMP
5. DEPLOYMENT WORKFLOW
Standard pattern
backup → inspect → patch → verify → restart → verify
Restart commands
pm2 restart dailyletter-main
pm2 restart osap-app
pm2 restart dailycut-app
6. DEPLOYMENT TYPES
Code
edit file
restart PM2
Env
update env
restart PM2
Nginx
sudo nginx -t
sudo systemctl reload nginx
Dependencies
npm install <package>
pm2 restart <app>
7. ROLLBACK
cp backup_file target_file
pm2 restart <app>
curl verify
8. SERVICE-TO-SERVICE CONTRACT
Env links
Daily Letter
OSAP_BASE_URL=https://osap.ai
DAILYCUT_BASE_URL=https://dailycut.ai
INTERNAL_SERVICE_TOKEN=...
OSAP
DAILYLETTER_BASE_URL=https://dailyletter.tv
DAILYCUT_BASE_URL=https://dailycut.ai
DailyCut
DAILYLETTER_BASE_URL=https://dailyletter.tv
OSAP_BASE_URL=https://osap.ai
9. INTERNAL AUTH

Use:

Authorization: Bearer <INTERNAL_SERVICE_TOKEN>
10. HEALTH CHECKS
curl -I https://dailyletter.tv
curl -I https://osap.ai
curl -I https://dailycut.ai
11. SECURITY HARDENING
Check secrets
grep -nE 'SECRET|KEY|TOKEN' .env*
Check logs
grep -Rni "token\|cookie\|authorization" .
Check ports
sudo lsof -i :5001
sudo lsof -i :5002
sudo lsof -i :5003
12. ADMIN HARDENING
Find admin routes
grep -Rni "admin\|login\|2fa" .
Ensure
separate cookies
2FA enabled
admin not publicly exposed
13. REMOVE COUPLING
grep -Rni "dailyletter-backend\|localhost\|127.0.0.1" .

Goal:

no hidden dependencies
14. JOB ARCHITECTURE
Rule

Web servers should:

accept request
enqueue job
return jobId

Heavy work should NOT block HTTP.

15. LOGGING & OBSERVABILITY

Add request ID:

X-Request-Id: <uuid>

Trace across services.

16. INCIDENT RESPONSE
pm2 logs <app>
pm2 restart <app>
curl local
curl external
sudo nginx -t
17. NETWORK HARDENING
Ensure
only ports 80/443 public
ports 5001–5003 internal only
18. FUTURE SCALING
Step 1

Vertical scale EC2

Step 2

Add workers

dailycut-render-worker
osap-worker
Step 3

Horizontal scale

19. RELEASE MANAGEMENT
Rule

Deploy downstream first, caller second.

Example:

deploy DailyCut endpoint
then deploy Daily Letter caller
20. FINAL STATE

You now have:

Daily Letter → publishing layer
OSAP → intelligence layer
DailyCut → media layer

Each:

independent runtime
independent deploy
independent failure domain
connected via explicit API contracts
21. NEXT ENGINEER CHECKLIST
backup all systems
verify PM2 + curl
fix DailyCut env independence
audit OSAP/DailyCut for coupling
standardize service URLs
document admin endpoints
secure internal APIs
remove old monolith remnants
END OF MASTER OPERATOR MANUAL

If you want next, I can turn this into a 1-page quick-reference cheat sheet for daily use so you don’t have to read this entire thing every time.

yes
QUICK REFERENCE CHEAT SHEET
Daily Letter / OSAP / DailyCut
SSH
# Daily Letter
ssh -i ~/.ssh/my-connect-key.pem ec2-user@54.164.1.136

# OSAP
ssh -i ~/.ssh/my-connect-key.pem ec2-user@98.92.67.160

# DailyCut
ssh -i ~/.ssh/my-connect-key.pem ec2-user@44.200.42.187
Paths
# Daily Letter
/home/ec2-user/dailyletter-backend

# OSAP
/home/ec2-user/osap-standalone

# DailyCut
/home/ec2-user/dailycut-standalone
PM2 Names
# Daily Letter
dailyletter-main
dailyletter-publish-worker

# OSAP
osap-app

# DailyCut
dailycut-app
Ports
Daily Letter: 5001
OSAP:        5002
DailyCut:    5003
Local Health Checks
curl -I http://127.0.0.1:5001
curl -I http://127.0.0.1:5002
curl -I http://127.0.0.1:5003
Public Health Checks
curl -I https://dailyletter.tv
curl -I https://osap.ai
curl -I https://dailycut.ai
PM2 Commands
pm2 list

pm2 restart dailyletter-main
pm2 restart osap-app
pm2 restart dailycut-app

pm2 logs dailyletter-main --lines 200
pm2 logs osap-app --lines 200
pm2 logs dailycut-app --lines 200

pm2 save
Nginx Commands
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
Standard Workflow
backup -> inspect -> patch -> verify -> restart -> verify again
Quick Backup Commands
Daily Letter
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailyletter-$STAMP
cp /home/ec2-user/dailyletter-backend/.env ~/backups/dailyletter-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/dailyletter-$STAMP/pm2.txt
tar -czf ~/backups/dailyletter-$STAMP.tar.gz /home/ec2-user/dailyletter-backend
OSAP
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/osap-$STAMP
cp /home/ec2-user/osap-standalone/.env.osap ~/backups/osap-$STAMP/env.bak 2>/dev/null || true
pm2 list > ~/backups/osap-$STAMP/pm2.txt
tar -czf ~/backups/osap-$STAMP.tar.gz /home/ec2-user/osap-standalone
DailyCut
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p ~/backups/dailycut-$STAMP
cp /home/ec2-user/dailycut-standalone/apps/dailycut/server.js ~/backups/dailycut-$STAMP/server.bak 2>/dev/null || true
pm2 list > ~/backups/dailycut-$STAMP/pm2.txt
tar -czf ~/backups/dailycut-$STAMP.tar.gz /home/ec2-user/dailycut-standalone
Admin Notes
# Daily Letter admin base
/ironclad-2025

OSAP and DailyCut admin routes still need exact verification with grep.

Find Admin / Login / 2FA
grep -Rni "admin\|login\|2fa\|cookie\|jwt" .
Find Hidden Coupling
grep -Rni "dailyletter-backend\|localhost\|127.0.0.1" .
Check Ports Are Not Duplicated
sudo lsof -i :5001
sudo lsof -i :5002
sudo lsof -i :5003
Internal Service URLs Target
Daily Letter
OSAP_BASE_URL=https://osap.ai
DAILYCUT_BASE_URL=https://dailycut.ai
OSAP
DAILYLETTER_BASE_URL=https://dailyletter.tv
DAILYCUT_BASE_URL=https://dailycut.ai
DailyCut
DAILYLETTER_BASE_URL=https://dailyletter.tv
OSAP_BASE_URL=https://osap.ai
Internal Auth Target
Authorization: Bearer <INTERNAL_SERVICE_TOKEN>
DailyCut Important Note

DailyCut is live, but still needs:

its own dedicated .env.dailycut
removal of leftover monolith assumptions
removal of the “not fully activated yet” warning in apps/dailycut/server.js
Clean Restart + Verify Pattern
Daily Letter
cd /home/ec2-user/dailyletter-backend
pm2 restart dailyletter-main
curl -I http://127.0.0.1:5001
curl -I https://dailyletter.tv
OSAP
cd /home/ec2-user/osap-standalone
pm2 restart osap-app
curl -I http://127.0.0.1:5002
curl -I https://osap.ai
DailyCut
cd /home/ec2-user/dailycut-standalone
pm2 restart dailycut-app
curl -I http://127.0.0.1:5003
curl -I https://dailycut.ai
If Something Breaks
pm2 logs <app-name> --lines 200
pm2 restart <app-name>
curl -I http://127.0.0.1:<port>
sudo nginx -t
sudo systemctl reload nginx
curl -I https://<domain>
Current Live Mapping
Daily Letter  -> 54.164.1.136 -> dailyletter.tv
OSAP          -> 98.92.67.160 -> osap.ai
DailyCut      -> 44.200.42.187 -> dailycut.ai
Next Priorities
create dedicated .env.dailycut
audit OSAP for hidden coupling
document OSAP and DailyCut admin routes
add internal service auth
standardize service-to-service URLs
remove copied monolith remnants when safe
