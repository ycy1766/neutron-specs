..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Tap-as-a-Service: filtering rules for lport mirrors
===================================================

https://bugs.launchpad.net/neutron/+bug/2168008

A tap mirror copies everything the mirrored port sends and receives. This
spec adds filtering rules to tap mirrors of type ``lport`` so that users can
choose, at the source, which traffic is copied to the collector. The rules
are a new ``rules`` sub-resource of tap mirrors, use the vocabulary of
security group rules, and are implemented with the OVN ``Mirror_Rule`` table
(OVN 25.09 and later).

The ``lport`` mirror type itself is covered by RFE 2168007, which was
approved without a spec at the drivers meeting of 2026-10-02 [1]_. The same
meeting asked for this spec because the rules add a new API and database
table.


Problem Description
===================

With the ``lport`` mirror type a project can send a copy of the traffic of
one of its ports to a collector port (an IDS, a packet broker) inside the
overlay. Today such a mirror always copies everything. That is usually far
more than the collector needs:

* the copies double the traffic of the mirrored port on the overlay and on
  the collector port;
* several mirrored ports often feed the same collector, so the collector has
  to drop most of what it receives;
* security teams want to express the selection where it is cheapest, at the
  source, for example "only inbound TCP/443 and DNS" or "everything except
  the backup traffic to 10.0.9.0/24".

Use cases:

* A project member mirrors a web server port and only wants inbound
  TCP/443 at the collector.
* A project member mirrors a database port and wants everything except the
  replication traffic to a known subnet.
* An operator mirrors a tenant port into a collector in a service project
  (admin only, as for lport mirrors) and only wants traffic to and from the
  internet-facing prefixes.

OVN already provides the mechanism: since 25.09 a ``Mirror`` row of type
``lport`` can reference ``Mirror_Rule`` rows (priority, match, action
``mirror`` or ``skip``). Neutron has no API for it.

Rules are not proposed for the ``gre`` and ``erspanv1`` mirror types. In
OVN those are plain OVS port mirrors configured by ``ovn-controller`` and do
not go through the logical pipeline, so there is no logical flow where a
match could be applied. ``lport`` mirrors are implemented in the
``ls_in_mirror`` and ``ls_out_mirror`` logical switch stages, which is where
the ``Mirror_Rule`` matches are installed.


Proposed Change
===============

Overview
--------

A new API extension ``tap-mirror-rules`` (requires ``tap-mirror`` and
``tap-mirror-lport``) adds a ``rules`` sub-resource to tap mirrors::

  POST   /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}
  DELETE /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}

The sub-resource follows the model of QoS policy rules (neutron-lib
sub-resource definition, controller with a parent, per-rule policies).
Rules are immutable: to change a rule, delete it and create a new one. This
keeps the mapping to OVN rows one-to-one and avoids partial updates of a
rule that is already applied on the dataplane.

Attributes by mirror type
-------------------------

At the RFE review it was pointed out that ``tap_mirrors`` now has attributes
that are only valid for some mirror types [1]_. The API makes this explicit
and rejects an attribute that does not apply, instead of ignoring it:

==========================  =======================  ======================
Attribute                   ``gre`` / ``erspanv1``   ``lport``
==========================  =======================  ======================
``port_id``                 required                 required
``remote_ip``               required                 rejected (``400``)
``directions`` tunnel ids   required, one per        must be ``null``
                            direction
``remote_port_id``          rejected (``400``)       required
``rules`` sub-resource      rejected (``400``)       supported
==========================  =======================  ======================

Today the lport implementation ignores a ``remote_ip`` or tunnel id given
for an lport mirror; it will be changed to reject them, so that a user can
not believe a value is used when it is not. The documentation will carry the
same table.

Rule attributes
---------------

==================================  ==========  =============================
Attribute                           Default     Notes
==================================  ==========  =============================
``priority``                        required    1-32767, higher is evaluated
                                                first
``action``                          ``mirror``  ``mirror`` or ``skip``
``direction``                       ``null``    ``IN``, ``OUT`` or ``null``
                                                (every direction the mirror
                                                copies)
``ethertype``                       ``IPv4``    ``IPv4`` or ``IPv6``
``protocol``                        ``null``    ``tcp``, ``udp``, ``sctp``,
                                                ``icmp``, ``ipv6-icmp``
``source_ip_prefix``                ``null``    CIDR
``destination_ip_prefix``           ``null``    CIDR
``source_port_range_min/max``       ``null``    1-65535, TCP/UDP/SCTP only
``destination_port_range_min/max``  ``null``    1-65535, TCP/UDP/SCTP only
==================================  ==========  =============================

