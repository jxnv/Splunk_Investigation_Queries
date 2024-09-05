### `eval` Command

- **Purpose:** Calculates an expression and stores the result in a new or existing field for use in the search pipeline. 
- **Syntax:**
  ```splunk
  ...| eval <field1>=<expression1>[, <field2>=<expression2>]
  ```

- **Operators Supported:**

  | **Type**         | **Operators**                    |
  |------------------|----------------------------------|
  | **Arithmetic**   | `+`, `-`, `*`, `/`, `%`         |
  | **Concatenation**| `+`, `.`                         |
  | **Boolean**      | `AND`, `OR`, `NOT`, `XOR`       |
  | **Comparison**   | `<`, `>`, `<=`, `>=`, `!=`, `=`, `LIKE` |

#### Example
Calculate sales performance and discount:

```splunk
index=sales sourcetype=vendor_sales VendorCountry="United States"
| stats sum(price) as Sales by VendorStateProvince
| eval Performance = case(Sales<=500,"Needs immediate evaluation",
Sales<1000,"Underperformer",Sales>=1000,"Overperformer")
| eval Verdict = if(Performance IN("Underperformer","Needs immediate evaluation"), "Send to marketing",null())
| eval Sales = "$".tostring(Sales,"commas")
```

- **Notes:**
  - Field values are case-sensitive.
  - String values must be "double-quoted".
  - Field names should be unquoted or single-quoted if they include special characters like spaces.
  - Use a period (`.`) instead of plus (`+`) when concatenating strings and numbers to avoid conflicts.

#### Ways to Write Multiple `eval` Expressions

- **Separate Expressions:**

  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | stats sum(sc_bytes) as bytes by usage
  | eval bandwidth = bytes/(1024*1024)
  | eval bandwidth = round(bandwidth, 2)
  ```

- **Nested Expressions:**

  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | stats sum(sc_bytes) as bytes by usage
  | eval bandwidth = round(bytes/(1024*1024), 2)
  ```

- **Linked Expressions:**

  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | stats sum(sc_bytes) as bytes by usage
  | eval bandwidth = bytes/(1024*1024),
    bandwidth = round(bandwidth, 2)
  ```

#### Referencing `eval` Fields
Temporary fields created using `eval` can be used in subsequent commands:

```splunk
index=network sourcetype=cisco_wsa_squid
| stats sum(sc_bytes) as Bytes by usage
| eval bandwidth = round(Bytes/pow(1024,2), 2)
| sort -bandwidth
| rename bandwidth as "Bandwidth (MB)"
```

---

### Evaluation Functions

- **Purpose:** Evaluate expressions based on events and return results.
- **Categories:**
  - Conversion
  - Comparison and Conditional
  - Cryptographic
  - Informational
  - Statistical
  - Multivalue
  - etc.

#### Comparison & Conditional Functions

| **Function**     | **Syntax**                         | **Description**                                               |
|------------------|------------------------------------|---------------------------------------------------------------|
| `case(X,"Y",…)`  | `case(X,"Y",…)`                     | Returns first value (Y) where expression (X) is TRUE.        |
| `cidrmatch("X",Y)`| `cidrmatch("X",Y)`                | TRUE or FALSE if IP matches CIDR notation.                   |
| `if(X,Y,Z)`      | `if(X,Y,Z)`                         | Returns Y if X is TRUE, otherwise Z.                         |
| `like(<string>,<pattern>)` | `like(<string>,<pattern>)`   | TRUE if <string> matches <pattern>.                          |
| `in(<field>,<value-list>)` | `in(<field>,<value-list>)`  | TRUE if value in <value-list> matches a value in <field>.    |
| `match(SUBJECT,"<regex>")` | `match(SUBJECT,"<regex>")`  | TRUE or FALSE if SUBJECT matches <regex>.                    |

#### Additional Functions

| **Function**    | **Syntax**                        | **Description**                                                |
|-----------------|-----------------------------------|----------------------------------------------------------------|
| `true()`        | `true()`                          | Always returns TRUE.                                          |
| `false()`       | `false()`                         | Always returns FALSE.                                         |
| `null()`        | `null()`                          | Always returns NULL.                                          |
| `nullif(X,Y)`   | `nullif(X,Y)`                     | Returns NULL if X=Y, otherwise returns X.                     |
| `validate(X,Y,…)` | `validate(X,Y,…)`               | Returns Y for first expression (X) that is FALSE.              |
| `searchmatch(X)` | `searchmatch(X)`                | TRUE if search string X matches the event.                    |
| `replace(X,Y,Z)` | `replace(X,Y,Z)`                 | Substitutes Z for every instance of regex pattern Y in X.      |

#### Using Evaluation Functions
Functions can be used in `eval`, `fieldformat`, `where`, and `stats` commands:

```splunk
...| eval <field>=function(…)
...| where function(…)
...| fieldformat <field>=function(…)
...| stats stats-func(eval(function(…)))
```

- **Example with `true`, `false`, `null`, and `nullif`:**

  ```splunk
  index=web sourcetype=access_combined
  | eval engagement = if(isnull(action),"no engagement",action)
  | table action, engagement
  ```

### `eval` Command Functions

#### `if` Function
- **Purpose:** Returns Y if expression X is TRUE; otherwise, returns Z.
- **Syntax:**
  ```splunk
  ...| eval <field> = if(X,Y,Z)
  ```
- **Example:**
  ```splunk
  index=sales sourcetype=vendor_sales
  | eval SalesTerritory = if((VendorID>=7000 AND VendorID<8000), "Asia", "Rest of the World")
  | stats sum(price) as TotalRevenue by SalesTerritory
  | eval TotalRevenue = "$".tostring(TotalRevenue, "commas")
  ```

#### `case` Function
- **Purpose:** Evaluates Boolean expressions and returns the corresponding value for the first TRUE expression.
- **Syntax:**
  ```splunk
  ...| eval <field> = case(X1,Y1,X2,Y2,…)
  ```
- **Example:**
  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | eval Risk = case(x_wbrs_score >= 5,"1 Very Safe",
    x_wbrs_score >= 3,"2 Safe",
    x_wbrs_score >= 0,"3 Neutral",
    x_wbrs_score >= -5,"4 Dangerous",
    x_wbrs_score < -5, "5 Very Dangerous")
  | timechart count by Risk
  ```

