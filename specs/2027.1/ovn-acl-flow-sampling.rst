..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
OVN ACL flow sampling for Network Log
=====================================

https://bugs.launchpad.net/neutron/+bug/2165306

The ML2/OVN logging driver uses ACL logging to send packet events to
ovn-controller. This proposal adds an output choice to Network Log so that
operators can request OVN ACL sampling instead. Neutron will manage the
sampling resources associated with the selected ACLs. Operators will configure
the host exporters and the systems that receive and process the samples.


Problem Description
===================

OVN supports sampling packets that match an ACL, including traffic belonging
to established connections. An operator can attach this configuration directly
to the OVN Northbound database, but Neutron does not maintain it. For example,
changing a security group's statefulness recreates its rule ACLs and loses
their sampling references. Deleting or disabling a Network Log does not
remove sampling configured manually on its ACLs.

Network Log already records which security group and events an operator wants
to observe. Extending that resource allows the OVN driver to maintain sampling
with the corresponding ACLs. An operator should be able to enable sampling for
one security group, keep controller logging for another, and remove either
request through the Network Log API.

The proposal covers ACL sampling for security groups on ML2/OVN. It does not
define a VPC flow-record API, an aggregation interval, or a storage service.
A sample is an observation of a packet; it is not a complete connection record
or an exact packet and byte counter.


Proposed Change
===============

Network Log API
---------------

A new ``logging-output-type`` extension will add ``output_type`` to the Log
resource, with values ``packet_log`` and ``flow_sample``. The default,
``packet_log``, preserves the backend's current logging behavior for existing
Logs and requests that omit the field.

The attribute will be accepted on create and returned by show and list. It
will support equality filtering and will not be mutable. A caller that needs
a different output will create a replacement Log and remove the old one.
The existing ``enabled`` attribute will control either output. A Log will
select one output; separate Logs can request both outputs for the same ACL.

For example, an operator can request sampling as follows. The resource UUID
in this example represents an existing security group::

    POST /v2.0/log/logs
    {
        "log": {
            "name": "web-samples",
            "resource_type": "security_group",
            "resource_id": "d80f19a6-5404-4c2d-953d-7a8910d8ac60",
            "event": "ALL",
            "output_type": "flow_sample",
            "enabled": true
        }
    }

Omitting ``output_type`` from the same request will select ``packet_log``.
Null, empty, and unknown output values will be rejected with HTTP 400, as will
an attempt to update the output of an existing Log.

The database change will be an expand migration adding the output column with
a ``packet_log`` server default. The Log versioned object will add the field
and retain that default when reading an older object. A flow-sample Log must
not be converted to an older object by discarding its output: an old consumer
would interpret it as a request for packet logging.

Initial scope and policy
------------------------

An OVN security-group ACL applies to a Port Group. Looking up the security
groups attached to a target port does not restrict those ACLs to that port.
A shared security group can also contain ports from several projects.

For ``flow_sample``, the request will require ``resource_type=security_group``
and an explicit ``resource_id``. A non-null ``target_id`` or an omitted
``resource_id`` will be rejected with HTTP 400. Sampling will cover the
security group's Port Group, including ports subsequently attached to it.
This proposal does not add per-port ACL copies or change the shared
default-drop architecture to implement a narrower selector.

The existing Network Log policy permits administrators and project managers.
The new output will additionally require an administrative policy check on
creation and on re-enabling a flow-sample Log, since sampling a shared SG can
include another project's traffic. Packet-log permissions will retain their
existing defaults. Existing Log read, disable, and delete policy checks will
continue to apply; these operations will not require sampling to be enabled
in the deployment.

Supporting project-manager requests or individual ports later will require
keeping sampling within the requested scope as SG membership and sharing
change. Checking the SG's project only when the Log is created is insufficient.

Deployment requirements and capability
--------------------------------------

