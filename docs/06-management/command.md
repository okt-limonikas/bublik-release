---
title: Commands
---

# Management Commands

Bublik provides Django management commands for administrative operations. These commands can be run via the Django manage.py interface or through Docker task commands.

## Import Commands

### importruns

Import test runs from a specified URL:

```bash
# Traditional deployment
python manage.py importruns <url>

# Docker deployment
task import URL=<url>
```

Options:
- `--project`: Specify the target project for the import
- `--force`: Force re-import even if the run already exists

### meta_categorization

Trigger meta categorization to update tag and meta categories based on configuration:

```bash
# Traditional deployment
python manage.py meta_categorization

# Docker deployment
task meta-categorization
```

This command should be run after updating `meta.conf` or `tags.conf` files.

## Run Management Commands

### delete_run

Delete a specific run by ID:

```bash
# Traditional deployment
python manage.py delete_run <run_id>

# Docker deployment
task delete-run RUN_ID=<run_id>
```

:::warning
This operation is irreversible. Make sure to back up your database before deleting runs.
:::

## Cache Commands

### Clear Dashboard Cache

Clear the dashboard cache after configuration changes:

```bash
python manage.py clear_cache --dashboard
```

Alternatively, you can clear the cache through the UI by clicking the clock icon in the upper right corner.

## Database Commands

### migrate

Run database migrations:

```bash
python manage.py migrate
```

This is typically done automatically during deployment.

### createsuperuser

Create an admin superuser:

```bash
python manage.py createsuperuser
```

## Backup Commands (Docker)

For Docker deployments, backup operations are available via task commands:

### Create Backup

```bash
task backup:create
```

### List Backups

```bash
task backup:list
```

### Restore Backup

```bash
task backup:restore BACKUP=<backup_name>
```

## Running Commands in Docker

For Docker deployments, access the Django shell to run management commands:

```bash
# Access the Django container shell
task shell

# Then run management commands
python manage.py <command>
```

Or run commands directly:

```bash
docker exec -it bublik-django python manage.py <command>
```

