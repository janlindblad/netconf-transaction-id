---
title: "Transaction ID Mechanism for NETCONF"
abbrev: "NCTID"
docname: draft-lindblad-netconf-transaction-id-latest
category: std

ipr: trust200902
area: General
workgroup: NETCONF
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Lindblad
    name: Jan Lindblad
    organization: Cisco Systems
    email: jlindbla@cisco.com

normative:
  RFC2119:

informative:



--- abstract

NETCONF clients and servers often need to have a synchronized view of
the server's configuration data stores.  The volume of configuration 
data in a server may be very large, while data store changes typically
are small when observed at typical client resynchronization intervals.

Rereading the entire data store and analyzing the response for changes
is an inefficient mechanism for synchronization.  This document 
specifies an extension to NETCONF that allows clients and servers to
keep synchronized with a much smaller data exchange and without any
need for servers to store information about the clients.

--- middle

# Introduction

When a NETCONF client connects with a NETCONF server, a frequently 
occurring use case is for the client to find out if the configuration 
has changed since it was last connected.  Such changes could occur for 
example if another NETCONF client has made changes, or another system 
or operator made changes through other means than NETCONF.

One way of detecting a change for a client would be to 
retrieve the entire configuration from the server, then compare 
the result with a previously stored copy at the client side.  This 
approach is not popular with most NETCONF users, however, since it 
would often be very expensive in terms of communications and 
computation cost.

Furthermore, even if the configuration is reported to be unchanged, 
that will not guarantee that the configuration remains unchanged 
when a client sends a subsequent change request, which arrives soon 
thereafter.

Evidence of a transaction-id feature being demanded by clients is that 
several server implementors have built proprietary and mutually 
incompatible mechanisms for obtaining a transaction id from a NETCONF 
server.