The server translates a rule into an OVN match expression. For example
``protocol=tcp``, ``destination_ip_prefix=10.0.0.0/24`` and
``destination_port_range_min=max=443`` become::

  ip4 && tcp && ip4.dst == 10.0.0.0/24 && tcp.dst == 443

The raw OVN expression is deliberately not exposed: it would tie the API to
one backend's expression language and would need its own validation. A
backend specific attribute can be added later if a real need appears.

Semantics
---------

* **Default allow.** OVN always installs an unconditional flow at priority
  100 that mirrors every packet of the port, and places the rules above it
  at ``100 + priority``. So a mirror without rules, or a packet that matches
  no rule, is mirrored. This is what tap mirrors do today. Users that want
  "mirror only what I list" add a catch-all ``skip`` rule at priority 1.
  AWS VPC Traffic Mirroring and K2 Cloud chose the opposite default (no
  rules, nothing mirrored) at their product layer; the drivers did not ask
  to change it at the RFE review, but reviewers may still prefer it (see
  Alternatives).
* **Priority 0 is rejected** because it would land on the same OpenFlow
  priority as the unconditional flow and the result would be undefined.
* **Direction.** The OVN driver already creates one OVN ``Mirror`` per
  direction for an lport tap mirror (``to-lport`` for ``IN``,
  ``from-lport`` for ``OUT``). A rule with a direction is attached to that
  mirror only; a rule without a direction is attached to every mirror of
  the tap mirror. A rule for a direction the tap mirror does not copy is
  rejected.
* **Relation to security groups** (follows from the pipeline position of
  the mirror stages): traffic sent by the port is matched before its
  security groups are evaluated; traffic delivered to the port is matched
  after them. The copies bypass the security groups of the collector port.
* **Duplicates and equal priorities.** A rule identical to an existing one
  (same priority, direction and match) is rejected with ``409``. Two
  different rules with the same priority and overlapping matches are
  accepted, as in OVN, but which one wins is undefined; the documentation
  will tell users to give overlapping rules distinct priorities. Rejecting
  any second rule with the same priority and direction is a possible
  stricter alternative that reviewers may prefer.
* **Mirror types.** Creating a rule on a ``gre`` or ``erspanv1`` tap mirror
  returns ``400`` with a clear message. Deleting a tap mirror deletes its
  rules.

Data model
----------

A new table ``tap_mirror_rules`` (expand migration) with a foreign key to
``tap_mirrors.id`` (``ON DELETE CASCADE``), ``project_id`` and one column per
attribute above. No change to existing tables; contract migrations are not
needed.

OVN driver
----------

* ``create_tap_mirror_rule_postcommit`` translates the rule and calls
  ``mirror_rule_add`` for the OVN mirror(s) of the selected direction(s);
  ``delete`` calls ``mirror_rule_del``.
* The ``Mirror_Rule`` table and the ``mirror_rule_add/del`` commands come
  from ovsdbapp change 950742 [2]_. The driver checks at runtime that the
  connected Northbound schema has the table. On an older OVN creating a
  rule fails with a clear error; lport mirrors without rules and gre/erspan
  mirrors keep working.
* TaaS has no OVN DB sync for tap mirrors today, so rules have none either.
  Adding a sync for mirrors and rules is out of scope for this spec.
* Other TaaS drivers do not support the extension; with them the extension
  is not loaded.

Policy
------

``create/get/delete_tap_mirror_rule`` default to
``ADMIN_OR_PROJECT_MEMBER`` (``READER`` for ``get``), the same as tap
mirrors. A rule always belongs to the project of its tap mirror.

Clients
-------

openstacksdk gets a ``TapMirrorRule`` resource and proxy methods, and
python-openstackclient the commands::

  openstack tap mirror rule create <mirror> --priority 100 \
      --direction IN --protocol tcp --dst-port 443
  openstack tap mirror rule list <mirror>
  openstack tap mirror rule show <mirror> <rule>
  openstack tap mirror rule delete <mirror> <rule>

Not supported in the initial release
------------------------------------

* Updating a rule in place.
* Rules on ``gre`` and ``erspanv1`` mirrors.
* Quotas on rules. The number of rules per mirror is bounded by the
  priority range; a quota can be added later if operators need one.
* ML2/OVS. TaaS for ML2/OVS uses tap services and tap flows and has no
  equivalent of ``Mirror_Rule``.


Alternatives
------------

API shape
~~~~~~~~~

During the RFE review it was noted that the ``tap_mirrors`` resource now
carries attributes that only apply to some mirror types (``remote_ip`` and
tunnel ids for ``gre``/``erspanv1``, ``remote_port_id`` and rules for
``lport``), and that the TaaS API already mixes tap services/tap flows
(ML2/OVS) with tap mirrors (ML2/OVN) [1]_. Options:

