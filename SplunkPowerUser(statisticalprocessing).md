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

## `stats` Command: Functions

The `stats` command provides various statistical functions organized into four categories:

- **Aggregate:** Summarizes event values into a single value.
- **Event Order:** Returns values based on field processing order.
- **Multivalue:** Returns a list of values for a field as a multivalue entry.
- **Time:** Returns values based on time.

### `stats` Command: Examples

#### **Distinct Count (`dc`)**
Calculates the count of unique values for a specified field.

```splunk
index=network sourcetype=cisco_wsa_squid
| stats dc(s_hostname) as "Websites Visited"
```

#### **Sum (`sum`)**
Aggregates the total of numerical values for a field, grouped by another field.

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as Bandwidth by s_hostname
| sort -Bandwidth
```

### Multiple Functions Example
You can apply multiple functions to the same search.

```splunk
index=sales sourcetype=vendor_sales
| stats count(price) as "Units Sold", sum(price) as "Total Sales" by product_name
| sort -"Total Sales"
```

### `stats` Command: `min`, `max`, `avg`

```splunk
index=network sourcetype=cisco_wsa_squid
| stats min(sc_bytes) as "Minimum Bytes",
max(sc_bytes) as "Maximum Bytes",
avg(sc_bytes) as "Average Bytes" by usage
```

### `list` Example
Lists every value, including duplicates, for a specified field.

**Scenario:** Find out which websites each employee accessed during the last 60 minutes.

```splunk
index=network sourcetype=cisco_wsa_squid
| stats list(s_hostname) as "Websites Visited" by cs_username
```

### `stats` Command: Full Scenario Example
**Scenario:** Marketing wants to perform a statistical analysis on APAC retail sales by country over the last 24 hours.

```splunk
index=sales sourcetype=vendor_sales VendorID>=7000 AND VendorID<9000
| stats count(price) as count, sum(price) as Total_Sales, min(price) as Minimum, max(price) as Maximum, range(price) as Range, avg(price) as Average, median(price) as Median, stdev(price) as sdev, var(price) as Variance by VendorCountry
| eval Average=round(Average,2), sdev=round(sdev,2), Variance=round(Variance,2)
| sort -Total_Sales
| rename count as "Units Sold", sdev as "Standard Deviation"
```
##Transforming Commands Summary
| Feature                  | `chart`                        | `timechart`                    | `stats`                        |
|--------------------------|--------------------------------|--------------------------------|--------------------------------|
| **`by` clause arguments** | Supports 2 fields              | Supports 1 field               | Supports many fields           |
| **Limit series shown**    | `limit=<int>` (default: 10)    | `limit=<int>` (default: 10)    | Not applicable (NA)            |
| **Filter other series**   | `useother=t`                   | `useother=t`                   | Not applicable (NA)            |
| **Filter null values**    | `usenull=t`                    | `usenull=t`                    | Not applicable (NA)            |
| **Set value groups along the x-axis** | `span=<int>`         | `span=<int>`                   | Not applicable (NA)            |


## eval Command
...| eval <field1>=<expression1>[, <field2>=<expression2>]
• Calculates an expression and puts the resulting value into a new
field or overwrites an existing field
• Fields created by eval can be reused in the search pipeline
• Extremely useful command that supports a vast assortment of
functions for performing specific tasks
• Can exist as a nested function in certain scenarios

## `eval` Command Operators

| **Type**           | **Operators**                         |
|--------------------|---------------------------------------|
| **Arithmetic**      | `+`, `-`, `*`, `/`, `%`              |
| **Concatenation**   | `+`, `.`                             |
| **Boolean**         | `AND`, `OR`, `NOT`, `XOR`            |
| **Comparison**      | `<`, `>`, `<=`, `>=`, `!=`, `=` `LIKE`|

### Syntax Example

```splunk
index=sales sourcetype=vendor_sales VendorCountry IN("United States", "Canada")
| stats sum(price) as "USA+Canada Sales", count as "Total Products Sold"
count(eval(VendorCountry = "United States")) as "Products Sold in US",
count(eval(VendorCountry = "Canada")) as "Products Sold in Canada" by product_name
| eval "USA+Canada Sales" = "$".'USA+Canada Sales'
```

- Field values are case-sensitive.
- String values must be double-quoted.
- Field names must be unquoted or single-quoted if they contain special characters like spaces.
- Use a period (`.`) for concatenating strings and numbers to avoid conflicts with the `+` operator.

### Multiple `eval` Examples

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as bytes by usage
| eval bandwidth = bytes / (1024 * 1024)
| eval bandwidth = round(bandwidth, 2)
```

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as bytes by usage
| eval bandwidth = round(bytes / (1024 * 1024), 2)
```

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as bytes by usage
| eval bandwidth = bytes / (1024 * 1024), bandwidth = round(bandwidth, 2)
```

### Referencing `eval` Fields

