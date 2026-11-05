---
name: serdes-data-sync
description: Sync the latest SerDes/FEC training data from S3 and prepare it for model work. Use when the user asks to refresh, sync, copy or update the production (ai-prod) data, the Helios data, or to convert Helios counters to the production hourly Parquet schema. First checks that the AWS profiles ai-prod and ai-helios are configured and that both S3 buckets can be read. Produces two training inputs, /opt/ai-prod and /opt/ai-helios. Does not train a model or choose bin boundaries.
---

# SerDes data sync

This skill gets two data sets ready for model work:

| Input | Location | Source | AWS profile |
|---|---|---|---|
| Production hourly Parquet, one organization | `/opt/ai-prod` | `s3://usw2-ai-ds-prod-ml-training-acs`, `org_id=f99aa5ac-f689-47e2-b1ba-c82d396a7311` only | `ai-prod` |
| Helios raw counters | `/opt/raw_helios/interface_telemetry` | `s3://usw2-ai-ds-dev-optical-datalake/helios/interface_telemetry/` | `ai-helios` |
| Helios hourly Parquet, converted | `/opt/ai-helios`, partition `org_id=helios` | Made from the Helios raw counters (local, no AWS) | none |

Helios raw data is five-minute cumulative counters. The conversion makes it the same
as the production hourly tables (same columns and types). Production data needs no
conversion.

The other skills use these two locations. This skill does not train a model and does
not choose bin boundaries.

## Production organization

The production data is available for this organization:

| Name | `org_id` | Use |
|---|---|---|
| Production analysis organization | `f99aa5ac-f689-47e2-b1ba-c82d396a7311` | The default. Every earlier analysis and model used this organization. |

- Sync production data for one `org_id` only. Never sync the whole bucket, a whole
  dataset prefix or a whole day without the `org_id` filter.
- Use the `org_id` the user gives. If the user gives none, use the default in the
  table. If the user asks for a different organization, or the request is unclear,
  ask the user for the `org_id` and wait. Do not guess one, and do not list the
  bucket to find one.
- Check that the value is a UUID (8-4-4-4-12 lowercase hex characters) before using it.
- Syncing never copies other organizations, but the local `/opt/ai-prod` already
  holds some. On 2026-11-04 the `interface-dom-1H` dataset had 1,343 partitions for
  `org_id=645da796-192f-4996-af43-33ab773fb961`, copied by an earlier script that
  had two organizations written into it. `interface-fec-histogram-1H` held only
  the default organization. Do not delete those partitions. Readers select one
  `org_id`, so they do not affect training.

## Rules

- Sync only adds files. Never use `--delete`, and never remove local data except as
  step 4 describes.
- Never print, store or commit credentials. Use the named AWS profiles from the
  user's `~/.aws/credentials`. Use `ai-prod` for the production bucket and
  `ai-helios` for the Helios bucket. Never use the `default` profile or any other
  profile, and never pass keys on the command line.
- Never write raw data, Parquet files or logs into the repository.
- Run all Python and notebooks with `/home/ubuntu/venv311`. When a notebook needs a
  kernel, select the `python3` kernel of that environment
  (`/home/ubuntu/venv311/share/jupyter/kernels/python3`). Do not use `venv38` or
  the system Python, which has no `pandas`.
- In `/opt/ai-helios`, write and delete only the `org_id=helios` partitions. The
  folder can also hold the cluster's own hourly data under another `org_id`
  (for example `org_id=29077a66-11ba-4e6d-b64f-9775df6517e7`). Never change,
  delete or overwrite those partitions or the `_SUCCESS` files.
- If a step fails, stop and report the exact error. Do not try another source.

## Step 1: Check profiles and bucket access first

Do this step before anything else. Run no sync or conversion until every check in
1 to 3 passes. If one check fails, stop the whole skill, report which check failed
with the exact error, and tell the user what to fix. Do not skip a failed check and
do not continue with only one of the two sources.

1. Check that both profiles are configured. The installed AWS CLI is version 1,
   which has no `list-profiles`. Run, for each of `ai-prod` and `ai-helios`:
   `aws configure list --profile <name> >/dev/null 2>&1 && echo ok || echo missing`.
   Do not print the output of `aws configure list` without that redirect.
2. Check that each profile has valid credentials:
   `aws sts get-caller-identity --profile ai-prod` and
   `aws sts get-caller-identity --profile ai-helios`.
