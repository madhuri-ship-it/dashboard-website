# Vurve Executive Performance Intelligence: Handover and Deployment Guide

For the person who will run, deploy or support the web dashboard. Written after a full test deployment on 7 October 2026. Everything marked "tested" was run end to end in that test. Anything marked "not tested" is called out so you know where to be careful.

---

## 1. What this is

A login-protected web page for the Vurve founder, admins and salon managers.

- Three tabs: Executive Command Center, Salon Performance Ops, Employee Productivity Index.
- Sign-in is with the person's `@vurvesalon.com` Google account. No Claude account is needed.
- Each person sees only the salons they are allowed to see. Other salons' numbers are never sent to their browser.
- Staff salary figures are hidden unless a person is specifically given that right.
- Data refreshes by itself from two Excel files on Google Drive. Nobody runs anything by hand.

The older Claude artifact is a separate copy. It does not update by itself. The web version replaces it.

## 2. How it works

```
Drive files (outlet sales + employee history)
        |
        v
Cloud Run job "vurve-refresh"  <-- Cloud Scheduler, hourly 07:00 to 21:00 India time
        |   rebuilds only when a Drive file changed
        v
Private Cloud Storage bucket: data.json, access.json, state.json, config/
        |
        v
Cloud Run service "vurve-dashboard"  <-- Google sign-in (Identity-Aware Proxy, IAP)
        |   checks the email against access.json, removes other salons and salary, builds the page
        v
Manager's browser
```

| Piece | Purpose |
|---|---|
| Cloud Run service `vurve-dashboard` | The website. Behind Google sign-in. Verifies the IAP signed token itself. |
| Cloud Run job `vurve-refresh` | Reads the two Drive workbooks and the access sheet and rebuilds the data. |
| Cloud Scheduler `vurve-refresh-hourly` | Starts the job every hour in working hours. |
| Cloud Storage bucket | Private store for the data and config files. |
| Service account `vurve-dash-sa` | The identity the website and job run as. |
| Google Sheet "Vurve Dashboard Access" | The only place that decides who sees what. |

## 3. Folder guide (the code package)

| Path | What it holds |
|---|---|
| `app/` | The website: `main.py`, sign-in check `auth.py`, who-sees-what `access.py`, storage `store.py`, templates, charts library |
| `refresh/` | The rebuild: `build_data.py` (all calculations), `drive_io.py`, `access_sheet.py`, `refresh_job.py` |
| `config/targets.json` | Monthly salon targets and tier months (currently the September figures) |
| `seed/incentive.xlsx` | Staff targets and salaries (August incentive report). **Contains salaries.** |
| `deploy/` | `DEPLOY.md` and `access_sheet_template.xlsx` |
| `tests/`, `tools/` | Automatic checks and a local-run helper |
| `Dockerfile`, `requirements.txt` | Container build |

## 4. Rules the dashboard uses (so numbers can be explained)

- Service ACH = Service + Packages revenue. Total sales = Service ACH + Product ACH + Membership sold.
- Clients = New guests + Old guests. Average bill (ABV) = Service ACH divided by Clients.
- Pro-rated target = monthly target divided by 30, times days in the period. Achievement % = ACH divided by target.
- Change % compares the selected period with an equal-length period just before it.
- Tiers use the average monthly Service ACH over April to September: Top at 45 lakh or more, Mid 25 to 45 lakh, Low below 25 lakh.
- Staff are grouped by role using the first matching keyword in the role text.

## 5. Who sees what: the access sheet

A Google Sheet named **Vurve Dashboard Access** with two tabs. Start from `deploy/access_sheet_template.xlsx` (upload it to Drive, then File, Save as Google Sheets).

**Clusters tab** (columns: Cluster, Salons). Salons are separated by commas.

**Users tab** (columns: Email, Name, Role, Clusters, Salons, Salary)

| Column | Meaning |
|---|---|
| Email | The person's work Google email |
| Role | `admin` sees all salons. `manager` (or blank) sees only their salons. |
| Clusters | One or more cluster names from the Clusters tab, comma separated |
| Salons | Optional extra individual salons for that person |
| Salary | `yes` lets them see staff salary. Blank or `no` hides it. |