- **Example with `true()` Function:**
  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | eval Risk = case(x_wbrs_score >= 5,"1 Very Safe",
    x_wbrs_score >= 3,"2 Safe",
    x_wbrs_score >= 0,"3 Neutral",
    x_wbrs_score >= -5,"4 Dangerous",
    x_wbrs_score < -5, "5 Very Dangerous",
    true(), "Not Known")
  | timechart count by Risk
  ```

#### `validate` Function
- **Purpose:** Returns Y for the first FALSE expression X. Returns NULL if all expressions are TRUE.
- **Syntax:**
  ```splunk
  ...| eval <field> = validate(X1,Y1,X2,Y2,…)
  ```
- **Example:**
  ```splunk
  index=security sourcetype=linux_secure
  | eval portWarning = validate(src_port > 0, "ERROR: Port is not a positive integer",
    port >= 1 AND port <= 9900, "WARNING: Port is out of range")
  ```

#### `in` Function
- **Purpose:** Returns TRUE if any value in `<value-list>` matches a value from `<field>`.
- **Syntax:**
  ```splunk
  ...| eval <field> = if(in(<field>,<value-list>),Y,Z)
  ```
- **Example:**
  ```splunk
  sourcetype=access_*
  | eval error = if(in(status, "404","500","503"),"true","false")
  | stats count by error
  ```

#### `searchmatch` Function
- **Purpose:** Returns TRUE if an event matches the search string X.
- **Syntax:**
  ```splunk
  ...| eval <field> = if(searchmatch(X),Y,Z)
  ```
- **Example:**
  ```splunk
  sourcetype=win_audit
  | eval eventInfo = if(searchmatch("642 8 SuccessAudit BUSDEV-003 Security 224542437"),_raw,"not found")
  | where eventInfo!="not found"
  | table eventInfo _time
  ```

#### `cidrmatch` & `match` Functions
- **Purpose:** 
  - `cidrmatch("X",Y)`: Returns TRUE if IP address Y matches subnet X.
  - `match(SUBJECT,"<regex>")`: Returns TRUE if SUBJECT matches regex pattern `<regex>`.
- **Syntax:**
  ```splunk
  ...| eval <field> = if(cidrmatch("X",Y), "TRUE_VALUE", "FALSE_VALUE")
  ...| eval <field> = if(match(SUBJECT,"<regex>"), "TRUE_VALUE", "FALSE_VALUE")
  ```
- **Example for `cidrmatch`:**
  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | eval isLocal = if(cidrmatch("10.2/16",bcg_ip), "IS local sub2", "NOT local sub2")
  ```

- **Example for `match`:**
  ```splunk
  index=network sourcetype=cisco_wsa_squid
  | eval proper_ip_address = if(match(src,"^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$"), "true", "false")
  ```

