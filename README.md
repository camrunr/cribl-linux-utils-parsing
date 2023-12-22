# Linux Utils Parsing
----

This pack will provide some basic tooling to parse various linux tools' outputs. Currently we support df, iostat, netstat, ps and vmstat. It's mostly intended as a demonstration of how to run commands and parse their output, but hopefully it provides some actual utility as well.

## Requirements Section
You will need to set-up something to run the commands and forward the raw output to Cribl. We recommend Edge, but others work too.

- ps :: Run with `ps auwx`
- netstat :: Run as `netstat -na`
- iostat :: Run as `iostat -x -k 5 2`
- vmstat :: Run as `vmstat -S m 5 2`
- df :: Run as `df -m`

Note: For iostat and vmstat we want to see 2 samples separated by 5 seconds to getrecent data. The first entry is since reboot, which is less useful to record.

## Using The Pack

To use the Pack, send data from the supported commands through the Pack. Out of the box, we will output metrics data only. The raw data is not kept. Check inside each pipeline to change or disable the metrics conversion/aggregation.

## Getting the data in

Create a source in your collection agent of choice, point the output to Stream, and finally to this Pack. Procedures for different agents will vary. Your incoming data will need to be tagged with a sourcetype field according to the Routes inside the pipeline. For example, events from `df` should be tagged with a `sourcetype` of `df`.

Below is a general outline for 2 different methods of getting the data in, and tagged properly:

---
### 1. Using Cribl Edge as the Collection Agent

Cribl Edge is controlled through the same Leader node you use for Cribl Stream.

#### In Cribl Edge context:

1) More -> Sources -> Exec
2) Using `df` as an example, add a new Exec source, and name it `df`
3) Enter the path to the df binary and the arguments required: `/usr/bin/df -m`
4) In Fields, add a new field for sourcetype => `'df'`
5) in Event Breakers choose the **Cribl Do Not Break** ruleset
   - We want the event to come accross in one chunk
5) Use Quick Connect to point data to your Stream Worker instances
   - If you haven't already created a destination to send Edge events to Stream, now is a good time!
   - Create a source on the **Stream** side for Cribl TCP, and a destination on the **Edge** side. Make sure the ports match.
   - This will enable Edge to send data –> Stream over TCP
5) Repeat as needed for other commands
6) Commit and deploy

#### In Cribl Stream context:
1) Create a route that points `sourcetype == 'df'` data to this Pack and to your intended destinations
2) Repeat as needed for other commands

The end result is a flow something like this:

    Edge runs command -> Edge reads output -> Cribl-TCP -> Stream -> Pack -> Destination

Alternatively, you could install the Pack in the Edge context, and apply the Pack in the Quick Connect interface. The data would then arrive in Stream already processed ready for aggregation, archive, final delivery, or whatever other goal you have.

    Edge runs command -> Edge reads output -> Pack -> Cribl-TCP -> Stream -> (other functions?) -> Destination

---
### 2. Agentless Data Ingest Using netcat and systemd Timers

Instead of using Cribl Edge or any other agent, you could collect this data just by scheduling commands to run periodically.

1) Ensure you have access to netcat (usually installed as `nc`) on the target systems
2) **In Stream** Using the included JSON included below, create a LinuxUtils event breaker ruleset
   - The breaker rules are based on prefixing the command output with `sourcetype=df` (or `iostat`, or whatever)
   - The rules add a sourcetype field to events aligned with this prefix (eg, sourcetype => `df`)
3) Set-up a raw TCP source in Cribl Stream and assign the EB from the previous step to it.
   -  We used port 10060 for this example. You are free to choose any port, but it must be reachable, and the scripts below must use it.
3) Create a Route in the routing table with filter `cribl_breaker.includes('LinuxUtils')`, and point it to this Pack and your destination
4) **On systems you want to collect from**
- Using `df` as an example, create a file called df.sh containing the following and make it executable:
```
    #!/bin/bash
    (echo sourcetype=df; /usr/bin/df -m) | /usr/bin/nc -NC your_stream_host 10060
```
- Create a file at /etc/systemd/system/df.service:
```
    [Unit]
    Description=Run DF

    [Service]
    Type=oneshot
    ExecStart=/path_to_your/df.sh
```
- Create a file at /etc/systemd/system/df.timer:
```
    [Unit]
    Description=df timer

    [Timer]
    OnUnitActiveSec=10s
    OnBootSec=10s

    [Install]
    WantedBy=timers.target
```
- Reload systemd: `systemctl daemon-reload`
- Enable your timer: `systemctl enable df.timer`
- Start your timer: `systemctl start df.timer`

Repeat this step for the other inputs types as required.

The desired end state is a flow something like this:

    systemd runs command -> TCP(via nc) -> Stream -> Pack -> Destination

---

There are many other ways to get the data flowing into Cribl. Once there, this Pack will parse into JSON, making it easy to create enrichments, filters, aggregations, and more.

## Release Notes

### Version 1.0.3

- Cleaned up this doc
- Finalized functions for the first 5 commands supported
- Added the event breaker used for direct delivery over nc/tcp
- Fixed mask regex for vmstat and netstat (NL/CR)
- First release


## Contributing to the Pack
To contribute to the Pack, please contact the author, jrust@cribl.io


## License
This Pack uses the [`Apache 2.0 License`](https://github.com/criblio/appscope/blob/master/LICENSE).

## Appendix - LinuxUtils event breaker

```
{
  "id": "LinuxUtils",
  "lib": "custom",
  "minRawLength": 256,
  "rules": [
    {
      "condition": "/^sourcetype=df/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "current",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/^sourcetype=df[\\r\\n]+/",
      "delimiterRegex": "/\\s+/",
      "fieldsLineRegex": "/(.*)/",
      "headerLineRegex": "/^Filesystem/",
      "nullFieldVal": "-",
      "cleanFields": true,
      "name": "df",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'df'"
        }
      ]
    },
    {
      "condition": "/^sourcetype=iostat/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "current",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/^sourcetype=iostat[\\r\\n]+/",
      "name": "iostat",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'iostat'"
        }
      ]
    },
    {
      "condition": "/^sourcetype=netstat/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "current",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/sourcetype=netstat[\\r\\n]+/",
      "name": "netstat",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'netstat'"
        }
      ]
    },
    {
      "condition": "/^sourcetype=ps/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "current",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/^sourcetype=ps[\\r\\n]+/",
      "name": "ps",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'ps'"
        }
      ]
    },
    {
      "condition": "/^sourcetype=vmstat/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "current",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/^sourcetype=vmstat[\\r\\n]+/",
      "name": "vmstat",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'vmstat'"
        }
      ]
    },
    {
      "condition": "/\\d+\\.\\d{6}\\sIP\\s\\w+\\.\\d+/.test(_raw)",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "format",
        "length": 150,
        "format": "%s.%f"
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/[\\r\\n]+/",
      "name": "tcpdump-dns",
      "fields": [
        {
          "name": "sourcetype",
          "value": "'tcpdump:dns'"
        }
      ]
    }
  ],
  "tags": "linux"
}
```