Rules:
- Salon names must match the dashboard exactly (for example `T Nagar`, `Nungambakkam`, `Sadashiv Nagar`). A name that does not match is ignored and logged as `access_warning`.
- **Delete the template's example rows.** In the test they stayed behind and showed as extra users. Check the log says the right number of users after each change.
- An email that is not on the sheet sees "No access". A manager with no salons is also refused.
- Edits take effect on the next refresh run (within the hour).

Test values used: folder ID `1UdL0vnyHwQNa6Q32v4Sg_qmC83ypVFJy`; access sheet ID `1_aFRmD2uif4zLT6n7JdGhG_9trxoLzD_VuF1-tFBRKg`. For production use your own IDs. The sheet ID is the long text between `/d/` and `/edit` in the sheet's address. Do not use the short number after `gid=`, which is only the tab number.

## 6. Deployment step by step

Replace the capital words. Commands in sections 6.1 to 6.9 were run in the test unless noted.

### 6.1 Before you start

- A Google Cloud project **inside your company organization** (`vurvesalon.com`) with billing on. IAP on Cloud Run needs a project that belongs to an organization, so a personal Gmail project will not work.
- Owner or Editor on the project, plus permission to link billing.
- The `gcloud` command-line tool installed and logged in as your admin account.
  - On a Mac without Homebrew, download Google's archive from cloud.google.com/sdk/docs/install, unpack it and run `./google-cloud-sdk/install.sh`. Open a new Terminal window afterwards.
  - `gcloud auth login`, then `gcloud config set project PROJECT_ID`.
- The project unzipped on your computer. Run all commands from inside the unzipped folder (the one that contains `requirements.txt`).

Set the names (repeat these in every new Terminal window):

