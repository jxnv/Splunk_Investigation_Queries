# Security Analyst Investigation Notes Splunk
This document contains Splunk queries for SOC investigation purposes. These prompts can be copied and pasted directly into Splunk, with "KeyArtifact" replaced by the appropriate search term relevant to your investigation.

```markdown
## Table of Contents
- [Detailed Table of Events Filtered by Key Artifacts](#detailed-table-of-events-filtered-by-key-artifacts)
- [Deduplication and Generating Stats for Key Artifact Sources](#deduplication-and-generating-stats-for-key-artifact-sources)
- [Top IPs Generating Events](#top-ips-generating-events)
- [Failed Login Attempts](#failed-login-attempts)
- [Successful Login Attempts](#successful-login-attempts)
- [User Activity Monitoring](#user-activity-monitoring)
- [Admin Activity Tracker](#admin-activity-tracker)
- [MFA Checker (Success)](#mfa-checker-success)
- [MFA Checker (Failure)](#mfa-checker-failure)
- [MFA Checker (Neutral)](#mfa-checker-neutral)
- [Authentication Review, 24 Hours](#authentication-review-24-hours)
- [Threat IP Prevalence](#threat-ip-prevalence)
- [Threat IP, Allowed or Denied](#threat-ip-allowed-or-denied)
```
## Detailed Table of Events Filtered by Key Artifacts
This query provides a detailed table of events filtered by key artifacts, including time, source, destination, and other relevant metadata.

```splunk
index=* KeyArtifact
| table _time, index, source, sourcetype, eventtype, action, dest, src, src_ip, dest_ip, app, status, signature
| rename _time as "Timestamp", user as "UserName", src as "Source", dest as "Destination", src_ip as "Source_IP", dest_ip as "Destination_IP", app as "Application"
| sort by Timestamp
```

## Deduplication and Generating Stats for Key Artifact Sources
This query deduplicates data by index, sourcetype, and source, and then generates statistics for each key artifact.

```splunk
index=* KeyArtifact
| dedup index, sourcetype, source
| stats count by index, sourcetype, source
| sort by index
```

## Top IPs Generating Events
This query retrieves the top 25 IP addresses generating events based on the key artifact.

```splunk
index=* KeyArtifact
| top limit=25 src_ip
```

## Failed Login Attempts
This query filters for failed login attempts by searching for actions or statuses indicating failure.

```splunk
index=* KeyArtifact action="failed" OR status="failure"
| table _time, user, src_ip, dest_ip, app, status
| sort by _time
```

## Successful Login Attempts
This query filters for successful login attempts, displaying relevant information such as time, user, source, and destination.

```splunk
index=* KeyArtifact action="success" OR status="success"
| table _time, user, src_ip, dest_ip, app, status
| sort by _time
```

## User Activity Monitoring
This query monitors user activity, including source and destination IPs, applications, and actions.

```splunk
index=* KeyArtifact
| table _time, user, action, src_ip, dest_ip, app
| sort by _time
```

## Admin Activity Tracker
This query tracks administrator activities, displaying the latest information for each key artifact.

```splunk
index=* KeyArtifact
| stats latest(_time) as Timestamp, values(user) as Username, values(host) as Host, values(process) as Process, values(commandline) as CommandLine, values(action) as Action
| table Timestamp, Username, Host, Process, CommandLine, Action
| sort by Timestamp
```

## MFA Checker (Success)
This query checks for successful multi-factor authentication (MFA) events, filtering by key artifacts and displaying detailed event information.

```splunk
index=* "mfa" "success" KeyArtifact
| stats latest(_time) as Timestamp, values(user) as Username, values(host) as Host, values(event) as Event, values(src) as Source, values(ip) as IP, values(src_ip) as Source_IP
| table Timestamp, Username, Host, Event, Source, IP, Source_IP
```
[earliest="timestamp"] can be used from this command, to review activity of user after authentication. 

## MFA Checker (Failure)
This query checks for failed multi-factor authentication (MFA) events, filtering by key artifacts and displaying detailed event information.
```splunk
index=* "mfa" "failure" KeyArtifact
| stats latest(_time) as Timestamp, values(user) as Username, values(host) as Host, values(event) as Event, values(src) as Source, values(ip) as IP, values(src_ip) as Source_IP
| table Timestamp, Username, Host, Event, action, reason, result, Source, IP, Source_IP
```
## MFA Checker (Neutral)
This query checks for failed or succcessful multi-factor authentication (MFA) events, filtering by key artifacts and displaying detailed event information.
```splunk
(index=* "mfa" ("success" OR "failure") KeyArtifact)
| eval Status=if(searchmatch("success"), "Success", "Failure")
| stats latest(_time) as Timestamp, values(user) as Username, values(host) as Host, values(event) as Event, values(src) as Source, values(ip) as IP, values(src_ip) as Source_IP, values(action) as Action, values(reason) as Reason, values(result) as Result, count by Status
| table Timestamp, Username, Host, Event, Status, Action, Reason, Result, Source, IP, Source_IP
| sort by Timestamp
```
## Authentication Review, 24 hours
Review the last 24 hours of an alerting user's authentication.
```splunk
index=* "authentication" KeyArtifact
| where _time >= relative_time(now(), "-24h@h")
| stats count as Total_Attempts, values(action) as Actions, values(host) as Host, values(src_ip) as Source_IP, values(result) as Results, earliest(_time) as First_Attempt, latest(_time) as Last_Attempt by user
| eval Duration = tostring(Last_Attempt - First_Attempt, "duration")
| table user, Total_Attempts, Actions, Host, Source_IP, Results, First_Attempt, Last_Attempt, Duration
| sort by Last_Attempt desc
```

## Threat IP Prevalence
This query checks for prevalence of threat IP in the environment
```splunk
index=* KeyArtifact
| stats count as Occurrences, dc(src_ip) as Unique_Source_IPs, dc(dest_ip) as Unique_Destination_IPs, values(src_ip) as Source_IPs, values(dest_ip) as Destination_IPs, values(host) as Hosts, earliest(_time) as First_Seen, latest(_time) as Last_Seen
| table Occurrences, Unique_Source_IPs, Unique_Destination_IPs, Source_IPs, Destination_IPs, Hosts, First_Seen, Last_Seen
| sort by Last_Seen desc
```

## Threat IP, allowed or denied
This query checks for if traffic to threat IP is allowed by the firewall.
```splunk
index=* ("allow" OR "allowed" OR "accept" OR "accepted" OR "pass" OR "permitted" OR "success" OR "granted" OR "okay" OR "forwarded" OR "opened" OR "passed" OR "deny" OR "denied" OR "drop" OR "dropped" OR "block" OR "blocked" OR "reject" OR "rejected" OR "failure" OR "refused" OR "reset" OR "stopped" OR "closed" OR "filtered") KeyArtifact KeyArtifact
| stats count as Occurrences, values(action) as Action, values(source_ip) as Source_IP, values(dest_ip) as Destination_IP, values(src_port) as Source_Port, values(dest_port) as Destination_Port, values(protocol) as Protocol, values(host) as Host, earliest(_time) as First_Seen, latest(_time) as Last_Seen
| table Occurrences, Action, Source_IP, Destination_IP, Source_Port, Destination_Port, Protocol, Host, First_Seen, Last_Seen
| sort by Last_Seen desc
```

