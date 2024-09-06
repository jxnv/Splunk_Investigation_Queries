# Manipulating Output

### **untable Command**
- Converts a 2-dimensional (stats-like) output into a flat table that can be charted or used for further analysis.
- Syntax: `untable <x-field> <y-name-field> <y-data-field>`
- Example: 
  ```spl
  ...| untable product_name VendorCountry prod_count
  ```
  - **Result**: Easier sorting and analysis of product data by VendorCountry.

### **xyseries Command**
- Converts stats-like data into chartable tabular output.
- Syntax: `xyseries <x-field> <y-name-field> <y-data-field>`
- Example:
  ```spl
  ...| xyseries _time host totalBytes
  ```
  - **Result**: Creates chartable output that provides meaningful visualizations across distinct data series.

### **xyseries vs chart**
- The `chart` command can be seen as similar to using `stats` followed by `xyseries`, but the latter is more customizable if you need more manipulation of data before plotting.
  - Equivalent search:
    ```spl
    ...| chart sum(sales) over product_name by clientip
    ...| xyseries clientip product_name sales
    ```

### **appendpipe Command**
- Transforms results and appends new output to the bottom of the result set.
- Syntax: `appendpipe [<subpipeline>]`
- Example:
  ```spl
  index=network sourcetype=cisco_wsa_squid usage!=Business
  | stats count by usage, cs_username
  | appendpipe [stats sum(count) as count by usage]
  ```

- **Multiple `appendpipe` Uses**: You can append multiple transformed results (e.g., usage totals and grand totals) to the original result set for a more comprehensive output.

### eventstats Command

```
...| eventstats statsfunction(<field>) [as <field>]
```

Generates statistics of all existing fields in your search results and saves them as values in new fields.  
• `statsfunction` is applied to `<field>` and the resulting value is appended to each result.  
• Supports multiple functions.

Example:
```
...| eventstats avg(lostSales) as averageLoss
```

Usage with conditions:
```
...| eventstats avg(lostSales) as averageLoss
| where lostSales > averageLoss
```

---

### streamstats Command

```
...| streamstats statsfunction(<field>) [as <field>] [by <field-list>] [window=<int>] [current=<Bool>]
```

Calculates statistics for each result row at the time the command encounters it and adds these values to the results.  
• Supports multiple functions.  
• `window` specifies the number of events to use; default is 0 (all events).  
• `current` includes the current event in summary calculations; default is `current=true`.

Example:
```
index=network sourcetype=cisco_wsa_squid action!="TCP_REFRESH_HIT"
| streamstats count as recentAttempts by bcg_ip
```

Example with sorting:
```
index=web sourcetype=access_combined action=purchase status=200 productId=*
| table _time, price
| sort _time
| streamstats avg(price) as averageOrder window=100 current=f
```

---

### Function Reference Chart

| **Category**  | **Function** | **Syntax**                    | **Description**                                                          |
|---------------|--------------|-------------------------------|--------------------------------------------------------------------------|
| Conversion    | `tostring`   | `tostring(X[,Y])`             | Converts the input (numeric or Boolean) to a string.                      |
|               | `tonumber`   | `tonumber(NUMSTR[,BASE])`     | Converts the input string NUMSTR to a number.                             |
| Text          | `lower`      | `lower(X)`                    | Converts the string X to lowercase.                                       |
|               | `upper`      | `upper(X)`                    | Converts the string X to uppercase.                                       |
|               | `substr`     | `substr(X,Y,Z)`               | Returns a substring of X, starting at Y, with Z characters.               |
| Comparison    | `&`          | `&`                           | Used for comparisons.                                                    |
| Conditional   | `coalesce`   | `coalesce(X,…)`               | Returns the first value that is not NULL from an arbitrary number of arguments. |

---

### Conversion Functions

#### `tostring`
```
...| eval <field> = tostring(X,Y)
```

Converts a numeric field to a formatted string.  
• Optional formatting options include `"commas"`, `"hex"`, and `"duration"`.

Example:
```
index=web sourcetype=access_combined action=purchase status=503
| stats sum(price) as lostRevenue
| eval string_lostRevenue = "$".tostring(lostRevenue)
| table lostRevenue, string_lostRevenue
```

#### `tonumber`
```
...| eval <field> = tonumber(NUMSTR[,BASE])
```

Converts a string to a number.  
• Optional `BASE` can be set from 2 to 36, default is 10.

Example:
```
...| eval myValue="1.4848974e+12"
| eval myValueAsInteger = tonumber(myValue)
```

---

### Text Functions

#### Uppercase and Lowercase Conversion

Example:
```
...| eval uppercase_product = upper(product_name), lowercase_categoryId = lower(categoryId)
| table product_name, categoryId, itemId, uppercase_product, lowercase_categoryId
```

---

### foreach Command

```
...| foreach <wc-field-list> [template-subsearch]
```

Applies a template to multiple fields using the `<<FIELD>>` token as a placeholder.

Example:
```
...| foreach www* [eval <<FIELD>> = round(<<FIELD>>/(1024*1024),2)]
```

### Why Normalize?

- **Consistency:** Data representing the same information can be named differently.
- **Purpose:** Normalization organizes data to appear similar across records, making searches easier.
- **Types of Normalization:** 
  - **Data Normalization**
  - **Field Normalization**
  
Normalization uses eval commands to standardize data for uniformity.

---

### substr Function

```
...| eval <field> = substr(X,Y[,Z])
```

- **Purpose:** Returns a substring of X starting at Y with Z characters.
- **X:** Literal string or existing field.
- **Y:** Start position (positive: from the start, negative: from the end).
- **Z:** (Optional) Number of characters to return (if not specified, returns rest of string).

Example (Extracting employee username):
```
...| eval employee = substr(bcg_workstation,6)
```

Example (Creating a dense field value):
```
...| eval ItemCode = tostring(substr(categoryId, 1, 3)." - ".upper(product_name)." - ".substr(itemId,5))
| table product_name, categoryId, itemId, ItemCode
```

---

### coalesce Function

```
...| eval field1 = coalesce(X1,X2,...)
```

- **Purpose:** Returns the first non-null value from X1, X2, etc.
- **Usage:** Ideal for normalizing fields where multiple field names represent the same data.

Example (Field normalization across logs):
```
(index=network sourcetype=cisco_wsa_squid) OR (index=web sourcetype=access_combined)
| eval oneIP = coalesce(clientip,c_ip)
| table c_ip, clientip, oneIP, sourcetype
```