```
export PROJECT_ID=PROJECT_ID
export REGION=asia-south1
export BUCKET=${PROJECT_ID}-vurve-dashboard-data
export SA=vurve-dash-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

Check billing is on (must say `billingEnabled: true`):

```
gcloud billing projects describe $PROJECT_ID
```

### 6.2 Switch on the services

```
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com storage.googleapis.com drive.googleapis.com sheets.googleapis.com cloudscheduler.googleapis.com iap.googleapis.com
```

The first website deploy may also ask to enable `cloudresourcemanager.googleapis.com`. Answer `y`.

### 6.3 Private bucket and service account

Create both **before** granting permission. In the test, running the permission command first failed with a 404.

```
gcloud storage buckets create gs://$BUCKET --location=$REGION --uniform-bucket-level-access --public-access-prevention
gcloud iam service-accounts create vurve-dash-sa --display-name="Vurve dashboard"
gcloud storage buckets add-iam-policy-binding gs://$BUCKET --member=serviceAccount:$SA --role=roles/storage.objectAdmin
```

### 6.4 Upload the starting config

```
gcloud storage cp config/targets.json gs://$BUCKET/config/targets.json
gcloud storage cp seed/incentive.xlsx gs://$BUCKET/config/incentive.xlsx
gcloud storage ls gs://$BUCKET/config/
```

`incentive.xlsx` contains staff salaries. Keep the bucket private. After the upload, delete the local copy of the `seed` folder if the laptop is shared.

### 6.5 Share Drive files with the service account

In Google Drive, share each of these with `vurve-dash-sa@PROJECT_ID.iam.gserviceaccount.com` as **Viewer** (untick "Notify people"):

- The folder holding `outlet_sales_history_2026.xlsx` and `employee_history_2026.xlsx` (the folder ID is the text after `/folders/` in its address).
- The **Vurve Dashboard Access** sheet.

If Drive says you cannot share outside the organization, a Workspace admin must allow it: Admin console, Apps, Google Workspace, Drive and Docs, Sharing settings.

### 6.6 Deploy the refresh job and run it once

```
gcloud run jobs deploy vurve-refresh --source . --region $REGION --service-account $SA --command python --args=-m,refresh.refresh_job --max-retries 1 --task-timeout 900s --memory 1Gi --set-env-vars DATA_BUCKET=$BUCKET,DRIVE_FOLDER_ID=FOLDER_ID,ACCESS_SHEET_ID=SHEET_ID
```

Answer `y` if it offers to create an Artifact Registry repository. The build takes a few minutes. Then run it:

```
gcloud run jobs execute vurve-refresh --region $REGION --wait
```

Check the result. The log lines are JSON, so read the JSON fields (reading the plain text field shows blank lines):

```
gcloud storage ls -l gs://$BUCKET/
gcloud logging read 'resource.type="cloud_run_job" AND resource.labels.job_name="vurve-refresh"' --limit 10 --format="value(jsonPayload.event,jsonPayload.through,jsonPayload.salons,jsonPayload.staff,jsonPayload.users,jsonPayload.clusters,jsonPayload.message)" --freshness=30m
```

Expect `data.json`, `access.json` and `state.json` in the bucket, plus `data_published <date> 15 486` and `access_published <users> <clusters>` in the log. Newest log line is first. A harmless "httplib2 transport does not support per-request timeout" warning is normal.

If the log says a file was not found, it is not shared with the service account (6.5). If it complains about Drive or Sheets permission or scope, create a key for the service account, store it in Secret Manager and mount it as `GOOGLE_APPLICATION_CREDENTIALS`. This was not needed in the test.

### 6.7 Deploy the website

```
gcloud beta run deploy vurve-dashboard --source . --region $REGION --service-account $SA --no-allow-unauthenticated --iap --min-instances 0 --max-instances 3 --memory 512Mi --set-env-vars DATA_BUCKET=$BUCKET,ALLOWED_DOMAINS=vurvesalon.com
```

Note the `Service URL` it prints. Do not open it yet.

### 6.8 Let people through the sign-in, and set the audience

```
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud run services add-iam-policy-binding vurve-dashboard --region $REGION --member=serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-iap.iam.gserviceaccount.com --role=roles/run.invoker
```

Then allow people through the Google sign-in. For a test, allow one person:

```
gcloud beta iap web add-iam-policy-binding --resource-type=cloud-run --service=vurve-dashboard --region=$REGION --member=user:YOU@vurvesalon.com --role=roles/iap.httpsResourceAccessor
```

For production, allow the whole company domain. The app then checks every person against the access sheet and shows "No access" to anyone not listed:

```
gcloud beta iap web add-iam-policy-binding --resource-type=cloud-run --service=vurve-dashboard --region=$REGION --member=domain:vurvesalon.com --role=roles/iap.httpsResourceAccessor
```

Now give the app its audience value. Without it everyone gets "Sign-in required" (a safe failure):

```
gcloud run services update vurve-dashboard --region $REGION --update-env-vars IAP_AUDIENCE="/projects/${PROJECT_NUMBER}/locations/${REGION}/services/vurve-dashboard"
```

That audience format worked in the test. If it ever gives "Sign-in required" after a Google change, read the exact value in Cloud Console, Security, Identity-Aware Proxy, `vurve-dashboard`, "Get JWT audience", and set `IAP_AUDIENCE` to it.

### 6.9 Schedule the refresh

```
gcloud iam service-accounts create vurve-scheduler
gcloud run jobs add-iam-policy-binding vurve-refresh --region $REGION --member=serviceAccount:vurve-scheduler@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/run.invoker
gcloud scheduler jobs create http vurve-refresh-hourly --location $REGION --schedule="0 7-21 * * *" --time-zone="Asia/Kolkata" --http-method=POST --uri="https://${REGION}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${PROJECT_ID}/jobs/vurve-refresh:run" --oauth-service-account-email=vurve-scheduler@${PROJECT_ID}.iam.gserviceaccount.com
gcloud scheduler jobs run vurve-refresh-hourly --location $REGION
gcloud run jobs executions list --region $REGION --limit 3
```

**Not yet confirmed in the test.** After running these, check that a new execution appears in the list. If the scheduler call is refused, Cloud Console, Cloud Scheduler shows the error. The job is safe to run as often as you like. It only rebuilds when a Drive file changed.

### 6.10 Test before telling anyone

1. Sign in as an admin: you should see all salons and all three tabs with charts.
2. Make yourself a manager in the access sheet for a moment, run the job, refresh the page after a minute (the page caches data for 60 seconds): you should see only that cluster's salons and no salary. Right-click, View page source, and search for another cluster's salon name. It must not be there.
3. Someone not on the sheet sees "No access". A private window that is not signed in is sent to Google sign-in and never sees data.
4. Put your own role back to admin.

## 7. Day-to-day

| Task | What to do |
|---|---|
| New day's data | Nothing. The Zenoti files land on Drive and the next hourly run picks them up. |
| Add or remove a manager, change a cluster | Edit the access sheet. Live within the hour. |
| New month's salon targets | Edit `config/targets.json`, then `gcloud storage cp config/targets.json gs://$BUCKET/config/targets.json` and force a rebuild (below). |
| Staff target or salary changes | Replace `seed/incentive.xlsx` (same name and sheet layout) and `gcloud storage cp seed/incentive.xlsx gs://$BUCKET/config/incentive.xlsx`, then force a rebuild. |
| Force a rebuild now | `gcloud run jobs execute vurve-refresh --region $REGION --update-env-vars FORCE=1 --wait` |
| Update the code | Redeploy both the job (6.6) and the website (6.7) from the new folder. |
| See who looked at what | Cloud Logging. The website logs one line per page view with the email and salon count. |