RESTCONF, RFC 8040 [RFC8040](https://tools.ietf.org/html/rfc8040), 
defines a mechanism for detecting changes in configuration subtrees 
based on Entity-tags (ETags).  In conjunction with this, RESTCONF 
provides a way to make configuration changes conditional on the server
confiuguration being untouched by others.  This mechanism leverages 
RFC 7232 [RFC7232](https://tools.ietf.org/html/rfc7232) 
"Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests".

This document defines similar functionality for NETCONF, 
RFC 6241 [RFC6241](https://tools.ietf.org/html/rfc6241).

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# NETCONF Extension

This document describes a NETCONF extension which modifies the 
behavior of get-config, get-data, edit-config and edit-data such
that clients are able to conditionally retrieve and update the 
configuration in a NETCONF server.  NETCONF servers that support 
this extension MUST announce the capability "FIXME".

## ETag attribute

Central to the configuration retrieval and update mechanisms described 
in the following sections is a meta data XML attribute called "etag".  
Servers MUST maintain an etag value for each configuration datastore 
they implement.  Servers SHOULD maintain etag values for YANG 
containers that hold configuration for different subsystems.  Servers 
MAY maintain etag values for any YANG container or list element they 
implement. 

The etag attribute values are opaque UTF-8 strings chosen freely by 
the server, except the etag string must not contain space, backslash 
or double quotes. The point of this restriction is to make it easy to 
reuse implementations that adhere to section 2.3.1 in RFC 7232 
[RFC7232](https://tools.ietf.org/html/rfc7232).  The probability that 
an etag value used historically is used again by this server SHOULD be 
made very low.

The etag attribute is defined in the namespace "FIXME".

## ETag value changes

When a NETCONF client retrieves the configuration from a NETCONF 
server that implements this specification, it MAY request that the
configuration is entity tagged.  The entity tags are attributes 
added to some of the retrieved configuration elements by the server.  
These elements are collectively called the "versioned elements".

The server returning the entity-tag (etag) attributes for the 
versioned elements MUST ensure the etag values are changed every time 
there has been a configuration change at or below the element bearing 
the attribute.  This means any update of a config true element will 
result in new etag values for all ancestor versioned elements, up to 
and including the datastore root itself.

The server MUST NOT change the etag values due to updates in config 
true data in other parts of the YANG data tree or due to changes in
config false data.

These rules are chosen to be consistent with the ETag mechanism in 
RESTCONF, RFC 8040 [RFC8040](https://tools.ietf.org/html/rfc8040), 
specifically sections 3.4.1.2 and 3.5.2.

# Configuration Retreival

Clients MAY request the server to return etag attribute values in the 
response by adding one or more etag attributes in get-config or 
get-data requests.  

The etag attribute may be added directly on the get-config or get-data 
requests, in which case it pertains to the entire datastore.  A client
may also add etag attributes to zero or more individual elements in 
the get-config or get-data request, in which case it pertains to the
subtree rooted at that element.

For each element that the client requests etag attributes, the server 
MUST return etags for all versioned elements at or below that point 
that are part of the server's respone.

If the client is requesting an etag value for an element that is not 
among the server's versioned elements, then the server MUST return the 
etag attribute on the closest ancestor that is a versioned element, 
and all children of that ancestor.  The datastore root is always a 
versioned element.

## Initial Configuration Response

When the client adds etag attributes to a get-config or get-data 
request, it should specify the last known etag values it has seen for 
the elements it is asking about.  Initially, the client will not know 
any etag value and should use "?".  

To retrieve etag attributes across the entire NETCONF server 
configuration, a client might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:txid="FIXME">
  <get-config txid:etag="?"/>
</rpc>
~~~

To retrieve etag attributes for a specific interface using an xpath 
filter, a client might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:txid="FIXME">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="xpath"
      xmlns:if="urn:ietf:params:xml:ns:yang:ietf-interfaces"
      select=
        "/if:interfaces/if:interface[if:name='GigabitEthernet-0/0']"
      txid:etag="?"/>
  </get-config>
</rpc>
~~~

To retrieve etag attributes for "ietf-interfaces", but not for "nacm",
a client might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:txid="FIXME">
  <get-config>
    <source>
      <running/>
    </source>
    <filter>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
        txid:etag="?"/>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm"/>
    </filter>
  </get-config>
</rpc>
~~~

When a NETCONF server receives a get-config or get-data request 
containing txid:etag attributes with the value "?", it MUST return 
etag attributes for all versioned elements below this point included 
in the reply.

If the server considers the container "interfaces" and the list 
"interface" elements to be versioned elements, the server's response 
to the request above might look like:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <data txid:etag="def88884321">
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                txid:etag="def88884321">
      <interface txid:etag="def88884321">
        <name>GigabitEthernet-0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
      <interface txid:etag="abc12345678">
        <name>GigabitEthernet-0/1</name>
        <description>Upward Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
    <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm"/>
      <groups>
        <group>
          <name>admin</name>
          <user-name>sakura</user-name>
          <user-name>joe</user-name>
        </group>
      </groups>
    </nacm>
  </data>
</rpc>
~~~

## Configuration Response Pruning

A NETCONF client that already knows some etag values MAY request that
the configuration retrieval request is pruned with respect to the 
client's prior knowledge.

To retrieve only changes for "ietf-interfaces" that do not have the 
last known transaction-id "abc12345678", but include the entire 
configuration for "nacm", regardless of etags, a client might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:txid="FIXME">
  <get-config>
    <source>
      <running/>
    </source>
    <filter>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
        txid:etag="abc12345678"/>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm"/>
    </filter>
  </get-config>
</rpc>
~~~

When a NETCONF server receives a get-config or get-data request 
containing a client specified etag attribute, there are several 
different cases:

* The element is not a versioned element, i.e. the server does not 
maintain an etag value for this element.  In this case, the server 
MUST look up the closest ancestor that is a versioned element, and 
proceed as if the client had specified the etag value for that 
element.

* The element is a versioned element, and the client specified etag 
attribute value is different than the server's etag value for this
element, then the server MUST return the contents as it would normally.

* The element is a versioned element, and the client specified etag 
attribute value does match the server's etag value, then server MUST 
return the element decorated with an etag attribute with the value "=".

* The element is a versioned element, the client specified etag 
attribute value does match the server's etag value, and the element is 
a container.  In this case the server MUST NOT return any of the 
children of the container.

* The element is a versioned element, the client specified etag 
attribute value does match the server's etag value, and the element is 
a list entry.  In this case the server MUST return the keys of the 
list entry, and MUST NOT return any other children of the list entry.

For example, assuming the NETCONF server configuration is the same as 
in the previous rpc-reply example, the server's response to request 
above might look like:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <data txid:etag="def88884321">
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                txid:etag="def88884321">
      <interface txid:etag="def88884321">
        <name>GigabitEthernet-0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
      <interface txid:etag="=">
        <name>GigabitEthernet-0/1</name>
      </interface>
    </interfaces>
    <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm"/>
      <groups>
        <group>
          <name>admin</name>
          <user-name>sakura</user-name>
          <user-name>joe</user-name>
        </group>
      </groups>
    </nacm>
  </data>
</rpc>
~~~

# Configuration Update

Whenever the configuration on a server changes for any reason, the 
server MUST update the etag values for all versioned elements that 
have children that changed.

How the server selects a new etag value or values to use for changed
elements is described in section [ETag attribute](#etag-attribute).

When a NETCONF client sends an edit-config or edit-data request, the 
server MUST change the etag value of all versioned elements that have 
children that were mentioned in those edit-config or edit-data 
payloads regardless of whether an actual value change took place or 
not.  The server MUST return the etag value assigned on the XML ok 
tag in the rpc-reply.

Note in particular that the server MUST update the etag value for 
elements that change as a result of the edit-config or edit-data, but 
are not explicitly part of the edit-config or edit-data payload, such
as dependent data under YANG 
[RFC7950](https://tools.ietf.org/html/rfc7950) when- or 
choice-statements.

For example, if a client wishes to update the interface description
for interface "GigabitEthernet-0/1" to "Downward Interface", it might 
send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:txid="FIXME">
  <edit-config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <target>
      <candidate/>
    </target>
    <test-option>test-then-set</test-option>
    <config>
      <interfaces 
        xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>GigabitEthernet-0/1</name>
          <description>Downward Interface</description>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
~~~

The server would update the description leaf in the candidate 
datastore, and return an rpc-reply as follows:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <ok txid:etag="ghi55550101"/>
</rpc-reply>
~~~

A subsequent get-config request for "ietf-interfaces", with 
txid:etag="?" might then return:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <data txid:etag="ghi55550101">
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                txid:etag="ghi55550101">
      <interface txid:etag="def88884321">
        <name>GigabitEthernet-0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
      <interface txid:etag="ghi55550101">
        <name>GigabitEthernet-0/1</name>
        <description>Downward Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
  </data>
</rpc>
~~~

In case the server at this point received a configuration change from 
another source, such as a CLI operator, adding an MTU value for the 
interface "GigabitEthernet-0/0", a subsequent get-config request for 
"ietf-interfaces", with txid:etag="?" might then return:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <data txid:etag="cli22223333">
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                txid:etag="cli22223333">
      <interface txid:etag="cli22223333">
        <name>GigabitEthernet-0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
        <mtu>768</mtu>
      </interface>
      <interface txid:etag="ghi55550101">
        <name>GigabitEthernet-0/1</name>
        <description>Downward Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
  </data>
</rpc>
~~~

## Conditional Configuration Update

Conditional Transactions are useful when a client is interested to
make a configuration change, being sure that the server configuration
has not changed since the client last inspected it.

By supplying the latest etag values known to the client
in its change requests (edit-config etc.), it can request the server 
to reject the transaction in case any changes have occurred at the 
server that the client is not yet aware of.

Unless datastore global locks are taken for potentially long times,
even if a client is constantly connected to a device, and even possibly
receiving notifications when a server device's configuration changes,
there is always a possibility that a change is introduced by another
party in the time window between when the client last received an 
update about the server's configuration until the server executes a 
configuration change request.  By leveraging conditional transactions, 
this race condition can be eliminated efficiently.  

When a NETCONF client sends an edit-config or edit-data request to a
NETCONF server that implements this specification, the client MAY 
specify expected etag values on the versioned elements touched by the
transaction.

If such an etag value differs from the etag value stored on the 
server, the server MUST reject the transaction.

If the server rejects the transaction because the configuration etag
value differs from the client's expectation, ther server MUST return
an rpc-error with the following values:

~~~
   error-tag:      operation-failed
   error-type:     protocol
   error-severity: error
~~~

Additionally, the error-info tag MUST contain an sx:structure
etag-value-mismatch-error-info, with mismatch-path set to the 
instance identifier value identifying one of the versioned elements 
that had an etag value mismatch, and mismatch-etag-value set to
the server's current value of the etag attribute for that versioned
element.

For example, if a client wishes to delete the interface 
"GigabitEthernet-0/1" if and only if its configuration has not been
altered since this client last synchronized its configuration with the
server (at which point it received the etag "ghi55550101"), 
regardless of any possible changes to other interfaces, it might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" 
     xmlns:txid="FIXME">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <test-option>test-then-set</test-option>
    <config>
      <interfaces 
        xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface nc:operation="delete" 
                   txid:etag="ghi55550101">
          <name>GigabitEthernet-0/1</name>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
~~~

If interface "GigabitEthernet-0/1" has the etag value "ghi55550101",
as expected by the client, the transaction goes through, and the 
server responds something like:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <ok txid:etag="xyz77775511"/>
</rpc-reply>
~~~

A subsequent get-config request for "ietf-interfaces", with 
txid:etag="?" might then return:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
           xmlns:txid="FIXME">
  <data txid:etag="xyz77775511">
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                txid:etag="xyz77775511">
      <interface txid:etag="def88884321">
        <name>GigabitEthernet-0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
  </data>
</rpc-reply>
~~~

In case interface "GigabitEthernet-0/1" did not have the expected etag 
value "ghi55550101", the server rejects the transaction, and might 
send:

~~~
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" 
           xmlns:if="urn:ietf:params:xml:ns:yang:ietf-interfaces"
           xmlns:txid="FIXME">
           message-id="1">
  <rpc-error>
    <error-type>protocol</error-type>
    <error-tag>operation-failed</error-tag>
    <error-severity>error</error-severity>
    <error-info>
      <txid:etag-value-mismatch-error-info>
        <txid:mismatch-path>
          /if:interfaces/if:interface[if:name="GigabitEthernet-0/0"]
        </txid:mismatch-path>
        <txid:mismatch-etag-value>
          cli22223333
        </txid:mismatch-etag-value>
      </txid:etag-value-mismatch-error-info>
    </error-info>
  </rpc-error>
</rpc-reply>
~~~

# YANG Modules

~~~ yang
module ietf-netconf-transaction-id {
  yang-version 1.1;
  namespace 
    'urn:ietf:params:xml:ns:yang:ietf-netconf-transaction-id';
  prefix txid;

  import ietf-netconf {
    prefix nc;
  }

  import ietf-netconf-nmda {
    prefix ncds;
  }

  import ietf-yang-structure-ext {
    prefix sx;
  }

  organization
    "IETF NETCONF (Network Configuration) Working Group";

  contact
    "WG Web:   <http://tools.ietf.org/wg/netconf/>
     WG List:  <netconf@ietf.org>

     Author:   Jan Lindblad
               <mailto:jlindbla@cisco.com>";

  description
    "NETCONF Transaction ID aware operations for NMDA.

     Copyright (c) 2021 IETF Trust and the persons identified as
     the document authors.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Simplified BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (http://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX; see
     the RFC itself for full legal notices.";

  revision 2021-11-01 {
    description
      "Initial revision";
    reference
      "RFC XXXX: Xxxxxxxxx";
  }

  typedef etag-t {
    type string {
      pattern ".* .*" {
        modifier invert-match;
      }
      pattern ".*\".*" {
        modifier invert-match;
      }
      pattern ".*\\.*" {
        modifier invert-match;
      }
    }
    description 
      "Unique Entity-tag value representing a specific transaction.
       Could be any string that does not contain spaces, double 
       quotes or backslash.  The values \"?\" and \"=\" have special
       meaning.";
  }

  sx:structure etag-value-mismatch-error-info {
    container etag-value-mismatch-error-info {
      description
         "This error is returned by a NETCONF server when a client
          sends a configuration change request, with the additonal
          condition that the server aborts the transaction if the
          server's configuration has changed from what the client
          expects, and the configuration is found not to actually
          not match the client's expectation.";
      leaf mismatch-path {
        type instance-identifier;
        description
          "Indicates the YANG path to the element with a mismatching
           etag value.";
      }
      leaf mismatch-etag-value {
        type etag-t;
        description
          "Indicates server's value of the etag attribute for one
           mismatching element.";
      }
    }
  }
}
~~~

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


# Changes

## Major changes in -01 since -00

* Updated the text on numerous points in order to answer questions 
that appeared on the mailing list.

* Renamed entag attribute to etag and namespace to txid. 
Harmonized/slightly adjusted value space with RFC7232 and RFC8040.

* Removed all text discussing etag values provided by the client 
(although this is still an interesting idea, if you ask the author)

* Clarified the etag attribute mechanism, especially when it comes to
matching against non-versioned elements, its cascading upwards in the 
tree and secondary effects from when- and choice-statements.

* Added a mechanism for returning the server assigned etag value in
get-config and get-data.

* Removed many comments about open questions.




--- back

# Acknowledgments
{:numbered="false"}

The author wishes to thank Benoît Claise for making this work happen,
and the following individuals, who all provided helpful comments:
Per Andersson, Kent Watsen, Andy Bierman, Robert Wilton, Qiufang Ma.