3. Check that each bucket can be read with its own profile. Each command lists one
   prefix and must exit with code 0:

   ```bash
   aws s3 ls s3://usw2-ai-ds-prod-ml-training-acs/interface-fec-histogram-1H/ \
     --profile ai-prod --region us-west-2 | tail -3
   aws s3 ls s3://usw2-ai-ds-dev-optical-datalake/helios/interface_telemetry/ \
     --profile ai-helios --region us-west-2 | tail -3
   ```

   Check the exit code of `aws`, not of `tail`. If the output is empty with exit
   code 0, the prefix is empty. Report that and stop. An `AccessDenied` error on
   `s3:ListBucket` means the profile has no read access to that bucket.
4. Run `df -h /opt`. Stop if less than 20 GB is free.
5. Check that the Python environment `/home/ubuntu/venv311` exists, and that
   `/home/ubuntu/venv311/bin/python -c "import pandas, pyarrow, numpy"` runs.
6. Check that the `helios-telemetry` repository is at
   `/home/ubuntu/workspace/helios-telemetry` and has
   `notebooks/serdes/helios_etl.py`. Step 4 needs it. This skill sits in the
   `helios-skills` repository, a sibling folder.

Report the result of the checks in one short table (check, profile or bucket,
pass or fail) before step 2.

## Step 2: Sync production data for one organization

Do not use `/opt/ai-prod/copy_s3_partitions.sh`. It has two `org_id` values written
into it and cannot take one as a setting, so it would copy a second organization.
Sync day by day with `aws s3 sync`, and exclude everything except the chosen
`org_id`. The model needs only `interface-fec-histogram-1H` and `interface-dom-1H`,
so sync only those two datasets.

1. Decide the `org_id` as described in "Production organization". Stop and ask
   the user if it is not clear.
2. Before syncing, run the `find` command from item 6 and keep the counts. Then
   find the newest local day:
   `ls /opt/ai-prod/interface-fec-histogram-1H | sort | tail -1`
3. Start from that day, not the day after it. The newest local day can be partial,
   because the data is stored in hourly Parquet files and the hourly ETL adds hours
   over time. Starting from the newest local day syncs that day again, so it fixes
   the newest day. `aws s3 sync` copies only files that are missing or changed, so
   the repeat is cheap. It does not fix an older day with a missing hour.
   The end day is today in UTC.
4. Run the loop below with `DRYRUN=--dryrun` first. It prints the files it would
   copy and copies nothing. Then run it again with `DRYRUN=` (empty) if the
   dry run lists files.

   ```bash
   ORG=f99aa5ac-f689-47e2-b1ba-c82d396a7311   # or the org_id the user gave
   START=<YYYY-MM-DD>; END=<YYYY-MM-DD>; DRYRUN=--dryrun
   [[ "$ORG" =~ ^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$ ]] || { echo "org_id is not a UUID"; exit 1; }
   d="$START"
   while ! [[ "$d" > "$END" ]]; do
     for p in interface-fec-histogram-1H interface-dom-1H; do
       src="s3://usw2-ai-ds-prod-ml-training-acs/$p/dt=$d/"
       if ! aws s3 ls "$src" --profile ai-prod --region us-west-2 >/dev/null 2>&1; then
         echo "SKIP $p dt=$d (not in source)"; continue
       fi
       aws s3 sync "$src" "/opt/ai-prod/$p/dt=$d/" --exclude "*" --include "*org_id=$ORG/*" \
         --profile ai-prod --region us-west-2 --only-show-errors $DRYRUN \
         && echo "OK   $p dt=$d" || echo "FAIL $p dt=$d"
     done
     d=$(date -u -d "$d + 1 day" +%F)
   done
   ```

   The `--exclude "*"` before `--include` is required. Without it the sync copies
   every organization. Do not remove it, and do not remove `--include`.
5. A `SKIP` line can mean the day is missing in the bucket, or that the profile
   cannot list it. If every day is `SKIP`, run the bucket check from step 1 again
   and stop. If a line says `FAIL`, run that day again once, then stop and report
   the error. An `AccessDenied` error on `s3:ListBucket` means the `ai-prod`
   profile has no read access. Do not try another profile.
6. After the sync, check whether other organizations are present in the two datasets:

   ```bash
   find /opt/ai-prod/interface-fec-histogram-1H /opt/ai-prod/interface-dom-1H \
     -type d -name 'org_id=*' ! -name "org_id=$ORG" | sed 's/.*org_id=//' | sort | uniq -c
   ```

   Compare the counts with the previous run. Folders from an organization already
   present before this run are expected (see "Production organization"). If the
   counts grew, or a new `org_id` appears, the sync copied another organization:
   stop, report it, and do not delete anything.