The bucket copies of `targets.json` and `incentive.xlsx` are the ones the job reads. Editing files on a laptop changes nothing until they are uploaded.

## 7b. Doing it in the Google Cloud Console (no commands)

Menu names can shift slightly when Google updates the Console. Use the search bar at the top ("Search for resources, docs, products") if a name has moved. Always check the project selector at the top shows your project first.

### Where the data, targets and incentive files are

1. Console menu (three lines, top left), **Cloud Storage**, **Buckets**.
2. Open the bucket named `PROJECT_ID-vurve-dashboard-data`.
3. You will see:
   - `data.json`: the published dashboard data. **It includes staff salary figures. Do not download or share it casually.**
   - `access.json`: who may see what, built from the access sheet.
   - `state.json`: the last Drive file times and last run, used to decide whether to rebuild.
   - a folder `config/` containing `targets.json` (salon targets) and `incentive.xlsx` (staff targets and salaries).
4. Click a file name to see its details. Use **Download** to save a copy to your computer.

### Replace the targets or incentive file

1. Open the bucket, then the `config` folder.
2. Click **Upload files** and choose the new `targets.json` or `incentive.xlsx`. The name must be exactly the same, and it replaces the old one.
3. Then force a rebuild (next section).

### Run the refresh job now, and read its log

1. Menu, **Cloud Run**, **Jobs**, click `vurve-refresh`.
2. Click **Execute** to run it. To force a full rebuild after changing targets or the incentive file, click **Execute with overrides**, add an environment variable `FORCE` with value `1`, and run it.
3. Open the **Executions** tab to see each run and whether it succeeded.
4. Click an execution, then **Logs**. Look for `data_published`, `access_published`, `nothing_to_do` (no change on Drive) and `refresh_failed` (a problem). The log also holds `access_warning` lines naming a salon or cluster that did not match.

### Website address and settings

1. Menu, **Cloud Run**, **Services**, click `vurve-dashboard`. The address is at the top.
2. The **Revisions** and **Logs** tabs show deployments and page views (one line per view, with the email and salon count).
3. To edit settings such as `IAP_AUDIENCE`, click **Edit and deploy new revision**, open **Variables and secrets**, change the value and deploy.

### Who may open the site (Google sign-in)

1. Menu, **Security**, **Identity-Aware Proxy**.
2. Find `vurve-dashboard` in the list. Tick it, and use the side panel's **Add principal** to allow a person or the whole domain `vurvesalon.com`, with the role **IAP-secured Web App User**.
3. The three-dot menu on that row, **Get JWT audience**, shows the exact audience value to use for `IAP_AUDIENCE`.

### Scheduler

Menu, **Cloud Scheduler**. The job `vurve-refresh-hourly` shows its schedule and last result. Use **Force run** to test it. If the last result shows a failure, click the job to see the error.

### Billing and alerts

- Menu, **Billing**, **Reports** shows what the project has cost so far.
- Menu, **Monitoring**, **Alerting** is where you create an alert for the log event `refresh_failed`.

### Service account and sharing checks

- Menu, **IAM and Admin**, **Service Accounts** lists `vurve-dash-sa`. Its email is the address you share the Drive folder and the access sheet with.
- **IAM and Admin**, **IAM** lists who can administer the project. Keep that list short.

