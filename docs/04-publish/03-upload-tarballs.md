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

| Status | Meaning |
|---|---|
| 400 | invalid tarball or metas, unknown project, or no upload storage |
| 403 | not an admin |
| 409 | this run was already uploaded; re-import it by URL with `force=true` |
| 413 | larger than the upload limit |

Rejected uploads are also listed among the import events.

## Viewing logs

Nothing can generate log pages next to an object in S3, so Bublik does it
itself. It works the way a TE log server does: the first time a JSON or HTML
page of a run is opened, Bublik generates it from the run's
`raw_log_bundle.tpxz` and writes it back to S3. After that the page is a plain
read. A large run can take a while the first time.

If the storage has a `public_endpoint_url`, pages that already exist are
handed to the browser as presigned URLs on that endpoint, and Bublik serves
none of the bytes. The bucket must then allow `GET` from Bublik's origin
(CORS). The browser fetches these URLs without credentials.

## Deployment settings

| Setting | Default | |
|---|---|---|
| `IMPORT_UPLOAD_MAX_SIZE` | 2 GiB | largest accepted tarball, in bytes |
| `BUBLIK_UPLOAD_MAX_BODY` | `2g` | nginx body limit for the upload endpoint (Docker); keep it in step with the above |
| `LOGS_S3_CACHE_DIR` | `logs-s3-cache` in the Bublik checkout (`/app/bublik/logs-s3-cache` in Docker) | where bundles are cached and pages generated; deleting it only costs regeneration |