7. Report: the `org_id`, the newest `dt=` and `hr=` in `interface-fec-histogram-1H`
   and `interface-dom-1H`, and the days skipped as "not in source".

## Step 3: Sync Helios raw data

```bash
aws s3 sync s3://usw2-ai-ds-dev-optical-datalake/helios/interface_telemetry/ \
  /opt/raw_helios/interface_telemetry --profile ai-helios --region us-west-2 \
  --only-show-errors --no-progress
```

If this fails with `AccessDenied`, the `ai-helios` profile cannot read the Helios
bucket. Stop and tell the user. Do not try another profile.

1. Count Parquet files before and after:
   `find /opt/raw_helios/interface_telemetry -name '*.parquet' | wc -l`
2. Report the number of new files and the newest `dt=` and `hr=` folder.
3. The server clock can be about 5 minutes behind. The newest hour is normally
   about 1 to 2 hours old, because the hourly job uploads after the hour closes.

## Step 4: Convert Helios data to the production schema

The conversion code is `/home/ubuntu/workspace/helios-telemetry/notebooks/serdes/helios_etl.py`.
It follows the production
ETL rules (per-bin counter deltas, hourly bin percent, hourly codeword rates) with
the Helios settings in `HeliosEtlConfig`. Use the defaults.

Reading about 2.4 million rows took about 85 seconds.

```bash
cd /home/ubuntu/workspace/helios-telemetry/notebooks/serdes && /home/ubuntu/venv311/bin/python - <<'EOF'
import json, sys
from pathlib import Path
sys.path.insert(0, ".")
from helios_etl import HeliosEtlConfig, build_hourly_datasets, read_helios_counters, write_hourly_datasets

config = HeliosEtlConfig()
raw = read_helios_counters(Path("/opt/raw_helios/interface_telemetry"))
histogram, dom, audit = build_hourly_datasets(raw, config)
del raw
print(json.dumps(audit, indent=2))
print(write_hourly_datasets(histogram, dom, Path("/opt/ai-helios"), config))
EOF
```

- The writer replaces only the `org_id=helios` partitions, so it is safe to run again.
  It rebuilds the whole date range each time.
- Output layout: `/opt/ai-helios/<dataset>/dt=YYYY-MM-DD/hr=HH/org_id=helios/`.
- `/opt/ai-helios` can hold two Helios sources side by side: the converted data
  (`org_id=helios`) and the cluster's data (another `org_id`). Readers must select
  one `org_id`. `read_hourly` in `fec_model.py` does this. Notebook
  `03_score_helios.ipynb` stops with "Expected one org_id" when it finds more than
  one. Report this to the user when the folder holds both.
- Do not change `HeliosEtlConfig`. The defaults keep only 400G ports, set a delta
  across a collection gap longer than 900 s to zero, and need 90% of each hour's
  counter time. Blank optical columns, BER and CRC rates stay null.

## Step 5: Check the result

1. **Schema.** For each of `interface-fec-histogram-1H` and `interface-dom-1H`, compare
   the column names and types of one production file with one converted file. They
   must be the same. Stop and report any difference.

   ```bash
   /home/ubuntu/venv311/bin/python - <<'EOF'
   from pathlib import Path
   import pyarrow.parquet as pq
   for ds in ("interface-fec-histogram-1H", "interface-dom-1H"):
       prod = next(Path("/opt/ai-prod", ds).rglob("*.parquet"))
       conv = next(Path("/opt/ai-helios", ds).rglob("org_id=helios/*.parquet"))
       same = [(f.name, str(f.type)) for f in pq.read_schema(prod)] == [(f.name, str(f.type)) for f in pq.read_schema(conv)]
       print(ds, "match" if same else "DIFFERENT")
   EOF
   ```

2. **Freshness.** Report the first and last hour of each input. Say clearly when the
   production data ends well before the Helios data. The two sources can end on
   different days.
3. **Audit.** Report the conversion audit values: raw rows, interfaces, counter
   resets, gaps set to zero, and hours dropped for low coverage.

## Final report

Give a short table: for production and for converted Helios, the first hour, the
last hour, the number of interfaces and the number of hourly rows. Then list any
warning, such as skipped days, a stale production copy or a schema difference.

State these limits when they apply:

- The conversion was tested for schema match. The values are not validated against
  the cluster's own hourly data in `/opt/ai-helios` (the other `org_id`).
- The Helios `rs` FEC mode is treated as `FEC119` later in training. This is an
  assumption.
- The Helios links are idle (almost no traffic), so the data shows little error
  growth.
