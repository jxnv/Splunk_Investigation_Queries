# Correlation Analysis

#### Transaction Command

**Syntax:**

```
| transaction (<field>|<field-list>) [options]
```

**Purpose:**

* Groups related events based on shared values in specified fields.
* Defines grouping behavior using options like `maxspan`, `maxpause`, `maxevents`, `startswith`, and `endswith`.
* Adds `duration`, `eventcount`, and `closed_txn` fields to events.

**Example:**

```
index=web sourcetype=access_combined
| transaction JSESSIONID
```

**Transaction Structure:**

* Contains the raw event data and timestamp of the earliest member.

**Duration and Eventcount Fields:**

* Added by the `transaction` command.
* `duration`: Time difference between the first and last event timestamps.
* `eventcount`: Number of events in the transaction.

**Event Ordering:**

* Ensure events are in **reverse chronological order** before using `transaction`.
* Use `sort -_time` to sort events by time.

**Example with Event Ordering:**

```
index=web sourcetype=access_combined
| ...
| sort -_time
| transaction JSESSIONID endswith=(status=503) maxevents=5
```

#### Transaction Command Options

* **`maxspan`:** Maximum total time between the earliest and latest events.
* **`maxpause`:** Maximum total time between events.

**Example with Options:**

```
index=web sourcetype=access_combined
| transaction clientip maxspan=180s maxpause=1m
| eval duration=tostring(duration,"duration")
| sort -duration
| table clientip duration
```

#### Use Cases

**Example 1: Tracking Web Server Issues**

```
index=web sourcetype=access_combined
| transaction JSESSIONID endswith=(status=503) maxevents=5
```

**Example 2: Analyzing Successful Transactions**

```
index=web sourcetype=access_combined
| transaction JSESSIONID startswith=(status=200) maxevents=4
```

**Example 3: Finding Transactions with Errors**

```
index=web sourcetype=access_combined
| transaction JSESSIONID
| search status=404
| highlight JSESSIONID, 404
```

#### Using `eval` Expressions with `transaction`

**Example: Calculating Purchase Time**

```
index=web sourcetype=access_combined
| transaction clientip JSESSIONID startswith=eval(action="addtocart" AND status=200) endswith=eval(action="purchase" AND status=200)
| table clientip, JSESSIONID, duration, eventcount
```

#### `closed_txn` Field

* Indicates whether a transaction is complete or incomplete.
* `closed_txn=1`: Complete.
* `closed_txn=0`: Incomplete.

#### `keepevicted` Option

* Controls whether incomplete transactions are kept or evicted.
* `keepevicted=1`: Keep incomplete transactions.
* `keepevicted=0`: Only show complete transactions (default).

**Example: Calculating Transaction Success Rate**

```
index=web sourcetype=access_combined
| transaction JSESSIONID startswith=(action=addtocart) endswith=(action=purchase) keepevicted=1
| search action=addtocart
| stats count(eval(closed_txn=0)) as "Total Failed", count(eval(closed_txn=1)) as "Total Completed", count as "Total Attempted"
| eval "Percent Completed" = round(((('Total Attempted' - 'Total Failed')*100)/'Total Attempted'),0)."%"
| table "Total Attempted","Total Completed","Percent Completed"
```

### Optimizing Search for `transaction`

**Key Point:**

* **Perform the search before the `transaction` command** to filter events efficiently.

**Example:**

```
index=web sourcetype=access_combined
| search action=addtocart
| transaction JSESSIONID startswith=(action=addtocart) endswith=(action=purchase) keepevicted=1
| stats count(eval(closed_txn=0)) as "Total Failed",
count(eval(closed_txn=1)) as "Total Completed",
count as "Total Attempted"
| eval "Percent Completed" = round(((('Total Attempted' - 'Total Failed')*100)/'Total Attempted'),0) . "%"
| table "Total Attempted","Total Completed","Percent Completed"
```

### Reporting on Transactions

**Example:**

```
index=web sourcetype=access_combined status=200 action=purchase
| transaction clientip maxspan=10m
| chart count by duration span=log2
```