The extension alias will indicate that the API understands the new field.
The existing loggable-resources response will describe the outputs supported
for each resource type. In a deployment with sampling enabled it will include::

    {
        "loggable_resources": [
            {
                "type": "security_group",
                "output_types": ["packet_log", "flow_sample"]
            }
        ]
    }

Sampling will be disabled by default. Advertising ``flow_sample`` will require
the operator's opt-in, the OVN logging driver, and support for the necessary
Northbound sampling tables and ACL columns. The deployment must also use an
OVN/OVS combination that supports the required sampling and ACL-label behavior
on every participating chassis. Schema detection alone cannot establish this
dataplane compatibility.

The initial implementation will not advertise flow sampling when an additional
non-OVN logging driver is loaded. Driver dispatch will distinguish outputs so
that a flow-sample Log cannot reach a packet-only driver or its agent RPC path.
Existing packet-log dispatch will be retained.

Requests for an unsupported output will fail with HTTP 400. Validation will
also apply to Logs created with ``enabled=false`` and will run again before
re-enabling a Log. A detected ownership or identifier conflict in the
Northbound database will be reported as HTTP 409. A database connection failure
will follow the logging service's driver-error handling rather than being
reported as an unsupported output.

Capability discovery will not report exporter or collector health. Operators
will configure OVS ``Flow_Sample_Collector_Set`` and the associated exporter
on the relevant bridges, and verify sample delivery separately. Flow-based
IPFIX uses the probability carried by the sample action. The OVS
``IPFIX.sampling`` setting applies to bridge sampling and is not an additional
sampling rate for this path.

Sampling resources and ownership
--------------------------------

When sampling is enabled, Neutron will manage ``Sampling_App`` rows for
``acl-new`` and ``acl-est`` and one dedicated ``Sample_Collector``. The
collector will initially use one probability for both stages and all selected
ACLs. Per-Log probabilities and exporter destinations are outside this change.

Operators will supply two distinct application IDs, a collector ID, a
collector-set ID, and a probability. Application and collector IDs are in
the range 1 through 255; the collector-set ID is a nonzero 32-bit value.
The probability will be in the range 1 through 65535. The enable flag will
default to false; identifiers will not have defaults that silently claim
cluster-wide resources.

There can be only one Sampling_App row for each application type. Neutron
will reject an existing row owned by another manager rather than adopting it,
even if its current values match the requested configuration. It will also
check application-ID and collector-ID conflicts. Collector names are not
unique keys and will not be used alone to identify an owned row. The generic
``drop`` sampling application is separate from sampling packets at drop ACLs
and will not be managed by this feature.

The application and collector rows will carry Neutron ownership metadata.
They will remain deployment resources while the feature is enabled, even if
there are temporarily no flow-sample Logs. Removing the last Log will not
delete them. Decommissioning will remove only owned rows that are no longer
referenced by other users. Neutron will not modify the host-local OVS exporter
configuration.

A selected stateful ``allow-related`` ACL will reference a Sample through
both ``sample_new`` and ``sample_est``. Stateless allowing ACLs and drop ACLs
will use ``sample_new``; they do not gain an established-connection event by
setting ``event=ALL`` on a Log. Logs selecting the same ACL will share its
sampling configuration. Sample creation and ACL attachment will occur in one
Northbound transaction. Removing an attachment will allow OVSDB's
strong-reference garbage collection to remove an otherwise unreferenced
Sample.

Sample has no ``external_ids`` column. Ownership therefore cannot be recorded
on that row. Neutron will record attachment ownership and the expected
reference on the ACL, and validate the referenced Sample's metadata and
collector before changing or removing it. If the reference is foreign or
differs from the ownership record, Neutron will report a conflict and leave
it unchanged. Checks and updates must be in the same IDL transaction, with
retry on concurrent changes.

ACL selection and overlapping Logs
----------------------------------

``ACCEPT`` will select the SG's allowing ACLs. ``DROP`` will use the
SG-specific drop ACLs maintained by the OVN logging driver, and ``ALL``
will select both.
The existing priority and match of those drop ACLs will be retained. Their
presence will depend on whether either output still requires them. A
flow-sample-only request will not enable controller logging as a side effect
of creating a drop ACL.

