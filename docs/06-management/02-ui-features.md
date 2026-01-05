---
title: UI Features
---

# UI Features

This document covers the user interface features available in Bublik for browsing, analyzing, and interacting with test results.

## History and Search

### History Page

The history page provides a comprehensive view of test results across multiple runs:

- **Linear Mode**: Shows individual test iterations chronologically
- **Aggregation Mode**: Groups similar results for trend analysis
- **Filter Persistence**: Filter settings are preserved in the URL for sharing

### Date Range Picker

The date range picker allows you to filter results by time period:

- **Preset Ranges**: Quick selection of common time periods (last 7 days, last 30 days, etc.)
- **Duration-Based Selection**: Specify a duration relative to today
- **Sliding Window**: Use range buttons to shift the time window forward or backward
- **Custom Range**: Select specific start and end dates

### Search Filters

Multiple filter types are available for narrowing down results:

- **Tags**: Filter by test tags
- **Parameters**: Filter by test argument values
- **Requirements**: Filter by test requirements
- **Labels**: Filter by metadata labels
- **Revisions**: Filter by source code revisions
- **Branches**: Filter by branch names
- **Verdicts**: Filter by test verdicts

Filters support both inclusive and exclusive modes, and can use expression syntax for complex queries.

### Direct History Links

Navigate directly to history filtered by specific criteria:

- **Run + Test Name**: Link to history showing a specific test across runs
- **Test Result**: Direct link to a specific test result iteration
- **Split-Button Navigation**: Quick access to different history views from result cards

## Run Navigation

### Run Details

The run details page shows comprehensive information about a test run:

- **Metadata Display**: All run metadata visible at a glance
- **Result Summary**: Quick statistics on passed/failed tests
- **Navigation Shortcuts**: Quick links to specific test results
- **Comments Section**: View and add run comments

### Quick Navigation

Several shortcuts are available for efficient navigation:

- **Go to Result**: Navigate directly to a specific result within a run
- **History Shortcut**: Jump to history filtered by the current test
- **Split Buttons**: Multiple navigation options in a single control

## Charts and Measurements

### Series Charts

The series charts mode provides visualization of measurement data:

- **History Charts**: View measurement trends over time
- **Per-Result Charts**: See serial measurements within a single test result
- **Chart Selection**: Select and combine multiple measurements
- **Stacking**: Overlay multiple charts for comparison

### Measurement Features

- **Selection Popover**: Choose which measurements to display
- **Chart Toolbar**: Access chart controls and options
- **Y-Axis Limits**: Toggle between auto-scaling and data-bound limits
- **Canvas Rendering**: Improved performance for large datasets

### Report Charts

In reports, additional chart features are available:

- **Multi-Argument Overlays**: Compare data across different argument values
- **Stacked Charts Mode**: Combine multiple charts in a unified display
- **Table of Contents**: Navigate report sections easily

## Results Display

### Result Cards

Test results are displayed in cards with:

- **Status Indicators**: Clear visual indication of pass/fail
- **Parameter Display**: Key test parameters shown
- **Important Tags**: Highlighted tags on hover
- **Action Buttons**: Quick access to logs, history, and details

### Multiple Expected Results

Tests can have multiple expected results:

- **All Expected Results Shown**: View all valid expected outcomes
- **Highlight Current**: The actual result is clearly indicated
- **Filter by Expected**: Filter history by specific expected results

### Parameters Diff Mode

When comparing results, diff mode highlights parameter differences:

- **Changed Values**: Parameters that differ are highlighted
- **Common Values**: Shared parameters shown normally
- **Quick Toggle**: Easy switch between diff and normal views

## Log Viewing

### Log Display

Logs are displayed with several viewing options:

- **Level Expansion**: Expand/collapse log levels (MI level shown by default)
- **Source Navigation**: Jump to source code when URLs are available
- **Verdict Filtering**: Filter log entries by verdict
- **Scenario Filtering**: Filter by test scenario

### Artifacts

Test artifacts are accessible through the UI:

- **Dropdown Menu**: Access artifacts from the result card
- **Multiple Types**: Support for text and packet capture files
- **Inline Viewing**: Text artifacts can be viewed directly in the browser

## Network Packet Analysis

For tests with packet captures:

- **Packet Capture Page**: Dedicated view for .cap and .pcap files
- **Dissection Tree**: Hierarchical view of packet structure
- **Trace Flow**: Visualize network traffic flow
- **Resizable Panels**: Customize the view layout

## Sidebar Navigation

### Project Picker

The sidebar includes a project picker for multi-project navigation:

- **Quick Switching**: Change projects without leaving the current page
- **Project Filtering**: All pages respect the selected project
- **Animation**: Smooth accordion animation for project list

### Settings

The settings dropdown provides access to:

- **Display Options**: Customize how results are shown
- **Cache Control**: Clear caches when needed
- **User Preferences**: Save personal settings

## URL Management

### Short URLs

Bublik supports short URLs for easy sharing:

- **Copy Short URL**: Available from the toolbar
- **With Page State**: Include current filters and view settings
- **Without Page State**: Share clean URLs without temporary state

### URL Persistence

Many UI states are preserved in the URL:

- **Filter Settings**: Search filters are URL-encoded
- **View Mode**: Current view mode is preserved
- **Navigation State**: Open/close states for expandable sections