Temporary fields created with `eval` can be referenced in subsequent commands in the search pipeline.

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as Bytes by usage
| eval bandwidth = round(Bytes / pow(1024, 2), 2)
| sort -bandwidth
| rename bandwidth as "Bandwidth (MB)"
```

## Evaluation Functions Overview

- **Evaluates expressions** based on events and returns results.
- Categories of evaluation functions include:
  - Conversion
  - Comparison and Conditional
  - Mathematical
  - Informational
  - Statistical
  - Text
  - Etc.

| Category      | Function       | Syntax      | Description                                                                 |
|---------------|----------------|-------------|-----------------------------------------------------------------------------|
| Mathematical  | round          | round(X,Y)  | Rounds X to Y decimal places, otherwise returns X as a whole number         |
| Mathematical  | pow            | pow(X,Y)    | Returns X to the power of Y                                                 |
| Statistical   | max            | max(X,…)    | Takes an arbitrary number of arguments and returns the maximum              |
| Statistical   | min            | min(X,…)    | Takes an arbitrary number of arguments and returns the minimum              |
| Statistical   | random         | random()    | Takes no arguments and returns a random integer                             |

### Statistical Functions: `max`, `min`, `random`

- **`max`**: Takes an arbitrary number of numeric or string arguments and returns the maximum value.
  - **Note**: Strings are considered greater than numbers.
  
  ```splunk
  ...| eval field1 = max(X1, X2, ...)
  ```

- **`min`**: Takes an arbitrary number of numeric or string arguments and returns the minimum value.
  
  ```splunk
  ...| eval field1 = min(X1, X2, ...)
  ```

- **`random`**: Takes no arguments and returns a pseudo-random integer ranging from 0 to 2^31 - 1.
  
  ```splunk
  ...| eval field1 = random()
  ```
### `eval` Expressions Without Functions
- Expressions can perform calculations or concatenate field values without requiring a function.
  
#### Scenario:
Calculate total online sales for the last week by including `price`, `sale_price`, and `discount percentage`. Then sort by descending discount value.
  
```splunk
index=web sourcetype=access_combined product_name=* action=purchase
| stats sum(price) as tlp, sum(sale_price) as tsp by product_name
| eval Discount = (((tlp - tsp)/tlp) * 100)
| eval Discount = round(Discount)
| sort -Discount
| eval Discount = Discount . "%"
```

### `eval` Command with Multiple Expressions
- Link two or more expressions together to create multiple fields, using functions or performing simple calculations and concatenations.

```splunk
index=web sourcetype=access_combined price=*
| stats values(price) as list_price, values(sale_price) as current_sale_price by product_name
| eval current_discount = round((list_price - current_sale_price) / list_price * 100, 2),
new_discount = (current_discount - 5),
new_sale_price = list_price - (list_price * (new_discount / 100))
```

### `eval` Command as a Function
- Nest `eval` within the `stats count` function to count events with a specific field value.
- Field values are case-sensitive and should be wrapped in double quotes.
- Requires an `as` clause to rename the field.

#### Scenario:
Count the number of events that occurred yesterday where the `vendor_action` was `Accepted`, `Failed`, or `session opened`.

```splunk
index=security sourcetype=linux_secure vendor_action=*
| stats count(eval(vendor_action="Accepted")) as Accepted,
count(eval(vendor_action="Failed")) as Failed,
count(eval(vendor_action="session opened")) as SessionOpened
```

### `rename` Command
```splunk
...| rename <field> as <newfield>
```
- Use the `rename` command to give fields more meaningful or user-friendly names.
- For field names with spaces or special characters, enclose them in double straight quotes (`" "`).

#### Scenario:  
Display the `clientip`, `action`, `productId`, and `status` of customer interactions in the online store for the last 4 hours.

```splunk
index=web sourcetype=access_combined
| table clientip, action, productId, status
| rename productId AS ProductID, action AS "Customer Action", status AS "HTTP Status"
```

- Once a field is renamed, the new field name must be used in subsequent commands.

```splunk
index=web sourcetype=access_combined
| table clientip, action, productId, status
| rename productId AS ProductID, action AS "Customer Action", status AS "HTTP Status"
| table action, status
```
- This will return no results because the original field names are no longer valid after renaming.

```splunk
index=web sourcetype=access_combined
| table clientip, action, productId, status
| rename productId AS ProductID, action AS "Customer Action", status AS "HTTP Status"
| sort "Customer Action", "HTTP Status"
```

- You can also use wildcards (`*`) to rename multiple fields matching a pattern.

```splunk
index=web sourcetype=access_combined
| table productId, product_name
| rename product* AS PROD*
| table PROD*
```

---

### `sort` Command
```splunk
...| sort (-|+) <field> [limit=<int> | <int>]
```
- The `sort` command sorts results in descending (`-`) or ascending (`+`) order.
- Ascending order is the default if no sign is provided.
- Use the `limit` option to restrict the number of results, or simply provide a number.
- Sorting is determined by the data type of the field:
  - Alphabetic strings: Lexicographic sorting
  - Numeric values: Numerical sorting
  - Combination: Sorting is based on the first character, whether numeric or alphabetic.

#### Scenario:  
Display vendor information for vendors starting with "Bea", and sort the results in descending order.

```splunk
index=sales sourcetype=vendor_sales Vendor=Bea*
| dedup Vendor, VendorCity
| table Vendor, VendorCity, VendorStateProvince, VendorCountry
| sort - Vendor
```

- If you want to apply the sort order to multiple fields, add a space after the `+` or `-` sign.

```splunk
| sort - Vendor, VendorCity
```