## 8. Run it on your own computer (no Google Cloud)

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python tools/build_local.py OUTLET.xlsx EMPLOYEE.xlsx seed/incentive.xlsx you@vurvesalon.com
CACHE_SECONDS=0 DEV_USER=you@vurvesalon.com flask --app app.main run
```

Open http://127.0.0.1:5000. `DEV_USER` skips Google sign-in and is refused automatically when the code runs on Cloud Run. Run all of this from the project folder.

Automatic checks (19 access checks passed in the test):

```
python tests/test_access_web.py local_data/data.json
```

One "DEV_USER must not be set on Cloud Run" error trace in that output is expected. That check deliberately simulates Cloud Run and confirms the bypass is refused.

## 9. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| "command not found: gcloud" or "python" in Terminal | Open a new Terminal window. For `python`, you are outside the project or the virtual environment is off: `cd` into the folder and `source .venv/bin/activate`. |
| Bucket permission command says 404 | The bucket was not created yet (6.3). |
| Job log says file not found | Folder, files or access sheet not shared with the service account, or a wrong ID. |
| Access sheet ID rejected | You copied the `gid=` tab number instead of the long ID between `/d/` and `/edit`. |
| More users than expected after the job | Example rows remain in the Users tab. Delete them and run the job again. |
| `access_warning ... unknown salon` | A salon name in the sheet does not match the dashboard's spelling exactly. |
| Everyone sees "Sign-in required" or a 401 page | `IAP_AUDIENCE` is missing or wrong (6.8). |
| Google says "You don't have access" | The person has not been added to the IAP access role (6.8). Allow a minute after changing it. |
| "No access yet" or a 403 page | Sign-in worked but the email is not on the access sheet, or the manager has no salons. |
| Page shows old data | The job has not run since the Drive files changed. Check the scheduler (6.9), then run the job. |
| Today's figures look too low | The Drive files may be incomplete for the latest days. The page shows only what the files contain. |
| Job log text shows blank lines | Read the JSON fields (`jsonPayload.event` and others), not `textPayload` (6.6). |

## 10. Security notes

- No one reaches the page without Google sign-in. The app also verifies the signed IAP token itself, so a forged header is refused (tested).
- The server removes other salons and salary before the page is built. They are not hidden in the browser, they are never sent (tested).
- The bucket blocks public access. Data and `incentive.xlsx` (with salaries) are only readable by the service account and project admins.
- Pages are served with no-store caching, a content security policy, and no-referrer headers.
- Only people who can edit the access sheet and the IAP role control access. Limit edit rights on both.

## 11. Known limits

- **Targets:** one set of salon targets (September) applies to every month, including October. Per-month targets need a code change. Staff targets and salaries are the August incentive figures until a new file is uploaded.
- "Catch-up per day" on the salon cards divides the gap by a fixed 31 days instead of the days remaining in the month. Known and not yet fixed.
- For a manager, "chain" comparisons mean the average of the salons they can see (the page says "cluster").
- The hourly rebuild needs both Drive files to keep the same names (`outlet_sales_history_2026.xlsx`, `employee_history_2026.xlsx`) and sit in the one shared folder. A new year's file name needs the job's `OUTLET_FILE_NAME` and `EMP_FILE_NAME` settings changed.
- Not verified from where this package was built: current Google Cloud pricing. Expect a very small bill (the service scales to zero) and confirm on the billing page after a week of use.

## 12. Handover checklist

- [ ] Project, bucket, service account and billing owner noted: ______________________
- [ ] Drive folder ID and access sheet ID recorded: ______________________
- [ ] Access sheet has only real people and real clusters (no example rows)
- [ ] Admin can see all salons. A test manager sees only their salons and no salary.
- [ ] Scheduler created, and a new job execution appears each hour
- [ ] Alert set up for job failures (log event `refresh_failed`) in Cloud Monitoring
- [ ] IAP set to the company domain for launch
- [ ] Local copy of `seed/incentive.xlsx` removed from shared laptops
- [ ] Two people besides the builder know how to edit the access sheet and force a rebuild
- [ ] October targets and the staff incentive file updated when received
