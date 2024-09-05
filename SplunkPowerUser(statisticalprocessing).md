## Statistical Processing Overview

### `| chart` Command

Syntax:
```
| chart <stats-func>(<wc-field>) over <row-split> [by <column-split>] [span=<int><timescale>] [limit=<int>] [useother=<bool>] [usenull=<bool>]
```
- **Visualizes Data in Table Format:** Generates results in a table that can be visualized.
- **Y-axis:** Specify using `<stats-func>(<wc-field>)`, where:
  - `<wc-field>` is the numeric field to aggregate, and wildcards are supported.
  - `<stats-func>` is the statistical function to apply (e.g., `count`, `sum`).
- **X-axis:** Define with `over <row-split>`.
- **Splitting Data:** Use the `by <column-split>` clause to create a multi-series dataset.
- **Options:**
  - `span`: Adjusts the granularity of time intervals.
  - `limit`: Limits the number of distinct values.
  - `useother`/`usenull`: Controls behavior of null or "other" values.

**Example:**
```
index=sales sourcetype=vendor_sales VendorID<4000
| chart count over VendorCountry by product_name
```

---

### `| timechart` Command

Syntax:
```
| timechart <stats-func>(<field>) by <split-by-field> [span=<int><timescale>] [limit=<int>]
```
- **Time-based Aggregations:** Performs statistical calculations over time, with `_time` as the X-axis.
- **Y-axis:** Populated by the result of `<stats-func>(<field>)`. If using `count`, no field specification is required.
- **Visualization:** Similar to `chart`, but explicitly designed for time-based data.

**Example:**
```
index=network sourcetype=cisco_wsa_squid
| timechart count by usage
```

#### `timechart` Command: `span` Option
- **Bucket Data by Time:** Time is divided into buckets depending on the time range. Without specifying `span`, it defaults based on the search range:
  - Last 60 minutes defaults to `span=1m`
  - Last 24 hours defaults to `span=30m`

**Example:**
```
index=security sourcetype=linux_secure vendor_action=*
| timechart span=15m count by vendor_action
```

#### `timechart` Command: `limit` Option
- Controls the number of distinct values returned by the `by` clause field.

---

### `| top` Command

Syntax:
```
| top (<field>|<field-list>) [by <field>] [countfield=<string>] [limit=<int>] [showperc=<bool>]
```
- **Find Most Common Values:** Identifies the most frequent values for a specified field.
- **Defaults:** Returns the top 10 results with a percentage column.
- **Options:**
  - `by <field>`: Groups results by another field.
  - `countfield`: Renames the count field.
  - `limit`: Limits the number of returned values.
  - `showperc`: Controls whether the percentage column is shown.

**Examples:**
```
index=security sourcetype=linux_secure (fail* OR invalid)
| top src_ip

index=network sourcetype=cisco_wsa_squid
| top cs_username x_webcat_code_full limit=5 showperc=f countfield="Total Viewed"
```

---

### `| rare` Command

Syntax:
```
| rare (<field>|<field-list>)
```
- **Find Least Common Values:** Identifies the rarest field values.
- **Options:** Similar to `top`, it supports options like `limit` and `showperc`.

**Example:**
```
index=sales sourcetype=vendor_sales
| rare product_name showperc=f limit=1
```

---

### `| stats` Command

Syntax:
```
| stats <stats-function>(<wc-field>) [as <wc-field>] [by <field-list>]
```
- **Aggregate Data:** Performs statistical operations on search results.
- **Functions:** Calculates statistics such as `count`, `sum`, `avg`, `min`, `max`, and more.

#### Common Aggregation Functions:
- **`count`**: Returns the count of events.
- **`sum(X)`**: Sums numeric values of field `X`.
- **`avg(X)`**: Returns the average value of field `X`.
- **`dc(X)`**: Counts distinct values of `X`.
- **`stdev(X)`**: Calculates the standard deviation of field `X`.

#### Multi-Value Functions:
- **`list(X)`**: Lists all values of field `X`.
- **`values(X)`**: Lists unique values of field `X`.

**Examples:**
```
index=security sourcetype=linux_secure (invalid OR failed)
| stats count by user, app, vendor_action

index=security sourcetype=linux_secure
| stats count(vendor_action) as ActionEvents, count as TotalEvents
```
