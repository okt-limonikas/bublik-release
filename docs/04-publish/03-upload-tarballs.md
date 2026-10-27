---
title: Upload Log Tarballs
---

# Upload Log Tarballs

Test Environment's `publish_logs` packs a finished run into `te_logs.tar`: a
flat tar with `meta_data.json` and `raw_log_bundle.tpxz` (optionally
`trc-brief.html` and `trc-stats.txt`). Instead of publishing that tar on a TE
log server and importing the resulting URL, you can hand it straight to
Bublik. Bublik stores it in S3-compatible object storage and imports it.

Uploads are available to admins, from the **Import** dialog or over the API.

## 1. Configure upload storage for the project

A project accepts uploads only once its **references** config has a logs base
stored in S3 and marked `"upload": true`. Without one, uploads to the project
are refused.

```json
{
  "LOGS_BASES": [
    {"name": "TE log server", "uri": ["https://logs.example.com/logs/"]},
    {
      "name": "Uploads",
      "uri": ["s3://bublik-logs/uploads/"],
      "storage": {"type": "s3", "upload": true}
    }
  ]
}
```

The `uri` names the bucket and key prefix runs are stored under. Every
`storage` field except `type` is optional and falls back to the matching
`S3_*` Django setting. In the Docker deployment those point at the bundled
SeaweedFS, so the entry above is all it needs.

To use another bucket or endpoint, say how to reach it:

```json
"storage": {
  "type": "s3",
  "endpoint_url": "https://s3.eu-west-1.amazonaws.com",
  "public_endpoint_url": "https://s3.eu-west-1.amazonaws.com",
  "region": "eu-west-1",
  "access_key": "${env:LOGS_S3_ACCESS_KEY}",
  "secret_key": "${env:LOGS_S3_SECRET_KEY}",
  "upload": true
}
```

:::warning
Configs can be read through the API, so credentials are **never** written into
them. `access_key` and `secret_key` are references to a variable of Bublik's
environment (`${env:NAME}`) or Django settings (`${settings:NAME}`), and the
name must start with `LOGS_`. A plain key is rejected when the config is
saved.
:::

