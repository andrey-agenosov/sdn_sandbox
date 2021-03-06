## Common terms

* Flow entry in a switch flow table consists of:
    - match fields
    - a priority for matching
    - counters
    - a set of instructions to apply to matching packet.

* Instructions associated with flow entry contain either _actions_ or _modify pipeline processing_.

    Examples of actions: packet forwarding, packet modification, group table processing.

    Pipeline processing instructions allow packets to be sent to subsequent tables for further processing and allow metadata to be communicated between tables.

    The same as above in another words with more details: an **instruction either modifies pipeline processing** (such as directing the packet to another flow table), or **contains a set of actions to add to the action set**, or **contains a list of actions to apply immediatly to the packet**.

* During pipeline processing a set of values (_pipeline fields_) can be attached to the packet:

    - the ingress port (a property of the packet throughout the OPF pipeline)

    - the metadata value

    - the Tunnel-ID (a packet may have this extra pipeline field when it's associated with a *logical OPF port*)

* Actions may be _accumulated in the Action Set_ of the packet or _applied immediately_ to the packet.

* A switch element that can measure and control the rate of packets is called a **meter**. It _triggers_ a meter **band** if the packet rate (or byte rate) passing through the meter exceeds a predefined threshold.

    Each meter may have several bands. Band specifies a target rate and a way packets should be processed if that rate is exceeded.

    Example is a rate limiter whose band drops the packet.

    Meters are attached directly to flow entries (as opposed to queues attached to ports). There's a separate meter table.

* Pipeline processing happens in two stages: *ingress processing and egress processing*. Egress processing happens after the determination of the output port and happens in the context of this port.


## Notes related to OPF channel & connection management

### Message types supported by OPF

* _controller-to-switch_
* _async_ (initiated by the switch, used to inform the controller about changes)
* _symmetric_ (can be initiated by both sides). **Experimenter** messages fall into this category.

### Message Handling

* In the absence of barrier messages, switches may reorder messages to maximize performance => **controllers shouldn't depend on a specific processing order**... If 2 messages from the controller depend on each other:
    - they must either *be separated by a barrier message* ([here](https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-sdn-openflow-messages-barrier-overview.html) is a very useful explanation)
    - or *be put in the same ordered bundle*.

### Connection setup & maintenance

* After connection setup is done **one of the 1-st things the controller should do** is to get the *Datapath ID of the switch* (as a reply to *OFPT_FEATURES_REQUEST* message).

* A switch management protocol such as *OF-CONFIG* is recommended to be used for configuring and managing security credentials.

### Multiple controllers

* Controller can [specify which types of async messages from switch are sent](https://www.opennetworking.org/wp-content/uploads/2014/10/openflow-switch-v1.5.1.pdf#subsubsection.7.3.10) over its channel in order to **control which message types can be enabled of filtered**.

* As a key takeaways from [CAP for Networks](https://people.eecs.berkeley.edu/~alig/papers/cap-for-networks.pdf):

    - controllers typically communicate through *out-of-band management network* to coordinate among themselves... so there could be a situation where **the controllers are partitioned from each other** while the data network bacame connected... as a result **network policies may be violated**.

    - _hybrid approaches_ (i.e. where controllers revert to *in-band control* where the out-of-band control network is partitioned) provide comparable simplicity (to out-of-band) while providing greater resilency.


## Notes related to the details of OPF Switch Protocol 

* Most structures are packed with padding and 8-byte aligned. All messages are sent in Big-endian.
* Message header contains:
    - version
    - type of message
    - total length of the message (including the header)
    - transaction id (replies use the same value as was in the request)

### About switch ports

* In general, the port config bits are set by the controller and not changed by the switch.

* The switch sends an *OFPT_PORT_STATUS message* to notify the controller of the change when:

    - the port config bits are changed by the switch through another administrative interface.

* Port addition, modification or removal never changes the content of the flow tables. **When a port is deleted it's left to the controller to clean up any flow entries** (or group entries) referencing that port.


### About switch queues

* A switch can optionally have one or more queues (to provide QoS) attached to a specific output port. Those queues
can be used to schedule packets exiting the datapath on that output port.

* Packets are directed to one of the queues based on the *packet output port* and the *packet queue id*.

* Queue processing happens logically after all pipeline processing.

* There is the **special action type** to set queue id when outputting a packet to a port: *OFPAT_SET_QUEUE*


### Flow Match

* OXM TLV - *OpenFlow Extensible Match type-length-value* format

* The payload of the OpenFlow match is a set of OXM Flow match fields.

* The first 4 bytes of an OXM TLV (*flow match field*) are its header (**the combination of oxm_class and oxm_field normally designates a protocol header field**, but it can also refer to a packet pipeline field):
    - *oxm_class*
    - *oxm_field*
    - *oxm_hasmask* - defines if the OXM TLV contains a bitmask
    - *oxm_length* -  the length of the OXM TLV payload in bytes, i.e. everything that follows the 4 bytes OXM TLV header

* If oxm_hasmask is 0, the OXM TLV’s body contains a value for the field, called *oxm_value*. The **OXM TLV match matches only packets in which the corresponding field equals oxm_value**.

* If oxm_hasmask is 1, *each 1-bit in oxm_mask constrains the OXM TLV to match only packets in which the corresponding bit of the field equals the corresponding bit in oxm_value*.

* Most match fields have prerequisites (another match field type and match field value that this match field depends on) - see [Header Match Fields](https://www.opennetworking.org/wp-content/uploads/2014/10/openflow-switch-v1.5.1.pdf#paragraph.7.2.3.8)

* Note for how switches deal with matches: **if the match in a flow mod message specifies a field but fails to specify its prerequisites, the switch must return an error** with *OFPET_BAD_MATCH* type and *OPFBMC_BAD_PREREQ* code (for example, specifies an IPv4 addr without matching the EtherType to 0x800).

* The controller can query the switch about which match fields are supported in each flow table.

* There're 2 types of match fields:
    - *header match fields* - matching values extracted from the packet headers
    - *pipeline match fields* - matching values attached to the packet for pipeline processing


### Flow Stats

* OPF *Extensible Stat format* follow the same convention as OXM format.

* Each flow table of the switch must support the [required stat fields](https://www.opennetworking.org/wp-content/uploads/2014/10/openflow-switch-v1.5.1.pdf#paragraph.7.2.4.4):

    - OXS_OF_DURATION
    - OXS_OF_IDLE_TIME
    - OXS_OF_FLOW_COUNT
    - OXS_OF_PACKET_COUNT
    - OXS_OF_BYTE_COUNT


### Flow Instructions

* Instructions (associated with the flow entry) are executed when a packet matches the entry.

* For the *Apply-Actions* instruction, the *actions field is treated as a list*. For the *Write-Actions* instruction, the *actions field is treated as a set*.

* *STAT_TRIGGER* instruction type allows the controller to receive a trigger for one or many thresholds, i.e. *when one of the stat field values of the flow entry crosses threshold*.


### Controller-to-Switch messages

* _Handshake_: the controller requests switch features, and the switch must reply with *OFPT_FEATURES_REPLY* message. The reply contains:
    - *datapath_id* - lower 48-bits are for a MAC address, the upper 16-bits are implementer-defined (for example, it could be VLAN ID to distinguish multiple virtual switch instances on a single physical switch)
    - the maximum number of packets the switch can buffer (when sending packets to the controller using *packet-in* messages)
    - number of tables supported by the switch, the type of connection (main or auxiliary)
    - bitmap which defines the switch capabilities... the *OFPC_PORT_BLOCKED* bit isn't set => the controller should prevent packet loops.

* The controller is able to *set and query configuration parameters in the switch*:
    - bitmap with combination of flags which indicate how IP fragments should be treated by the switch
    - the number of bytes of each packet sent to the controller by the switch pipeline.

* Flow table of a switch has a *dynamic configuration* which can be controlled with the [*OFPT_TABLE_MOD*](https://www.opennetworking.org/wp-content/uploads/2014/10/openflow-switch-v1.5.1.pdf#paragraph.7.3.4.1) message.

    Flow table dynamic configuration defines:
    - if a flow table is authorized to **evict flows** (the OFPTC_EVICTION bit is set => switch can evict flow tables)
    - if switch must generate **vacancy events** (means that flow table has or hasn't a space for new flows).

    Parameters for vacancy events can be controlled by specifying the OFPTMPT_VACANCY property (list of properties is a separate field inside a table modification message):
    - for instance when the remaining space in the flow table decreases to less than a specified threshold, a **vacancy down event must be generated** to the controller (if vacancy down events are *enabled*).

* For the *flow mod message*:
    - the *buffer_id* refers to a packet buffered at the switch and sent to the controller by a *packet-in* message (if the buffer_id is valid, flow-mod removes the corresponding packet from the buffer and processes it through the OpenFlow pipeline after the flow is inserted, starting at the first flow table)
    - the *flags* field could be used to control whether the *Send flow removed message* is to be sent by the switch **when flow expires or is deleted**.
    - the flags of the flow entry are **not changed on flow modify** (but the *OFPFF_RESET_COUNTS* could be used and switch must honor this flag).
    - **(match, priority) is an unique identifier of the flow entry in the table**

* About *meter-modification messages*:
    - the OPF protocol defines some *virtual* meters which can't be associated with flows (for slow datapath, for controller connection)
    - one of the fields is *the list of bands*, and if the current rate of packets exceeds the rate of multiple bands, the band with the highest configured rate is used
    - the *rate* field of the band indicates **the rate value above which the corresponding band may apply to packets**. The rate value is in kbits/sec (unless the flags field includes OFPMF_PKTPS)

#### Multipart messages

* Multipart messages are used to request statistics or to set or retrieve state information from the switch.

    The type field of such messages specifies the kind of information being passed (both in a request and in a response) and determines how the body field is to be interpreted.

    If a controller receives a sequence of multipart reply messages without a message with OFPMPF_REPLY_MORE flag as zero, the controller must discard the whole reply (after a controller defined amount of time greater than 1 second from the last message).

* A controller can request the capabilities of currently configured flow tables or request another configuration of flow tables via *OFPMP_TABLE_FEATURES* message.

    If the request body isn't empty the controller can enable, disable or modify flow tables and also replace the full pipeline. For instance, a flow table can be removed from the pipeline.

    The command field inside the body of the table features request determines the operation to be performed:
    - replace full pipeline
    - modify flow table capabilities
    - enable/disable flow table in pipeline.

    The maximum number of flow entries that can be inserted into specific flow table is the read-only field and returned by the switch.

    The properties field describes misc capabilities of the flow table. Almost each item in the list of properties determines the capability either for regular flow entry or for the table-miss flow entry.

    (!) Fields that can be used as match fields are determined by the OFPTFPT_MATCH property. Fields  not listed in this property can not be used in the table as a match field... unless they are used as a prerequisite.

* A controller can **track changes to the flow tables**. It makes sense in a multi-controller deployment - a controller will get events for any change made to the flow tables by other controllers. Such events are generated by **flow monitors**. A flow monitor configuration is managed by the *OFPMP_FLOW_MONITOR* multipart message type.

    Flow monitor matches a subset of flow entries in the flow table.

    Flow monitor is specific for a particular controller connection within a switch.

    Switch must send the OFPMP_FLOW_MONITOR reply as a notification to the controller if a change to a flow table matches some flow monitor.

    A controller must take into account a flow monitoring pause (due to overflowing the buffer space within the switch). After receiving the OFPME_PAUSED message the controller's view of the flow table is incomplete.

### Asynchronous messages

* Controller can set and query the *async* messages which it wants to receive **via a given OPF channel**.
