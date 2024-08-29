# Phishing Investigation Notes
This document contains details on investigation when a phishing email is observed in the environment.

```markdown
## Table of Contents
- [Prevalence of the Alerting Sender](#prevalence-of-the-alerting-sender)
- [Prevalence of the Alerting Subject Line](#prevalence-of-the-alerting-subject-line)
- [Prevalence of the Alerting Hash](#prevalence-of-the-alerting-hash)
- [Prevalence of the Alerting URL](#prevalence-of-the-alerting-url)
- [Query for Successful Navigation to the Phishing URL](#query-for-successful-navigation-to-the-phishing-url)
- [Activity Review of the Alerting User](#activity-review-of-the-alerting-user)
- [Unusual Access Patterns](#unusual-access-patterns)
- [Account and Device Correlation](#account-and-device-correlation)
```
## Prevalance of the alerting sender.

```splunk
index=* KeyArtifact 
| stats count values(recipient) as Recipients by sender
| table sender, Recipients, count

```
## Prevalance of the alerting subjectline
```splunk
index=* KeyArtifact
| stats count values(recipient) as Recipients by subject
| table subject, Recipients, count
```
## Prevalance of the alerting hash
```splunk
index=* KeyArtifact
| stats count values(recipient) as Recipients by sha256_hash, sha_hash, md5_hash
| table sha256, sha, md5, hash, Recipients, count
```

## Prevalance of the alerting URL
```splunk
index=* KeyArtifact 
| stats count values(recipient) as Recipients by url
| table url, Recipients, count
```

## Query for Successful Navigation to the Phishing URL
```splunk
index=* KeyArtifact
| search status="success" OR action="navigate" OR action="allowed"
| stats count values(user) as Users by url
| table url, Users, count
```

## Activity Review of the alerting user
```splunk
index=* earliest=-4h KeyArtifact
| stats count values(action) as Actions, values(src_ip) as Source_IPs, values(dest_ip) as Destination_IPs, values(app) as Applications, values(status) as Statuses by recipient, _time
| table recipient, _time, Actions, Source_IPs, Destination_IPs, Applications, Statuses, count
```

## Unusual Access Patterns
```splunk
index=* KeyArtifact earliest=-7d
| eval hour_of_day=strftime(_time, "%H")
| stats count by recipient, hour_of_day
| table recipient, hour_of_day, count
```

## Account and Device Correlation
```splunk
index=* KeyArtifact
| stats count values(device_id) as Devices, values(account_id) as Accounts by recipient
| table recipient, Accounts, Devices, count
```