A drop observation identifies the selected drop ACL. It does not identify an
explicit SG deny rule, since security groups contain allow rules and an
implicit default deny. If a port belongs to multiple SGs, an observation from
one of their drop ACLs must not be presented as proof that this SG alone
caused the denial.

For each ACL, the driver will calculate the union of enabled Logs separately
for each output. With packet Log A and sampling Log B, both outputs will be
present. Deleting B will detach sampling while preserving A's packet logging.
With sampling Logs B and C, deleting B will leave sampling attached for C.
An output will be removed only when no enabled Log requires it.

The calculation will include the existing packet logging fields and the
lifecycle of SG-specific drop ACLs. It cannot be implemented by independently
clearing all logging fields when one Log is deleted: ACL labels are also
used by the sampling path. Reconciliation will preserve packet logging's
related-traffic behavior when sampling is added or removed.

Observation identity
--------------------

Sample metadata is a nonzero 32-bit identifier with a Northbound uniqueness
constraint. Neutron will allocate it with collision detection and bounded
retry, and reuse it while the same owned attachment remains valid. Repeated
reconciliation and a neutron-server restart will not allocate new Samples
for unchanged attachments.

A nonzero ACL label can take precedence over Sample metadata as the emitted
observation point. Existing packet logging uses that label, so this feature
will not clear it to make sampling work. The supported dataplane combinations
must preserve sampling when packet logging and sampling coexist, including
with a single collector and register-based sampling.

The observation domain also contains a logical datapath identifier. Depending
on the OVN path and the presence of an ACL label, its application portion may
distinguish new and established observations or be zero. Consumers cannot
assume that application IDs always separate these stages. Operators will need
the applicable OVN mapping to interpret exported identifiers.

These identifiers are not Log UUIDs or permanent SG-rule identifiers. Adding
or removing packet logging can change the effective observation point through
the ACL label. Replacing an ACL or restoring a Northbound database can also
change the mapping. Established connections can retain earlier conntrack
metadata across a configuration change. The feature does not promise an
instantaneous remapping of every existing connection or retroactive samples.

Lifecycle and recovery
----------------------

The SQL Log records the requested state. Updating it and programming OVN are
separate transactions. A postcommit driver failure can leave a stored Log
whose output has not yet been applied. Clients can use list and show before
retrying a failed create. This change will not add a delivery-status resource
or imply that API success confirms reception by an external collector.

Log operations and SG ACL creation will use the same output-state calculation.
The driver will reapply sampling when SG rules are created or replaced,
including replacement caused by a statefulness change. Port Group membership
will determine the traffic observed by an existing shared ACL.

A periodic task under the OVN maintenance lock will reconcile SQL intent with
owned Northbound state. It will inspect enabled Logs and stale attachments
left by disabled or deleted Logs. Looking only for existing SQL rows would
miss failed cleanup after deletion. Log resources are not already covered
by OVN's resource-revision maintenance, so this reconciliation must be added
explicitly.

After a conflict or concurrent change, reconciliation will reread the current
Logs before retrying conditional Northbound updates. It will also reevaluate
the stored state after reconnection or a change of maintenance leader.
Ownership conflicts will be logged with the affected resource for the
operator to resolve.

Database synchronization will restore sampling on recreated SG-rule ACLs.
SG-specific logging drop ACLs must also be reconciled explicitly; the current
rule-ACL comparison does not reconstruct them. Repeating either recovery path
with unchanged intent must preserve attachment and metadata values.

Upgrade and disable
-------------------

Operators will enable flow sampling after all relevant API workers and
services understand the new Log object and output field. The default-off
setting prevents older workers from encountering flow-sample intent during
the initial rolling upgrade. Configuration must agree across workers before
the feature is advertised.