| Field | Meaning |
|---|---|
| `endpoint_url` | S3 endpoint Bublik uses |
| `public_endpoint_url` | S3 endpoint browsers can reach; see [Viewing logs](#viewing-logs) |
| `region` | bucket region |
| `access_key`, `secret_key` | `LOGS_` secret references |
| `upload` | uploads to this project go to this base |

Runs already in S3 can also be imported without uploading, by URL:
`/api/v2/importruns/source/?url=s3://bublik-logs/uploads/<path>/`. A prefix
holding `meta_data.json` is one run. Any other prefix is searched for runs
below it.

## 2. Upload

### From the UI

Open **Import**, switch to **Upload .tar**, pick the project and drop one or
more tarballs. If you keep the project as "From meta_data.json", the
tarball's `PROJECT` meta picks it. Each file shows its progress and its own
error. The imports appear in the import events table like URL imports.

### From CI

Create a personal access token of an admin in **Settings → Access tokens**.
A dedicated user such as `ci-uploader` works well. Then:

```bash
curl -H "Authorization: Bearer ${BUBLIK_TOKEN}" \
     -F file=@te_logs.tar -F project=<project-id> \
     "${BUBLIK_URL}/api/v2/importruns/upload/"
```

| Field | |
|---|---|
| `file` | the tarball (required) |
| `project` | project id; defaults to the tarball's `PROJECT` meta |
| `force` | `true` to re-import a run that already exists |

The response is the same as for a URL import, plus the run's `url`:

```json
{"job_id": 42, "job_tasks_data": [...], "url": "s3://bublik-logs/uploads/sockapi-ts/2026/09/23/bench1-10:11:12/"}
```

With the Docker deployment, `task logs:upload -- te_logs.tar <project-id>`
does the same using `BUBLIK_TOKEN`.

### What is checked

The tarball gets the same checks a TE log server applies
(`publish-logs-unpack.sh`):

- it is a plain (uncompressed) tar of regular files at the top level;
- it holds only the files listed above, including `meta_data.json` and
  `raw_log_bundle.tpxz`;
- `meta_data.json` has `version` 1 and the metas `CAMPAIGN_DATE`, `TS_NAME`,
  `CFG` and `START_TIMESTAMP` (`USER` is optional).

The run is stored with the log server's layout,
`<prefix>/TS_NAME/YYYY/MM/DD/CFG[-USER]-HH:MM:SS/`.

### Bundles with ready-made log pages

A run made outside TE, such as a Playwright report converted by
`bublik-e2e playwright`, can be uploaded the same way. Its tar holds
`bublik.json` instead of `raw_log_bundle.tpxz`, so it has these files:

- `meta_data.json` and `bublik.json` at the top level, with an optional `.done`;
- its JSON log pages under `json/`;
- its attachments under `attachments/` (`attachments/node_id<N>/attachments.json`
  and the files it names).

Files may be nested only under `json/` and `attachments/`, as plain relative
paths. Directory entries are allowed and links are not. `TS_NAME`, `CFG` and
`USER` may be up to 64 characters, not 32. Everything is stored as it is.
Bublik serves the pages and attachments straight from S3 and has nothing
to generate or cache.

```bash
tar -C bundle -cf bundle.tar meta_data.json bublik.json .done json attachments
curl -H "Authorization: Bearer ${BUBLIK_TOKEN}" -F file=@bundle.tar \
     "${BUBLIK_URL}/api/v2/importruns/upload/"
```

`bublik-e2e playwright --upload --url "$BUBLIK_URL" --token "$BUBLIK_TOKEN"`
writes the bundle and uploads it in one step.

| Status | Meaning |
|---|---|
| 400 | invalid tarball or metas, unknown project, or no upload storage |
| 403 | not an admin |
| 409 | this run was already uploaded; re-import it by URL with `force=true` |
| 413 | larger than the upload limit |

Rejected uploads are also listed among the import events.

## Viewing logs

Nothing can generate log pages next to an object in S3, so Bublik does it
itself, the way a TE log server does. The first time a JSON or HTML page of a
TE run is opened, Bublik downloads the run's `raw_log_bundle.tpxz` into a local
cache, generates the page there and serves it. Other pages of the run are then
generated from the same downloaded bundle. A large run can take a while the
first time.

Opening a run's `json/` or `html/` folder in the source listing generates the
whole folder and lists it.

### What is stored where

| | Where | For how long |
|---|---|---|
| What was uploaded: `meta_data.json` and `raw_log_bundle.tpxz`, or a bundle's `bublik.json`, `json/` and `attachments/` | S3 | Kept. Deleting the run in Bublik does not delete them. |
| The downloaded bundle and the pages generated from it | `LOGS_S3_CACHE_DIR` on the Bublik host | Until the run goes unused for `LOGS_S3_CACHE_TTL` seconds (1 hour by default) |

Generated pages are never written to S3, so the bucket holds only the
original files. The cache cleans itself up: requests sweep it at most every
5 minutes and delete the folders of runs nobody opened for
`LOGS_S3_CACHE_TTL` seconds. A run that is opened again is downloaded and
generated anew.

To sweep from cron instead, or to clear the cache now:

```bash
python manage.py prune_logs_cache            # folders unused for LOGS_S3_CACHE_TTL
python manage.py prune_logs_cache --ttl 0    # every folder not in use right now
```

Bundles with ready-made log pages (see above) never touch the cache: their
pages and attachments are served straight from S3.

If the storage has a `public_endpoint_url`, files that exist in S3 (uploaded
pages and attachments) are handed to the browser as presigned URLs on that
endpoint, and Bublik serves none of the bytes. The bucket must then allow
`GET` from Bublik's origin (CORS). The browser fetches these URLs without
credentials. Generated pages always go through Bublik.

## Deployment settings

| Setting | Default | |
|---|---|---|
| `IMPORT_UPLOAD_MAX_SIZE` | 2 GiB | largest accepted tarball, in bytes |
| `BUBLIK_UPLOAD_MAX_BODY` | `2g` | nginx body limit for the upload endpoint (Docker); keep it in step with the above |
| `LOGS_S3_CACHE_DIR` | `logs-s3-cache` in the Bublik checkout (`/app/bublik/logs-s3-cache` in Docker) | where bundles are cached and pages generated; deleting it only costs regeneration |
| `LOGS_S3_CACHE_TTL` | `3600` | seconds a run's cache folder outlives its last use |
