# Syslog Router Integration for Elastic

> Note: This AI-assisted guide was validated by our engineers to ensure it is technically accurate.

## Overview

The Syslog Router integration for Elastic routes incoming syslog events to the correct Elastic integration data stream using regex pattern matching on the `message` field. It acts as a centralized traffic controller for syslog messages, allowing a single Elastic Agent to receive a mixed stream of logs from multiple network devices and forward each event to its appropriate integration-specific data stream for parsing.

This integration facilitates:

- Consolidating syslog from many different network devices (Cisco, Fortinet, Palo Alto, and more) onto a single listener port instead of managing separate listeners for every vendor.
- Automatically identifying log sources and routing them to specialized data streams such as `cisco_asa.log` or `panw.panos`.
- Extending the routing logic with custom patterns to handle proprietary or non-standard syslog formats.

### Compatibility

This integration supports routing events from 22 pre-configured integrations out of the box:

- Arista NG Firewall
- Check Point
- Cisco ASA
- Cisco FTD
- Cisco IOS
- Cisco ISE
- Cisco Secure Email Gateway
- Citrix WAF (CEF format only)
- Fortinet FortiEDR
- Fortinet FortiGate
- Fortinet FortiMail
- Fortinet FortiManager
- Fortinet FortiProxy
- Imperva SecureSphere (CEF format only)
- Iptables
- Juniper SRX
- Palo Alto Next-Gen Firewall
- QNAP NAS
- Snort
- Sonicwall Firewall
- Sophos XG
- Stormshield

**DISCLAIMER**: Due to subtle differences in how devices can emit syslog events, the default patterns may not work in all cases. Some integrations that support syslog are not listed here because their patterns would be too complex or could overlap with other integrations, which might cause false matches. Custom patterns may need to be created for those cases.

This integration is compatible with Elastic Stack version 8.14.3 or later, or 9.0.0 and above.

### How it works

This integration receives syslog events through TCP, UDP, or Filestream inputs. Each incoming event is evaluated against an ordered list of regex patterns defined in the **Reroute configuration**. When a pattern matches the `message` field, the `_conf.dataset` field is set to the target integration's data stream name (for example, `cisco_asa.log`). The integration's routing rules then reroute the event to that target data stream, where the target integration's ingest pipeline handles parsing. Events that do not match any pattern remain in the `syslog_router.log` data stream.

## What data does this integration collect?

This integration acts as a transit layer. It collects raw syslog events and routes them to other Elastic integrations for parsing. The integration itself has a minimal ingest pipeline that sets `ecs.version` and handles errors.

Data is collected through the following inputs:

- **Syslog events (TCP):** Listens for incoming TCP syslog connections on a configurable address and port (default: `localhost:9514`).
- **Syslog events (UDP):** Listens for incoming UDP syslog packets on a configurable address and port (default: `localhost:9514`).
- **Syslog events (Filestream):** Monitors local log files (default: `/var/log/syslog.log`). Disabled by default.

Based on the routing configuration, data is directed toward specialized integrations including:
- **Network security logs:** Firewall traffic and security policy events (for example, `cisco_asa.log`, `panw.panos`, `fortinet_fortigate.log`).
- **Authentication and identity logs:** Identity services and access logs (for example, `cisco_ise.log`, `citrix_waf.log`).
- **Intrusion detection alerts:** IDS/IPS signatures (for example, `snort.log`, `fortinet_fortiedr.log`).
- **System events:** Hardware health and configuration changes (for example, `qnap_nas.log`, `arista_ngfw.log`).

## What do I need to use this integration?

The Syslog Router is an Elastic-built routing tool, not a third-party vendor integration. There are no vendor-side prerequisites beyond configuring your network devices to send syslog to the Elastic Agent host.

- An Elastic Agent installed and enrolled in a Fleet policy on a host that can receive syslog traffic from network devices.
- Network connectivity from the syslog-sending devices to the Elastic Agent host on the configured listen port (default `9514` for TCP/UDP).
- The Elastic integration assets for each target data stream must be installed in Kibana before events can be correctly parsed (for example, install the Cisco ASA integration assets before routing Cisco ASA syslog events).

## How do I deploy this integration?

### Agent-based deployment

