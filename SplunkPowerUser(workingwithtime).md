# Splunk Power User Guide

## Working with Time in Splunk

### Time Unit Abbreviations
In Splunk, time is managed with shorthand notations that control the range and precision of searches. Below are key concepts for manipulating time effectively.

### Time Modifier Format
- **earliest**=`[+|-]<timeInt><timeUnit>@<timeUnit>`
- **latest**=`[+|-]<timeInt><timeUnit>@<timeUnit>`

- `<timeInt><timeUnit>`: Represents a time duration as an integer followed by a unit. For example, `3h` indicates 3 hours, `5m` represents 5 minutes, etc.
- `@<timeUnit>`: Snaps the search time to a specified unit, like the start of a day, week, hour, etc.

> **Note:** `@` always rounds down to the beginning of the specified unit.

### Key Principles
- **Snap to Specific Time Units:** Use `@` to align to specific time units.
  - Example: `@w0` for Sunday, `@w1` for Monday, etc.
- **Backward Time Traversal:** Modifiers always look backward in time.
  - Example: `-7d@d` will search the past 7 days, starting from midnight on the first day.

### Time Modifiers: Examples
These examples assume the current time is 9:45 AM.

- **`-30m@h`**: Looks back 30 minutes and snaps to the previous full hour (returns data from 9:00).
- **`earliest=-h@h`**: Rounds down to the beginning of the current hour.
- **`earliest=-7d@d`**: Searches the last 7 days, starting at 00:00 on the first day.
- **`earliest=@d+3h`**: Starts the search at 03:00, snapping to the beginning of the day and adding 3 hours.

### Time Unit Modifiers

Here’s a reference for commonly used time units in Splunk:

- **current** = now
- **second** = `s`, `sec`
- **minute** = `m`, `min`
- **hour** = `h`, `hr`
- **day** = `d`
- **week** = `w`, `week`
- **days of the week**: 
  - `w0` = Sunday
  - `w1` = Monday 
  - `w6` = Saturday
- **month** = `mon`, `month`
- **quarter** = `q`
- **year** = `y`

### Default Time Fields in Splunk

Splunk automatically creates time-based fields for each event that can be used in searches:

- **`date_hour`**: Hour of the event
- **`date_mday`**: Day of the month
- **`date_minute`**: Minute of the event
- **`date_month`**: Month of the event
- **`date_second`**: Second of the event
- **`date_wday`**: Day of the week
- **`date_year`**: Year of the event
- **`date_zone`**: Time zone of the event

If no time is specified, Splunk defaults to using the **indexed time** (the time when the event was indexed).

### Time Binning: Grouping Events by Time
You can use the `bin` command to group events into equal-sized time chunks. For example, using `bin span=1h _time` will group events by 1-hour intervals. This is particularly useful for summarizing data over consistent time frames.

-------------------------------------------------------------------------------------------------
## Formatting Time in Splunk

You can use the `eval` command to reformat time fields or create new time-based fields in your Splunk searches.

### Syntax:
```spl
| eval <field1>=<expression1> [, <field2>=<expression2>]
```

### Examples:
1. **Current Time**:
   ```spl
   | eval field1 = now()
   ```
   - Returns the time the search started (in epoch seconds).

2. **Event Processing Time**:
   ```spl
   | eval field1 = time()
   ```
   - Returns the time the event was processed by the `eval` command.

3. **Relative Time**:
   ```spl
   | eval field1 = relative_time(X, Y)
   ```
   - Returns an epoch time relative to a supplied time.
   - **X** is the starting time in epoch seconds.
   - **Y** is a relative time specifier (e.g., `-1d@d` for 1 day ago at midnight).

   **Example**:
   ```spl
   | eval yesterday = relative_time(now(), "-1d@h")
   ```
   - This creates a field called `yesterday`, representing the start of the previous day.

---

### Time Formatting Functions
Splunk provides the `strftime` and `strptime` functions to format or parse time strings.

- **`strftime`**: Converts epoch time into a formatted string.
- **`strptime`**: Converts a formatted time string into epoch time.

#### Common Formatting Variables:
- **Time:**
  - `%H` → 24-hour format (00–23)
  - `%T` → 24-hour (H:M:S)
  - `%I` → 12-hour format (01–12)
  - `%M` → Minute (00–59)
  - `%p` → AM or PM

