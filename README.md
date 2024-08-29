## Splunk_Investigation_Queries
SOC Investigation Splunk Queries, these prompts can be copied and pasted, with only the replacement of "KeyArtifact" for an effective framework of an investigation

## General Splunk Queries/Tables

# Detailed Table of Events Filtered by Key Artifacts
Index=* KeyArtifact
| Table _time, index, source, sourcetype, eventtype, action, dest, src, src_ip, dest_ip, app, status, signature
| rename _time as "Timestamp", user as "UserName", src as "Source", dest as "Destination", src_IP as "Source_IP", dest_ip as "Destination_IP", app as "Application"
| sort by Timestamp

# Deduplication and generating stats for the key artifact from sources
Index=* KeyArtifact
| dedup index, sourcetype, source
| stats count by index, sourcetype, source
| sort by index

#  Top IPs Generating Events
index=* KeyArtifact
| top limit=25 src_ip

# Failed Login Attempts
index=* KeyArtifact action="failed" OR status="failure"
| table _time, user, src_ip, dest_ip, app, status
| sort by _time

# Successful Login Attempts
index=* KeyArtifact action="success" OR status="success"
| table _time, user, src_ip, dest_ip, app, status
| sort by _time
