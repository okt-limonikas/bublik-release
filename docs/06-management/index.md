---
title: Management
---

# Management

This section covers the management features available in Bublik for handling test runs, comments, comparisons, and import operations.

## Run Management

### Run Details Page

The run details page provides comprehensive information about a specific test run:

- **Run Information**: Displays metadata, start/finish times, and run status
- **Test Results**: Shows all test iterations with their results, parameters, and verdicts
- **Quick Navigation**: Direct links to specific test results and history
- **Run Statistics**: Summary of passed, failed, and unexpected results

### Run Comments

Comments can be added to runs to provide additional context, notes, or explanations:

1. Navigate to the run details page
2. Locate the comments section
3. Add your comment with relevant information
4. Comments are associated with the run and visible to all users

Comments support per-test granularity, allowing you to comment on specific test results within a run.

### Run Comparison

Bublik supports comparing multiple runs to identify differences in test results:

1. Navigate to the run comparison page
2. Enter the run IDs or run URLs you want to compare
3. The comparison view shows:
   - Test results side by side
   - Highlighted differences
   - Parameter variations between runs

:::tip
The comparison form accepts both run URLs and run IDs for flexibility in specifying runs to compare.
:::

### Run Status

Runs can have various statuses indicating their overall health:

- **OK**: All tests passed as expected
- **Warning**: Some unexpected results within acceptable thresholds
- **Error**: Significant unexpected results exceeding thresholds
- **Compromised**: Run marked as compromised (incomplete or invalid)

Status thresholds are configured via `RUN_STATUS_BY_NOK_BORDERS` in the project configuration.

## Import Events

Import events track the status and history of run imports into Bublik.

### Viewing Import Events

Access the import events page to monitor import operations:

- **Event Status**: Shows whether imports are pending, in progress, completed, or failed
- **Grouped View**: Events are grouped by URL for easier navigation
- **Direct Links**: Quick access to corresponding run details for completed imports

### Import Status Types

- `RECEIVED`: Import request received and queued
- `STARTED`: Import processing has begun
- `COMPLETED`: Import finished successfully
- `FAILED`: Import encountered an error

### Re-importing Runs

Failed imports can be retried using the "Try Again" button on the import events page. This is useful when:

- Network issues caused the initial import to fail
- Log files were temporarily unavailable
- Server issues interrupted the import process

### Import with Project Assignment

When importing runs, you can specify the target project:

1. In the import form, select the project from the dropdown
2. The run will be associated with the selected project
3. Ensure the `PROJECT` meta in `meta_data.json` matches for consistency

## Short URLs

Bublik supports short URLs for easy sharing of run details and test results:

- **Copy Short URL**: Available in the UI for quick sharing
- **With/Without Page State**: Choose whether to include current page filters and settings
- **Persistent Links**: Short URLs remain valid and redirect to the full URL

## Configuration Management

### Updating Configurations

Configurations can be managed through the Admin interface:

1. Navigate to Admin > Configuration Manager
2. Select the configuration type (per_conf, meta, report)
3. Edit or create configurations
4. Validate against the schema before saving

### Schema Viewing

The configuration editor provides schema viewing to help understand available options and their formats. Click the schema button in the admin interface to view the current schema definition.