Disabling the deployment option will reject new flow-sample requests and
enable operations. Existing Logs will remain available for read, disable, and
delete. When Northbound is reachable with the required schema, reconciliation
will detach owned sampling references while retaining stored Log intent.
Re-enabling the option will apply enabled Logs again. If Northbound is
unreachable, disabling the option cannot guarantee immediate cessation of
sampling. Before a downgrade, operators must disable the feature and verify
that attachments have been removed.

Alternatives
------------

A deployment-wide switch would require fewer API changes, but would not let
operators choose different outputs for individual Logs. Replacing packet
logging outright would change existing deployments without an explicit request.

An external tool could continue to manage Sample rows and ACL attachments.
It would also need to follow Neutron's SG and ACL lifecycle. Keeping attachment
management in the logging driver lets it use the same intent as the Log API.
Exporter configuration and sample processing remain external responsibilities.

Reusing operator-provided global sampling rows would support deployments with
another owner of the acl-new and acl-est applications. It would also split
configuration and recovery responsibility between managers. The initial
implementation instead requires exclusive ownership of those application rows
and reports an existing foreign owner as a deployment conflict.


Implementation
==============

The implementation consists of:

* The neutron-lib API extension and Log database and object changes.
* Output validation and dispatch in the logging service.
* Sampling resource and ACL attachment management in the OVN logging driver.
* Maintenance and database synchronization support for restoring sampling.

Any ovsdbapp command needed for conditional attachment management will include
transaction tests. Dataplane compatibility, particularly ACL-label coexistence,
must be established before advertising the feature.

The existing ``openstack network log`` commands are provided by the
python-neutronclient OSC plugin. Create will gain ``--output-type``; show,
list, and loggable-resources output will expose the new fields. This work does
not require a new standalone client command or an SDK resource migration.


Testing
========

API and object tests will cover omitted output, invalid values, immutability,
disabled creation, re-enabling, policy, capability, existing database rows,
and conversion of older objects. Driver tests will ensure that flow-sample
intent is not sent through packet-only drivers or their RPC paths.

OVN functional tests will exercise owned and foreign global rows, identifier
collisions, foreign ACL references, shared attachments, and both output
deletion orders. They will verify that flow-sample-only SG drop ACLs do not
enable controller logging. Recovery tests will cover postcommit failure,
concurrent Log changes, restart, ACL replacement, database synchronization,
and disable while Northbound is unavailable.

Dataplane tests will cover new, established, and reply traffic, stateless
rules, DROP, and both ACL directions. They must include nonzero labels with
a single collector, packet logging before and after sampling, and connections
that predate an output change. The expected result includes unchanged packet
filtering and packet-log behavior as well as samples reaching the configured
exporter. A separate delivery test will verify flow-based IPFIX reception;
an absent collector must not alter the ACL's allow or drop decision.


Documentation Impact
====================

API documentation will describe the output attribute, initial selector and
policy restrictions, and capability response. The logging guide will cover
the opt-in configuration, supported OVN/OVS combinations, ownership conflicts,
operator-managed exporter setup, and the disable and downgrade sequence.
It will distinguish sample identifiers and statistical observations from
per-Log records, exact counters, and guaranteed collector delivery.


References
==========

* RFE: https://bugs.launchpad.net/neutron/+bug/2165306
* Drivers discussion:
  https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-09-11-13.00.log.html
* OVN ACL sampling configuration (ACL, Sample, Sample_Collector, Sampling_App):
  https://www.ovn.org/support/dist-docs/ovn-nb.5.html
* OVN logical flows for ACL sampling:
  https://docs.ovn.org/en/latest/ref/ovn-logical-flows.7.html
* OVN sample action and observation identifiers:
  https://www.ovn.org/support/dist-docs/ovn-sb.5.html
* Host IPFIX exporter configuration (OVS IPFIX and Flow_Sample_Collector_Set):
  https://www.openvswitch.org/support/dist-docs/ovs-vswitchd.conf.db.5.html
* Related flow-log RFE: https://bugs.launchpad.net/neutron/+bug/2071323