Elastic Agent must be installed on a host that will act as the syslog receiver. For more details, check the Elastic Agent [installation instructions](docs-content://reference/fleet/install-elastic-agents.md). You can install only one Elastic Agent per host.

### Set up steps: Install target integration assets

Before adding the Syslog Router, install the assets for each integration you want to route to:

1. In Kibana, navigate to **Management > Integrations**.

2. Find the relevant integration by searching or browsing the catalog. For example, the Cisco ASA integration.

![Cisco ASA Integration](../img/catalog-cisco-asa.png)

3. Navigate to the **Settings** tab and click **Install Cisco ASA assets**. Confirm by clicking **Install Cisco ASA** in the popup.

![Install Cisco ASA assets](../img/install-assets.png)

4. Repeat for each integration whose syslog events you expect to receive.

### Set up steps: Configure syslog on network devices

Configure each network device to forward syslog to the Elastic Agent host on the configured port. For example:

- **Cisco ASA**: Use the `logging host` command to point to the Elastic Agent host and port.
- **Palo Alto PAN-OS**: Create a Syslog Server Profile under **Device > Server Profiles > Syslog** pointing to the Elastic Agent.
- **Fortinet FortiGate**: Configure syslog under **Log & Report > Log Settings** with the Elastic Agent host as the destination.

Refer to each vendor's documentation for specific syslog forwarding instructions.

### Set up steps in Kibana

1. In Kibana, navigate to **Management > Integrations**.
2. Search for **Syslog Router** and select it.
3. Click **Add Syslog Router**.
4. Enable and configure the desired input(s):
   - **TCP input**: Set **Listen Address** (`localhost` by default) and **Listen Port** (`9514` by default). Optionally configure **SSL Configuration** for encrypted syslog transport.
   - **UDP input**: Set **Listen Address** (`localhost` by default) and **Listen Port** (`9514` by default).
   - **Filestream input** (disabled by default): Set **Paths** to the syslog file locations (default: `/var/log/syslog.log`).
5. Review the **Reroute configuration** YAML. The integration comes pre-configured with 22 patterns. Modify the list as needed for your environment (refer to [Configuration](#configuration) below).
6. Select the **Elastic Agent policy** to assign this integration to.
7. Click **Save and continue**.

## Configuration

### Overview

The integration comes preconfigured with a number of pattern definitions. The pattern definitions are used in the order given. Care must be taken to ensure the patterns are executed in the correct order. Regular expressions which are more relaxed and could potentially match against multiple integrations should be run last and stricter patterns should be run first. The next priority should be given to integrations which will see the most traffic.

Pattern definitions may be reordered by moving the entire `if/then` block up or down in the list. For example, moving **Imperva SecureSphere** above **Cisco ASA**:

**Before:**

```yaml
- if:
    and:
      - not.has_fields: _conf.dataset
      - regexp.message: "%ASA-"
  then:
    - add_fields:
        target: ''
        fields:
          _conf.dataset: "cisco_asa.log"
          _conf.tz_offset: "UTC"
          _temp_.internal_zones: ['trust']
          _temp_.external_zones: ['untrust']
- if:
    and:
      - not.has_fields: _conf.dataset
      - regexp.message: "CEF:0\\|Imperva Inc.\\|SecureSphere"
  then:
    - add_fields:
        target: ''
        fields:
          _conf.dataset: "imperva.securesphere"
    - decode_cef:
        field: message
```

**After:**

```yaml
- if:
    and:
      - not.has_fields: _conf.dataset
      - regexp.message: "CEF:0\\|Imperva Inc.\\|SecureSphere"
  then:
    - add_fields:
        target: ''
        fields:
          _conf.dataset: "imperva.securesphere"
    - decode_cef:
        field: message
- if:
    and:
      - not.has_fields: _conf.dataset
      - regexp.message: "%ASA-"
  then:
    - add_fields:
        target: ''
        fields:
          _conf.dataset: "cisco_asa.log"
          _conf.tz_offset: "UTC"
          _temp_.internal_zones: ['trust']
          _temp_.external_zones: ['untrust']
```

Individual pattern definitions can be turned off by removing the definition entirely or by inserting comment characters (`#`) in front of the appropriate lines:

```yaml
# - if:
#     and:
#       - not.has_fields: _conf.dataset
#       - regexp.message: "%ASA-"
#   then:
#     - add_fields:
#         target: ''
#         fields:
#           _conf.dataset: "cisco_asa.log"
#           _conf.tz_offset: "UTC"
#           _temp_.internal_zones: ['trust']
#           _temp_.external_zones: ['untrust']
```

### Adding New Patterns

Example configuration:

```yaml
- if:
    and:
      - not.has_fields: _conf.dataset
      - regexp.message: "CEF:0\\|Imperva Inc.\\|SecureSphere"
  then:
    - add_fields:
        target: ''
        fields:
          _conf.dataset: "imperva.securesphere"
    - decode_cef:
        field: message
```

At its core, the Syslog Router integration uses the [built-in conditionals and processors](https://www.elastic.co/guide/en/beats/filebeat/current/defining-processors.html) provided within Beats. While there are certain requirements that need to be maintained, additional conditions and processors can be added, if required.

The top level of each configuration contains an `if`/`else` condition. In the `if` statement, an `and` combines two conditions. The first ensures that another match has not already occurred, while the second condition is a `regex`, or regular expression, which performs the actual match. If the regular expression matches the `message` field, then the processors in the `then` statement of the configuration will run.

If multiple patterns are required, they may be combined with an `or` condition:

```yaml
- if:
    and:
      - not.has_fields: _conf.dataset
      - or:
        - regexp.message: <PATTERN_1>
        - regexp.message: <PATTERN_2>
```

In the `then` statement, a list of processors can be given. At minimum, an `add_fields` processor needs to be added with the following fields:

**Required fields:**

- `_conf.dataset`: The dataset (`integration.data_stream`) to forward to. This field is used by the routing rules in the integration to route documents to the correct pipeline.

Additional processors, such as `decode_cef` or `syslog`, can be provided if additional processing is required.

### Validation

1. Send a test syslog message to the Elastic Agent host on the configured port. For example, for Cisco ASA: `echo "<190>%ASA-6-302013: test message" | nc localhost 9514`.
2. In Kibana, navigate to **Analytics > Discover**.
3. Select the `logs-*` data view.
4. Search for the test event using KQL: `data_stream.dataset : "cisco_asa.log"`.
5. Verify the event was routed to the correct data stream and parsed by the target integration's pipeline.
6. To check for unmatched events, filter for `data_stream.dataset : "syslog_router.log"` and examine the `message` field.

## Troubleshooting

### Common Configuration Issues

- **Port binding failure**: If the Elastic Agent fails to start the listener, verify the configured port (for example, `9514`) is not already in use by another syslog service. On Linux, use `ss -tulpn | grep <port>` to identify conflicts.
- **Events routed to the wrong integration**: Check the order of `if/then` blocks in the **Reroute configuration**. Stricter patterns (such as CEF headers or vendor-specific strings) should appear before more relaxed patterns that could match multiple vendors.
- **Events remain in `syslog_router.log` instead of the target data stream**: The event did not match any pattern. Examine the `message` field against the configured regex patterns. You may need to add a custom pattern for your device's syslog format.
- **Routed events are not parsed correctly**: Ensure the target integration's assets are installed in Kibana. The Syslog Router only routes events; the target integration's ingest pipeline handles parsing.

### Ingestion Errors

- **`error.message` is present on routed events**: The target integration's ingest pipeline encountered a parsing error. Verify that the syslog format matches what the target integration expects. Some integrations require specific formats (for example, Citrix WAF requires CEF format).
- **CEF format issues**: For Citrix WAF and Imperva SecureSphere, ensure the source device is explicitly configured to export in CEF format. Standard syslog formats will not match the default router patterns for these vendors.
- **Missing `_conf.dataset` field**: If this field is absent, the event defaults to the `syslog_router.log` stream. Review the `message` field and verify it matches a regex defined in the routing configuration.

### Vendor Resources

- [Elastic Beats Processors Documentation](https://www.elastic.co/guide/en/beats/filebeat/current/defining-processors.html) - Documentation for the conditional logic and processors used in the Syslog Router configuration.

## Performance and scaling

- **Pattern ordering matters**: Patterns are evaluated in order and stop at the first match. Place the most frequently matched patterns near the top to reduce regex evaluations per event. Remove unused vendor blocks to minimize CPU overhead.
- **Transport selection**: UDP offers higher throughput with lower overhead. TCP is recommended where delivery guarantees are required. When using TCP, tune options such as `max_connections` and `max_message_size` to match your environment.
- **Agent scaling**: For high-throughput environments, deploy multiple Elastic Agents behind a network load balancer to distribute the ingestion load.

For more information on architectures that can be used for scaling this integration, check the [Ingest Architectures](https://www.elastic.co/docs/manage-data/ingest/ingest-reference-architectures) documentation.

## Reference

### log

The `log` data stream is the transit data stream for all syslog events. Events are routed from this data stream to their target integration data stream based on the pattern matching configuration.

#### log fields

{{ fields "log" }}

### Inputs used

{{ inputDocs }}