#### `replace` Function
- **Purpose:** Substitutes Z for every occurrence of regex pattern Y in X.
- **Syntax:**
  ```splunk
  ...| eval field1 = replace(X,Y,Z)
  ```
- **Example:**
  ```splunk
  index=sales sourcetype=sales_entries
  | stats count by AcctCode
  | eval AcctCode = replace(AcctCode, "(\d{4}-)\d{4}", "\1xxxx")
  ```

- **Example with Multiple Capture Groups:**
  ```splunk
  index=sales sourcetype=sales_entries
  | eval clientip_new = replace(clientip, "(\d+\.)\d+\.\d+(\.\d+)", "\1xxx.xxx\2")
  | table clientip, clientip_new
  ```

#### `fieldformat` Command
- **Purpose:** Changes the format of a field's values without altering the underlying value.
- **Syntax:**
  ```splunk
  ...| fieldformat <field>=<eval-expression>
  ```
- **Example:**
  ```splunk
  index=security
  | stats count as totalCount by sourcetype
  | fieldformat totalCount = tostring(totalCount, "commas")
  ```

### `where` Command

- **Purpose:** Filters search results based on `<eval-expression>`.
- **Syntax:**
  ```splunk
  ...| where <eval-expression>
  ```

- **Operators:**

  | **Mathematical Operators** | **Boolean Operators** |
  |-----------------------------|------------------------|
  | `+`, `-`, `*`, `/`, `%`   | `AND`, `OR`, `NOT`    |
  | `<`, `>`, `<=`, `>=`, `!=`, `=` or `==` |     |

- **Example:**
  ```splunk
  index=web sourcetype=access_combined
  | timechart count(eval(action="changequantity")) as changes, count(eval(action="remove")) as removals
  | where removals > changes
  ```

- **Example with `LIKE` Operator:**
  ```splunk
  ...| where <string> LIKE <pattern>
  ```

- **Example with `like` Function:**
  ```splunk
  ...| where like(<string>,<pattern>)
  ```
### `where` Command Examples

#### Filtering with `like` Function
- **Scenario:** IT wants a list of user accounts similar to "admin" (i.e., starting with "adm").
- **Syntax:**
  ```splunk
  index=security sourcetype=linux_secure
  | where like(user,"adm%")
  | dedup user
  | table user
  ```
- **Alternative Syntax:**
  ```splunk
  index=security sourcetype=linux_secure
  | where user like "adm%"
  | dedup user
  | table user
  ```

#### Filtering with `isnull` and `isnotnull`
- **Scenario:** A sales campaign manager wants to find which 1-hour periods had no sales in Canada over the last 24 hours.
- **Syntax:**
  ```splunk
  index=sales sourcetype=vendor_sales
  | timechart span=1h sum(price) as sum
  | where isnull(sum)
  ```

### `fillnull` Command

- **Purpose:** Replaces null values in specified fields.
- **Syntax:**
  ```splunk
  ...| fillnull [value=<string>] [<field-list>]
  ```
- **Example:**
  ```splunk
  index=sales sourcetype=vendor_sales
  | timechart span=1h sum(price) as sum
  | fillnull value="0" sum
  ```

- **Explanation:**
  - `value=<string>`: Specifies what to replace null values with. Defaults to `0` if not specified.
  - `<field-list>`: Restricts which fields to apply `fillnull` to.

### Summary

This document provides a comprehensive overview of key Splunk commands and functions used for data manipulation and analysis:

1. **`eval` Command**: Essential for calculating expressions and creating new or existing fields. It supports a variety of operators, including arithmetic, concatenation, boolean, and comparison. It can handle multiple expressions simultaneously and allows for temporary fields to be used in subsequent commands. Functions like `if`, `case`, `validate`, `in`, `searchmatch`, `cidrmatch`, `match`, and `replace` enable complex evaluations and data transformations.

2. **Evaluation Functions**: These functions, categorized into conversion, comparison, cryptographic, informational, statistical, and multivalue types, help in evaluating expressions and conditions. Examples include `if`, `case`, `validate`, `in`, `searchmatch`, `cidrmatch`, `match`, and `replace`.

3. **`where` Command**: Filters search results based on conditions. It supports various mathematical and boolean operators, as well as string pattern matching with `LIKE`. The `isnull` and `isnotnull` functions can identify missing values, and the `like` function helps filter results based on string patterns.

4. **`fillnull` Command**: Used to replace null values in specified fields with a default value or a specified replacement. This command is useful for filling gaps in data and ensuring completeness in analysis.

By utilizing these commands and functions, users can effectively manipulate, filter, and analyze data in Splunk to derive meaningful insights and drive decision-making processes.
