# klp

_Kool Log Parser_

> **⚠️ This project is no longer actively maintained.**
>
> Development has moved to [**Kelora**](https://github.com/dloss/kelora), a complete rewrite in Rust with improved performance and features. Visit [kelora.dev](https://kelora.dev) to learn more.

klp is a lightweight, command-line interface (CLI) tool for analyzing and understanding structured logs in various formats.
Designed for software developers involved in debugging and troubleshooting.

Supports [logfmt](https://brandur.org/logfmt), [JSON Lines](https://jsonlines.org),
[CSV](https://en.wikipedia.org/wiki/Comma-separated_values), [TSV](https://en.wikipedia.org/wiki/Tab-separated_values), [SQLite](https://sqlite.org) and many other formats.

Single file Python script without dependencies apart from Python 3.7+ and its standard library. 

![](demo/basic.png)

## Table of Contents

- [Key Features](#key-features)
- [Getting Started](#getting-started)
  - [Installation](#installation)
  - [Quick Start](#quick-start)
  - [Basic Usage](#basic-usage)
- [Core Features](#core-features)
  - [Input Handling](#input-handling)
  - [Error Handling](#error-handling)
  - [Event Creation](#event-creation)
  - [Overview and Statistics](#overview-and-statistics)
  - [Pattern Normalization and Deduplication](#pattern-normalization-and-deduplication)
  - [Time Management](#time-management)
  - [Search and Filter](#search-and-filter)
  - [Output Control](#output-control)
  - [Configuration File](#configuration-file)
- [Advanced Usage](#advanced-usage)
  - [Complex Filtering with Python Expressions](#complex-filtering-with-python-expressions)
  - [Advanced Input Transformations Using Python Code](#advanced-input-transformations-using-python-code)
  - [Custom Output Formatting using Python](#custom-output-formatting-using-python)
  - [Parsing Non-Line-Based Formats](#parsing-non-line-based-formats)
- [Additional Resources](#additional-resources)
  - [Complementary Tools](#complementary-tools)
  - [Alternative Tools](#alternative-tools)

## Key Features

- **Focus on Essentials**: 
Instantly highlight key fields: timestamp, log level, and message.
- **Advanced Time Analysis**:
Track time gaps in logs, fuse bursts of events, and filter by complex time ranges.
- **Smart Filtering**:
Filter by log level, time spans, or using Python expressions. 
- **Pattern Search**:
Grep in specific fields and show context lines. 
Use a wide range of built-in patterns to search or extract URLs, IPs, paths, email addresses, errors, and more. 
- **Visual Insights**:
Get statistics about the number of events and the occuring log levels, visually mark time gaps between events and see a succinct map of log level patterns.
- **Enhanced Readability**:
Color-coding, per-key display, expandable newlines and custom output templates using Python expressions. 
- **Format Conversion**:
Convert between common formats like logfmt, JSON, JSONL, CSV, TSV and SQLite.
- **Powerful Log Transformation**:  
Transform unstructured logs into structured data using Python code and convenient helper methods.
Create simple ad-hoc parsers for your own custom formats, add proper field names and save the result in any of the supported output formats.

## Getting Started

### Installation

Install using pip:

```bash
$ pip install klp-logviewer
```

Or just download `klp.py` and run it using `python3`, without any installation.

### Quick Start

Here are some basic examples to get you started with klp:

```bash
# View logs in logfmt format
$ klp mylog.logfmt

# Specify other input formats:
$ klp -f jsonl applog.jsonl
$ klp -f csv mydata.csv
$ klp -f line text.txt

# Only show specific keys
$ klp -k timestamp,level,message mylog.logfmt

# Don't print the keys, only the values
$ klp -p mylog.logfmt

# Filter logs by log level:
$ klp -l error,warning mylog.logfmt

# Get an overview of your log data:
$ klp -S mylog.logfmt

# View only new log entries (useful with streaming logs):
$ tail -f mylog.logfmt | klp -n
```

> [!NOTE]
> klp defaults to the [logfmt](https://brandur.org/logfmt) input format and will extract any `key=value` patterns it finds, 
> even if the file is in a different format. 
> For non-logfmt logs, specify the format explicitly (e.g. `-f syslog`, or `-f line` for plain text).


### Basic Usage

klp can be used as a filter, reading from stdin:

```bash
$ tail -f log.txt | klp
```

Or you can read from a file:

```bash
$ klp log.txt
```

klp can directly process compressed log files:

```bash
$ klp app.log.gz
$ klp logs.zip
```

Options can be given before or after the filename.
See `klp --help` for the list of available options.


#### Basic Filtering

**Filter by Log Level:**
Use `--loglevels`/`-l` to restrict the output to specific log levels:

```bash
$ cat mylog.txt | klp -l warning,error,fatal
```

Exclude log levels with `--not-loglevel`/`-L`:
```bash
$ cat mylog.txt | klp -L trace,debug
```

**Filter by Time:**
klp tries to parse timestamps automatically. 
The following keys are treated as possibly holding timestamps: `timestamp`, `ts`, `time`, `at`.

```bash
# Show events from the last hour
$ klp --since 1h app.log

# Show events from both a start and end time
$ klp --from "2024-02-05T20:18:10Z" --to "2024-02-05T20:19:20Z" app.log
```

**Filter by Regex:**

Use `--grep`/`-g` for simple text search with regular expressions:

```bash
# Search for specific text
$ klp -g "error" app.log

# Case-insensitive search
$ klp -g "error" -i app.log

# Search in specific fields
$ klp -g "message~error" app.log
```

## Core Features

### Input Handling

#### Supported input formats

klp supports a wide range of input formats:

- `logfmt`: Key-value pairs (**default**)
- `jsonl`: JSON Lines (shortcut: `-j`)
- `json`: JSON (only for complete files, not for streaming)
- `csv`: Comma-separated values 
- `tsv`: Tab separated values 
- `psv`: Pipe separated values 
- `line`: lines of text (trailing whitespace removed)
- `tsNm`: logs containing a timestamp (with N components) and a message. See below for examples. 
- `tsNlm`: logs containing a timestamp (with N components), a log level and a message
- `log4j`: Apache Log4j format
- `cef`: Common Event Format (CEF)
- `unix`: Common format of Unix server logs (timestamp, hostname, service, optional pid, message)
- `rfc5424`: Syslog as specified in RFC5424 (including a priority)
- `syslog`: Syslog (either RFC 5424 or the classical Unix one without priority)
- `clf`: NCSA Common Log Format
- `combined`: Combined Log Format of Apache httpd
- `sqlite`: SQLite database (use `--input-tablename` if more than one table is present)
- `data`: Parse everything as one string 

Use the `--input-format` or `-f` option to specify the input format. For example:

```bash
$ klp -f jsonl input.log
```

The `tsNm` formats allows parsing logs that consist of a timestamp, and a message. 
`N` indicates how many whitespace-separated components the timestamp consists of.
Use `tsNlm` for logs that contain a log level between the timestamp and the message.
Here are some examples:

```bash
# Log format: <timestamp> <message>
# Example:    2024-02-05T20:18:10.538Z Connection established to database
$ klp -f ts1m server.log

# Log format: <date> <time> <level> <message>
# Example:    2024-02-05 20:18:10 INFO Starting background worker
$ klp -f ts2lm worker.log

# Log format: <date> <time> <timezone> <message>
# Example:    2024-02-05 20:18:10 UTC Processing batch job #1234
$ klp -f ts3m batch.log
```

Similar formats exist up to `ts5m`/`ts5lm` for logs with more timestamp components.
The timestamp parts are joined and parsed using klp's standard timestamp parsing logic.

More complicated formats can often be parsed using the `line` format and creating or transforming events using Python code (`--input-exec`).
See the [Advanced input transformations using Python code](#advanced-input-transformations-using-python-code) section below.
Alternatively, you can check out [jc](https://github.com/kellyjonbrazil/jc), a tool that converts numerous formats to JSON, which can be read by klp `-f json`.

#### Input Options

klp provides several options to control how input data is read and parsed:

- `--input-encoding`: Specify the text encoding of the input data (default: utf-8)
  ```bash
  # Read a file with Latin-1 encoding
  $ klp --input-encoding latin1 legacy.logfmt
  
  # Process Windows-1252 encoded logs
  $ klp -f line --input-encoding cp1252 windows.log
  ```

- `--no-header`: For CSV, TSV, or PSV formats, indicates that the file has no header row.
  This will auto-generate numbered column keys.
  ```bash
  # Process a headerless CSV file (columns will be named col0, col1, etc.)
  $ klp -f csv --no-header data.csv
  ```

- `--input-delimiter`: Specify the delimiter for CSV, TSV, or PSV formats
  ```bash
  # Process a file with semicolon-separated values
  $ klp -f csv --input-delimiter ";" european.csv
  
  # Read a custom-delimited file
  $ klp -f csv --input-delimiter ":" custom.log
  ```

- `--skip`: Avoid parsing the first N lines (e.g. to skip comments or old data).
  ```bash
  # Skip the first 5 lines of a CSV file
  $ klp -f csv --skip 5 data.csv
  ```

- `--logical-lines`: Handle log entries that span multiple physical lines by combining related lines into a single event. Two methods are supported:
 
  1. *Indentation Continuation*: Lines starting with whitespace are joined with a space between them (indentation removed)

  ```bash
  $ cat multi_indented.log
  Starting SQL query:
      SELECT * FROM users
      WHERE active = true
  Query completed

  $ klp -f line --logical-lines multi_indented.log
  line="Starting SQL query: SELECT * FROM users WHERE active = true"
  line="Query completed"
  ```
  2. *Backslash Continuation*: Lines ending with `\` are concatenated with the next line (backslash removed)

  ```bash
  $ cat multi_backslash.log
  Dec 16 10:15:30 shade app[123]: Starting a long process \
  with multiple arguments and lots of long configuration o\
  ptions
  Dec 16 10:15:31 shade app[123]: Process completed

  $ klp -f unix --logical-lines multi_backslash.log
  timestamp="Dec 16 10:15:30" hostname=shade service=app pid=123 message="Starting a long process with multiple arguments and lots of long configuration options"
  timestamp="Dec 16 10:15:31" hostname=shade service=app pid=123 message="Process completed"
  ```

  The `--logical-lines` option can be combined with any line-based input format and works with streaming input.
  It processes the lines before they are parsed according to the input format, so it works transparently with all klp's features.

#### Error Handling

Control how klp handles errors with the `--errors` option:

- `ignore` (default): Silently continue processing when errors occur. With `--debug`, errors will still be printed.
- `print`: Print error messages to stderr but continue processing
- `exit`: Print error messages to stderr and exit with status code 1

```bash
# Print parsing errors but continue processing
$ klp --errors print malformed.log

# Stop processing at first error
$ klp --errors exit app.log

# Process as much as possible, ignoring errors (default)
$ klp --errors ignore mixed.log

# Show errors even in ignore mode
$ klp --errors ignore --debug mixed.log
```

### Event Creation

klp parses each line of the input file (or stdin stream) into an *event*.
If a line cannot be parsed, it is ignored silently.
An event can have several *fields* (key/value pairs).
The events in a file or stream don't have to have the same fields.

klp tries to automatically detect timestamps, log levels and log messages
by looking for the following keys:

* Timestamp keys: `ts`, `time`, `timestamp`, `t`, `at`, `_t`, `@t`
* Log level keys: `log_level`, `loglevel`, `level`, `lvl`, `loglevel`, `severity`, `levelname`, `@l`
* Message keys: `message`, `msg`, `@m` 

If your timestamp or log level fields aren't in these defaults, use `--ts-key KEY` and `--loglevel-key KEY` to specify them.
klp has special features to select events by timestamp or log level (see below).


### Overview and Statistics

#### Stats View
Use `--stats-only`/`-S` to get some basic statistics about number of events in the log file,
the timespan, the names of the keys and the log levels:

```bash
$ klp -S mylog.logfmt
Events shown: 8043 (100% of 8043 lines seen)
Time span shown: 2024-02-05T19:58:59.874Z to 2024-02-05T20:19:20.575Z (0:20:20.701000)
Keys seen: timestamp,log_level,source,target,thread_id,message
Log levels seen: DEBUG,TRACE,INFO (keys: log_level)
```

Or use `--stats`/`-s` to print those stats (to stderr), in addition to regular output.

#### Level Map
Use `--levelmap`/`-M` to print only the first character of the log level of each event.
The timestamp is for the first event shown in that line:

```bash
$ klp -M mylog.logfmt
2024-02-05T20:18:10.538Z TTTTDTTTDITITTTTTTTTTTTTTTTDTTTDITITTTTTTTTTTTTTTTDTTTDITITTTTTTTTTTTTTTTDTTTDIT
2024-02-05T20:18:30.546Z ITTTTTTTTTTTTTTTDDTTITITTTTTTTTTTTTTTITITTTTTTTTTTTTTTTTITITTTTTTTTTTTTTTTTITITT
```

#### Key Map
Visualize patterns for any key:

```bash
$ klp --keymap status_code app.log
```
### Pattern Normalization and Deduplication

Use `--normalize` to replace common patterns (like IP addresses, emails, timestamps) with placeholders and only show unique event patterns. This helps identify distinct message types by ignoring variable data.

```bash
# Original logs
$ klp app.log
timestamp="2024-02-14T12:00:01Z" message="Failed login from 192.168.1.100 for user@example.com"
timestamp="2024-02-14T12:00:05Z" message="Connection from 10.0.0.50 to https://api.example.com"
timestamp="2024-02-14T12:00:10Z" message="Failed login from 192.168.1.101 for admin@example.com"
timestamp="2024-02-14T12:00:15Z" message="Connection from 10.0.0.51 to https://api.example.com"

# With normalization - shows unique patterns only
$ klp --normalize app.log
timestamp="<timestamp>" message="Failed login from <ipv4> for <email>"
timestamp="<timestamp>" message="Connection from <ipv4> to <url>"
```

The following patterns are detected and normalized:
- Timestamps → `<timestamp>` (only in timestamp fields)
- Email addresses → `<email>`
- IPv4/IPv6 addresses → `<ipv4>`, `<ipv6>`
- URLs → `<url>` (requires protocol prefix like http://)
- Domain names → `<fqdn>`
- MAC addresses → `<mac>` (colon or dot separated)
- Unix path names → `<path>`
- Hash values:
  - MD5 → `<md5>` (32 hex chars)
  - SHA1 → `<sha1>` (40 hex chars)
  - SHA256 → `<sha256>` (64 hex chars)
- UUIDs → `<uuid>`
- Version strings → `<version>` (starting with v/V)
- Function calls → `<function>` (name with parentheses)
- Hex color codes → `<hexcolor>` (6 digits with # prefix)
- OAuth tokens → `<oauth>`

The normalization considers only the fields that would be displayed (affected by `--keys`/`-k` and `--keys-not`/`-K`). 
Timestamps are only normalized in fields recognized as timestamp fields (configurable with `--ts-key`).

```bash
# Only consider message patterns, ignore timestamp differences
$ klp --normalize -k message app.log
message="Failed login from <ipv4> for <email>"
message="Connection from <ipv4> to <url>"

# Focus on specific event properties
$ klp --normalize -k level,component,message app.log
level=error component=auth message="Invalid credentials for <email>"
level=warn component=network message="Timeout connecting to <url>"
```

This feature is particularly useful for:
- Identifying distinct log patterns in large files
- Finding unique error types by normalizing variable data
- Analyzing log structure and message templates
- Reducing noise from repetitive events with different parameters

### Time Management

#### Time Filtering

klp tries to parse timestamps automatically.
The following keys are treated as possibly holding timestamps: `timestamp`, `ts`, `time`, `at`.
You can specify your own key with `--ts-key KEY`.

klp includes a list of common time formats that will be tried for parsing.
Use `--ts-format` to specify your own format, using [Python strptime() format codes](https://docs.python.org/3/library/datetime.html#format-codes) (e.g. `%a %d %b %Y, %I:%M%p` to parse `Mon 31 Jan 2022, 11:59PM`).
A short reference of the format codes can be printed with `klp --help-time`.

There are multiple ways to specify the time range that you're interested in:

* absolute time: `--from ISODATETIME` and `--to ISODATETIME` 
* relative from now: `--since INTERVAL` and `--until INTERVAL` 
* timespan, computed from program start: `--duration INTERVAL`
* timespan, computed from first event shown: , `--timespan INTERVAL`

Timespans can be specified with these units (for example: `--since 1w2h3.2s`):

- `w` = weeks
- `d` = days
- `h` = hours
- `m` = minutes
- `s` = seconds
- `ms` = milliseconds
- `us` = microseconds

#### Timezone Control
By default, klp preserves timestamps as they appear in the logs.
Use the following options to control how timestamps are displayed:

* `--localtime`/`-z`: Convert timestamps to your local timezone
* `--utc`/`-Z`: Convert timestamps to UTC

```bash
# Show timestamps in local timezone
$ klp --localtime -c mylog.logfmt
2015-03-26T06:27:38.000+01:00 debug Started observing beach

# Show timestamps in UTC
$ klp --utc -c mylog.logfmt
2015-03-26T05:27:38.000Z debug Started observing beach
```

Note that timezone conversion can slightly impact performance when processing large log files.
Only use these options when timezone conversion is specifically needed.

These options cannot be used together - choose either local time or UTC.
They work with all input formats and can be combined with other timestamp-related features like `--since`, `--until`, and `--timespan`.


#### Time Gaps
Use `--mark-gaps` to visually separate events that are far apart in time:

```bash
$ klp prometheus.logfmt -c --mark-gaps 1h
2024-02-05T11:00:02.081Z info Deleting obsolete block
_______________________________ time gap: 1:59:59.984000 ________________________________
2024-02-05T13:00:02.065Z info write block
2024-02-05T13:00:02.075Z info Head GC completed
2024-02-05T13:00:02.076Z info Creating checkpoint
2024-02-05T13:00:02.235Z info WAL checkpoint complete
_______________________________ time gap: 1:59:59.666000 ________________________________
2024-02-05T15:00:01.901Z info write block
2024-02-05T15:00:01.905Z info Head GC completed
_______________________________ time gap: 1:59:59.984000 ________________________________
2024-02-05T17:00:01.889Z info write block
2024-02-05T17:00:01.893Z info Head GC completed
```

#### Event Condensing
The `--fuse` option allows you to condense events that occur close together in time.

```bash
$ klp -c --fuse 1h prometheus.logfmt
1 2024-02-05T15:00:01.901Z info write block
3 2024-02-05T15:00:01.905Z info Head GC completed
_______________________________ time gap: 1:59:59.984000 ________________________________
1 2024-02-05T17:00:01.889Z info write block
9 2024-02-05T17:00:02.111Z info Deleting obsolete block
_______________________________ time gap: 1:59:59.782000 ________________________________
1 2024-02-05T19:00:01.893Z info write block
3 2024-02-05T19:00:01.897Z info Head GC completed
```

This shows only the first and last events for each group of events that occur within 1 hour of each other.
From the event number in the first column you can see how many events were left out.

### Search and Filter

#### Grep Capabilities
Use `--grep`/`-g` to limit the processing to lines that match a given regular expression.
When this flag is given multiple times, any of those regexes matching 
will allow the line to be processed (logical OR).

```bash
# Search with multiple patterns (OR)
$ klp -g "error" -g "warning" app.log

# Search in specific fields
$ klp -g "message~error" app.log
```

To specify lines that should NOT be processed, use `--grep-not`/`-G`/`-v`.

Search is case-sensitive by default. 
Use `--ignore-case`/`-i` for case-insensitive matches.

To output only specific parts of the match, see the [extracting capture groups](#extract-capture-groups) section 