### Transaction Considerations

* **Resource-intensive:** Use `transaction` judiciously.
* **Suitable Use Cases:**
  * Grouping events on multiple fields.
  * Defining event grouping based on start/end values or time segments.
  * Keeping raw data associated with each event.
* **Alternative: `stats` command:**
  * Faster and more efficient.
  * Can perform calculations.
  * Can group events based on a single field value.

**Example Comparing `transaction` and `stats`:**

**Using `transaction`:**

```
index=web sourcetype=access_combined earliest=-1y latest=now
| transaction JSESSIONID
| table JSESSIONID, action, product_name
| sort JSESSIONID
```

**Using `stats`:**

```
index=web sourcetype=access_combined earliest=-1y latest=now
| stats values(action) as "action", values(product_name) as "product_name" by JSESSIONID
| sort JSESSIONID
```

**Note:**

* `transaction` has a limit of 1000 events per transaction.
* `stats` has no limit.

**Additional Example:**

**Counting failed events by source IP using `transaction`:**

```
index=security sourcetype=linux_secure failed
| transaction src_ip
| table src_ip, eventcount
| sort - eventcount
```

**Using `stats`:**

```
index=security sourcetype=linux_secure failed
| stats count as eventcount by src_ip
| sort - eventcount
```


### Subsearches

* **Purpose:** Execute a search within another search and provide results as arguments.
* **Syntax:** Enclosed in square brackets.
* **Execution:** Executed first.
* **Starting Command:** Must begin with a generating command (e.g., `inputlookup`, `search`).
* **Combination:** Results are combined with an OR boolean and attached to the outer search with an AND boolean.

**Example:**

```
index=indexName sourcetype=sourcetypeName
[subsearch]
| additional commands
```

**Multiple Subsearches and Nesting:**

* You can use multiple subsearches in a single search.
* Subsearches can be nested within each other.

**Filtering Data:**

* Subsearches are excellent for filtering data that's difficult to describe directly in a search expression.

**Example:**

```
index=network sourcetype=cisco_wsa_squid
[search index=network sourcetype=cisco_wsa_squid
| stats count by username
| sort -count
| return 1 username]
| stats count by usage, username
| stats list(usage) as usage, list(count) as count by username
```

### Commands That Use Multiple Searches

* **`append`:** Attaches subsearch results to the end of the outer search.
* **`appendcols`:** Appends fields of subsearch results to the outer search results.
* **`union`:** Combines results from multiple datasets (including subsearches).
* **`join`:** Combines results based on shared fields.

### `append` Command

**Purpose:** Combines results from dissimilar searches.

**Example:**

```
index=web sourcetype=access* productId=* action=purchase status=200
earliest=-1h@h latest=@h
| stats sum(price) as lastHourSales by productId
| append
[search index=web sourcetype=access* productId=* action=purchase
status=200 earliest=-2h@h latest=-1h@h
| stats sum(price) as previousHourSales by productId]
| stats first(*) as * by productId
```

### `appendcols` Command

**Purpose:** Appends fields from subsearch results to outer search results.

**Example:**

```
index=security sourcetype=linux_secure "failed password" NOT "invalid user"
| timechart count as known_users
| appendcols
[search index=security sourcetype=linux_secure "failed password"
"invalid user"
| timechart count as unknown_users ]
```

### `join` Command

**Purpose:** Combines results based on shared fields.

**Example:**

```
index=sales sourcetype=vendor_sales VendorID>=9000
| join product_name
[ search index=sales sourcetype=vendor_sales
VendorID>=7000 AND VendorID<9000
| dedup product_name]
| dedup product_name
| table product_name
```

### `union` Command

**Purpose:** Combines results from multiple datasets.

**Example:**

```
| union datamodel:"training.winauth_user_fails"
[ search index=security sourcetype=linux_secure FAIL* user=* ]
| eval user = coalesce(user, User)
| dedup user
| table user
| sort user
```

**Note:** The `union` command can be used with data models, saved searches, lookups, and subsearches.


