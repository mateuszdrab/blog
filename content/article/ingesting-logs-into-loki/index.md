---
title: "Ingesting logs into Loki"
date: 2025-01-27T14:50:00+00:00
tags: ['logs', 'Loki', 'promtail', 'Grafana']
draft: false
categories: 
 - Observability 
---

Here is the standard Loki log processing flow that I use for my logs.

The pipeline is comprised of the following stages:

- adding `job` label (so that I can query all logs ingested from files)
- add `directory` label (by obtaining the directory name from the `filename` label)
- packing the `filename` label into the log entry using the `stage.pack` stage (reducing cardinality of the labels, querying can be done by the `directory` label)
- adding `hostname` and `agent_hostname` labels to the logs (`agent_hostname` refers to the machine running the agent, `hostname` is obtained from the logs. This is not implemented on my Windows Agent configuration, but it is designed for situations where agent might be handing logs from other sources, for example, syslog or event log)
- dropping the `computer` label
- dropping logs older than 1 hour (aligns with server side configuration to minimize errors)

All file logs enter at the `loki.relabel.file.receiver`, Windows event logs enter at the `loki.relabel.default.receiver`.

```
loki.relabel "default" {
  forward_to      = [loki.process.final.receiver]

  rule {
    action        = "replace"
    replacement   = constants.hostname
    target_label  = "hostname"
  }
  
  rule {
    action        = "replace"
    replacement   = constants.hostname
    target_label  = "agent_hostname"
  }

  rule {
    action        = "labeldrop"
    regex         = "computer"
  }
}

loki.process "final" {
  forward_to      = [loki.write.default.receiver]

  stage.drop {
    older_than  = "1h"
    drop_counter_reason = "line_too_old"
  }
}

loki.relabel "file" {
  forward_to      = [loki.process.file.receiver]

  rule {
    action        = "replace"
    replacement   = "file"
    target_label  = "job"
  }

  rule {
    action        = "replace"
    regex         = "(.*)\\\\.*"
    replacement   = "$1"
    target_label  = "directory"
    source_labels = ["filename"]
  }
  
}

loki.process "file" {
  forward_to      = [loki.relabel.default.receiver]

  stage.pack {
    labels = ["filename"]
    ingest_timestamp = false
  }
}
```