#. **Rules as a sub-resource of tap_mirrors (proposed).** Smallest
   change, reuses the resource the OVN driver implements, one OVN ``Mirror``
   per tap mirror direction. The type specific attributes are validated per
   type and the documentation lists which ones apply.
#. **A separate resource for lport mirrors** (for example
   ``/taas/port_mirrors`` with ``port_id``, ``remote_port_id``,
   ``directions`` and ``rules``). Cleaner attribute set, but a second
   resource for the same OVN object, a second set of policies and clients,
   and the lport mirror API approved in RFE 2168007 would have to move.
#. **Rules as a separate top-level tap_mirror_filters resource** shared
   by several mirrors (the AWS model). More flexible but more complex, and
   OVN ``Mirror_Rule`` rows belong to one ``Mirror``, so sharing would be
   emulated by copying rows.

The proposal is option 1. Reusing the tap service and tap flow resources
instead of tap mirrors was suggested on RFE 2168007; keeping tap mirrors was
agreed at the drivers meeting, because the OVN driver only implements tap
mirrors and an OVN ``Mirror`` maps one-to-one to a tap mirror. A cleanup of the overall TaaS
API (tap services and tap flows for ML2/OVS versus tap mirrors for ML2/OVN)
is worth doing but is independent of this spec and should be proposed in a
separate RFE.

Default deny
~~~~~~~~~~~~

Make a mirror with rules copy only the matching traffic. This is what AWS
and K2 Cloud do. It is implementable by having the driver add a priority 1
``skip`` rule whenever the first rule is created, but it makes the result
depend on hidden rows and changes the meaning of a mirror when its first
rule is added. Users who want it can add the catch-all rule themselves.


Upgrade Impact
--------------

New table only. The extension is loaded only with the OVN driver; it works
when the Northbound schema has ``Mirror_Rule`` (OVN 25.09+). During a
rolling upgrade, old Neutron servers do not know the extension and reject
rule calls; existing mirrors are not affected.


Testing
-------

* Unit tests: API definition, plugin validation, translation to OVN match
  (every attribute combination and the invalid ones), OVN driver calls, DB
  migration and cascade delete, and a Northbound schema without
  ``Mirror_Rule`` (rule creation fails cleanly, lport mirrors still work).
* Functional tests in ovsdbapp for ``mirror_rule_add/del`` against a real
  OVN Northbound schema (skipped when the schema has no ``Mirror_Rule``),
  part of [2]_.
* The ``neutron-tempest-plugin-tap-as-a-service-ovn`` job currently builds
  OVN ``branch-24.03``, which has no lport mirror. A job variant with OVN
  25.09 or later will be added, running:

  * API tests: create, list, show and delete rules; rejected cases (rule on
    a gre mirror, priority 0, a direction the mirror does not copy, port
    range without an L4 protocol, identical rule); rules deleted with their
    tap mirror; the attribute checks of the table above.
  * Scenario tests with a collector VM running ``tcpdump``: a ``skip`` rule
    removes a flow from the collector, a higher priority ``mirror`` rule
    brings it back, and an ``IN`` only rule leaves the outbound traffic
    untouched.

TaaS has no functional test suite today, so the OVN backend is covered end
to end by the tempest job above. If reviewers prefer, a small functional
suite for the OVN driver can be added on top of the neutron OVN functional
base.


Documentation Impact
--------------------

* TaaS user guide: rules, default allow, how to get "only what I list",
  relation with security groups, rules only on lport mirrors.
* TaaS API reference and release note.
* OVS/OVN feature parity table for TaaS, requested at the RFE review.


Work Items
----------

* neutron-lib: ``tap-mirror-rules`` API definition and exceptions.
* ovsdbapp: ``Mirror_Rule`` support [2]_.
* tap-as-a-service: DB, plugin, OVN driver, policies, docs, CI job.
* openstacksdk and python-openstackclient support.
* neutron-tempest-plugin: API and scenario tests.

A working implementation exists and was tested end to end on OVN 26.03
with Neutron 2025.1, including a skip rule and a higher priority mirror
rule overriding it.


References
==========

.. [1] Neutron drivers meeting, 2026-10-02:
   https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-10-02-13.06.log.html
.. [2] ovsdbapp: nb: add support for mirror-rules in mirrors:
   https://review.opendev.org/c/openstack/ovsdbapp/+/950742

* RFE for lport tap mirrors: https://bugs.launchpad.net/neutron/+bug/2168007
* OVN Mirror_Rule: https://github.com/ovn-org/ovn/commit/3dd8f36b85b3e648bc506839aa62e81b21c41e30
* AWS VPC Traffic Mirroring filters:
  https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-filter.html
* K2 Cloud Traffic Mirroring:
  https://docs.k2.cloud/en/services/interconnect/traffic_mirroring.html
