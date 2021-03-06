---
title: Propeller
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Propeller

Propeller is a Manta component that wreaks havoc, causing components to fail and
otherwise seeing if Manta can stand up to abuse.  This is to attempt to track
down problems in other-than-prod environments before problems arise in prod.

Some things that propeller might be able to do:

1. svcadm disable/enable individual processes.
1. pstop individual processes.
1. Disable/enable "process groups" (i.e. all muskie node processes)
1. Shutdown zones or hosts.
1. Ungraceful host/zone power off.
1. Block/corrupt network traffic to/from a zone.
1. Block/corrupt network traffic to/from a datacenter.
1. Exhaust resources (cpu, memory, disk, etc) in a zone or on a host.
1. Overload processes with requests.
1. Create/delete many things.

# Goals

1. propeller should act randomly, up to some configured "chaos threshold".
   "chaos threshold" is loosely defined at this point.  It may mean some number
   of outstanding actions, or each action may have a "weight", and the threshold
   is based on summed weights (as an example, disabling a muskie process is
   much lighter than blocking network traffic in/out of a DC).
1. propeller should be able to undo any action that it performs.
1. propeller should stop performing actions if Manta was unable to recover
   from the current set of actions (this may be "hard").
1. A detailed record should be kept on what, exactly, propeller did and at what
   time.  This record should be easy for an operator to view.
1. propeller should make current outstanding actions easy to discover and
   reverse.

# High level design

A "component" is something that can be messed with in Manta.  It could be a
network, host, zone, or process.

An "action" is composed of an action to perform along with optional cleanup.
For example, "rebootZone" is an action that would reboot a zone.  It doesn't
require a cleanup step.  Another could be "corruptZoneTraffic" that would set up
an ipd rule to corrupt some percentage of traffic.  The cleanup step would
remove the ipd rule.

Since there will be many disparate actions, it'll follow a plugin model where an
action works on one "component".  When propeller starts for the first time it
will gather all the datacenter "components" with as much information as possible
about the "component".  Propeller will choose an "action" and a corresponding
"component", record that it is going to perform "action" on "component", and
execute.  It will do that up to the configured "chaos threshold".  After some
time period it will undo the set of "actions".  Once all actions have been
undone, propeller will monitor whether Manta has recovered.  It will not perform
a new set of actions until it has.  It will alarm if Manta takes "too long" to
recover.

# Implementation plan

1. Propeller repo **(done)**.
1. Set up deployable repo **(done)**.
1. Generate components **(done)**.
1. Common for x-dc sdc calls (needs to call at least sapi, vmapi, cnapi)
   **(done)**.
1. Common to handle running commands on hosts and in zones **(done)**.
1. Propeller daemon with one or two actions (rebooting zones) **(done)**.
1. Persist actions to something on disk (LevelDB?), to zfs dataset.
1. Tools to query outstanding and historical action sets.
1. Tools to undo current actions.
1. Implement lots of actions.
1. Cause some scars!

# Questions

1. Should it also check to see that the *component* came back up?  Is the
   process serving requests, for example?  If so, there should be a "check"
   function along with the "action" and "cleanup".