- **Days:**
  - `%d` → Day of the month (01–31)
  - `%w` → Day of the week (0–6, Sunday = 0)
  - `%a` → Abbreviated weekday (`Sun`)
  - `%A` → Full weekday name (`Sunday`)
  - `%F` → ISO date format (Year-Month-Day, `%Y-%m-%d`)

- **Months and Years:**
  - `%b` → Abbreviated month name (`Jan`)
  - `%B` → Full month name (`January`)
  - `%m` → Month number (01–12)
  - `%Y` → Year (`2020`)

### Example: Formatting a Date String
```spl
| eval yesterdayString = strftime(yesterday, "%F %H:%M")
```
- Converts the epoch time in the `yesterday` field to a readable string like "2024-09-01 14:30."

## Using Time Commands in Splunk

### `timechart` Command:
The `timechart` command performs statistical aggregations over time, automatically plotting time on the x-axis.

#### Syntax:
```spl
| timechart <stats-function>(<field>) BY <field> [span=<int><timescale>] [limit=<int>]
```

- **Purpose**: Performs statistical aggregations (e.g., `count`, `sum`) and trends data over time.
- **By Clause**: Splits results by another field (optional).
- **Span Option**: Controls the time interval (e.g., `15m`, `1h`, `1d`).
- **Limit Option**: Restricts the number of series plotted (optional).

#### Example:
```spl
index=security sourcetype=linux_secure vendor_action=*
| timechart span=15m count BY vendor_action
```
- This query aggregates event counts over 15-minute intervals, split by the `vendor_action` field.

---

### `timewrap` Command:
The `timewrap` command compares time-based data over specified intervals, such as day-over-day or week-over-week comparisons.

#### Syntax:
```spl
| timewrap [<int>]<timescale>
```

- **Purpose**: Allows comparison of data over time periods like week-over-week or month-over-month.
- **Timescale**: Defines the period to compare (e.g., `1d`, `1w`).

#### Example:
```spl
index=security sourcetype=linux_secure "failed password" earliest=-14d@d latest=@d
| timechart span=1d count AS Failures
| timewrap 1w
```
- This query counts failed password attempts daily and compares the counts week-over-week.

### Timezone Adjustment Example
index=sales sourcetype=vendor_sales earliest=-2d@d latest=@d
| eval my_hour = strftime(_time,"%H")
| search my_hour>=2 AND my_hour<5
| bin span=1h _time
| stats sum(price) as "Hourly Sales"" by _time
| eval Hour=strftime(_time, "%b %d, %I %p")
table Hour, "Hourly Sales"

### Timezone Example

This query analyzes vendor sales data, filtering events based on specific hours of the day and calculating hourly sales totals within the specified time range.

#### Explanation of the Query:

1. **Search for Sales Data**
   ```
   index=sales sourcetype=vendor_sales earliest=-2d@d latest=@d
   ```
   - Searches within the `sales` index and the `vendor_sales` sourcetype.
   - Limits the results to the last two days (`earliest=-2d@d`) up until the end of the current day (`latest=@d`).

2. **Extracting the Hour from the Event Time**
   ```
   | eval my_hour = strftime(_time, "%H")
   ```
   - Uses the `strftime` function to extract the hour from the event's timestamp (`_time`) in 24-hour format (`%H`).
   - Stores the result in a new field called `my_hour`.

3. **Filter Events Based on Specific Hours**
   ```
   | search my_hour>=2 AND my_hour<5
   ```
   - Filters the events to only include those that occurred between 2 AM and 5 AM.

4. **Binning Events by Hour**
   ```
   | bin span=1h _time
   ```
   - Groups the events into 1-hour intervals based on their timestamp (`_time`).

5. **Calculating Hourly Sales Totals**
   ```
   | stats sum(price) as "Hourly Sales" by _time
   ```
   - Calculates the total sales (`sum(price)`) for each hour and assigns it to the field `"Hourly Sales"`.
   - Groups the results by the `_time` field.

6. **Formatting the Time for Display**
   ```
   | eval Hour = strftime(_time, "%b %d, %I %p")
   ```
   - Converts the `_time` field into a more readable format: "Month Day, Hour AM/PM" (e.g., `Sep 01, 03 PM`).
   - Saves the formatted time as `Hour`.

7. **Final Display**
   ```
   | table Hour, "Hourly Sales"
   ```
   - Displays the `Hour` and `Hourly Sales` fields in a table format for easy viewing.
