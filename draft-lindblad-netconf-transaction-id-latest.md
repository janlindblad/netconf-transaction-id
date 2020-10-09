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

TODO Abstract

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

> Comment, to be removed    
  Evidence of this feature being demanded by clients is that numerous 
  server implementors have built proprietary and mutually incompatible 
  mechanisms for obtaining a transaction id from a NETCONF server.

RESTCONF, RFC 8040 [RFC8040](https://tools.ietf.org/html/rfc8040), 
defines a mechanism for detecting changes in configuration subtrees 
based on Entity-tags (ETags).  In conjunction with this, RESTCONF 
provides a way to make configuration changes conditional on the server
confiuguration being untouched by others. This mechanism leverages 
RFC 7232 [RFC7232](https://tools.ietf.org/html/rfc7232) 
"Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests".

This document defines similar functionality for NETCONF, 
RFC 6241 [RFC6241](https://tools.ietf.org/html/rfc6241).

> Comment, to be removed    
  The intent behind this specification is to stay close to the 
  RESTCONF E-Tags mechanism so that existiner server functionality
  should be possible to reuse for NETCONF to a very high degree.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Configuration Entity and Timestamp Tagging

When a NETCONF client retrieves the configuration from a NETCONF 
server that implement this specification, it may request that the
configuration is entity or timestamp tagged.  The entity and
timestamp tags are XML attributes added to the retrieved configuration
elements by the server.

The entity and timestamp attributes are guaranteed to change every
time there has been a configuration change at or below the element
bearing the attribute.

Clients request entity and timestamp tags to be added by setting an
attribute on the configuration retrieval request to the value "true".  
To retrieve both etag and timestamp tags, a client might send:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
  <get-config xmlns:ietf-netconf-transaction-id="..."
              ietf-netconf-transaction-id:entag="true"
              ietf-netconf-transaction-id:timestamp="true">
...
~~~

## Entity-Tags Encoding

Entity-tags are opaque base64 encoded strings that represent a 
particular configuration state for a configuration datastore subtree.  
Servers SHOULD ensure that the probability for two different 
configurations having the same Entity-tag value is extremely small.

Implementations are RECOMMENDED to implement entity-tags by computing
a hash over the configuration subtree.

## Timestamps Encoding

Timestamps are ISO 8601 
[ISO8601](https://www.iso.org/standard/70907.html) 
encoded strings in UTC timezone that represent the last time the
element or any of its descendent elements had the configuration 
changed.  Servers SHOULD ensure that the server's clock is reasonably
accurate.

## Behavior

The server SHOULD maintain an entity-tag and timestamp for each YANG 
container and list that represents configuration, but servers are 
allowed to not implement entity-tags and timestamps for some 
such containers and lists.  Entity-tags and timestamps as defined 
in this document MUST NOT be present on non-configuration elements.

For YANG elements where an entity-tag and timestamp is maintained, 
the server MUST return an entity-tag and timestamp when the element is
retrieved using the get-config or get-data operations.  

The element entity-tag and timestamp MUST be updated whenever the 
container or list itself, or any descendant configuration element is
altered, regardless of wheter such a change was initiated over
NETCONF or other means.  The entity-tag MUST NOT be updated due to 
changes in non-configuration elements.  

## Entity-Tag Protocol Usage

The entity-tags and timestamps are conveyed by the server to the client
as XML attributes on container and list instances in the output. The 
attributes' XML namespace MUST be the ietf-netconf-transaction-id 
YANG module's namespace.

A client might send the following get-config request:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
  <get-config xmlns:ietf-netconf-transaction-id="..."
              ietf-netconf-transaction-id:entag="true"
              ietf-netconf-transaction-id:timestamp="true">
    <source>
      <running/>
    </source>
    <filter>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"/>
    </filter>
  </get-config>
</rpc>
~~~

To this, a server implementing ietf-netconf-transaction-id might 
respond:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                xmlns:ietf-netconf-transaction-id="..."
                ietf-netconf-transaction-id:entag="abc12345678"
                ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
      <interface ietf-netconf-transaction-id:entag="def88884321"
                 ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
        <name>GigabitEthernet-0/0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
      <interface ietf-netconf-transaction-id:entag="ghi77775678"
                 ietf-netconf-transaction-id:timestamp="2020-08-12T00:16:11Z">
        <name>GigabitEthernet-0/0/1</name>
        <description>Upward Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
  </data>
</rpc>
~~~

The "entag" attribute values in the example above are meant to 
represent hash values, computed over the configuration of each
element and its descendants.

# Conditional Transactions

Conditional Transactions are useful when a client is interested to
make a configuration change, being sure that the server configuration
has not changed since the client last inspected it.

Even if a client is constantly connected to a device, and even possibly
receiving notification when a server device's configuration changes,
there is always a possibility that a change is introduced by another
party in the time window between when the client last received an 
update about the server's configuration until the server executes a 
configuration change request.

By introducing conditional transactions, this race condition can be 
eliminated efficiently.  If the client provides the transaction-id
it expects the device to have as part of it's configuration change 
request, and the device guarantees to only execute the request in case 
the transaction-id in the request matches that on the server, the race 
condition is removed.

## Behavior

In case any of the entag values provided by the client do not 
match the value on the server, the entire transaction MUST be rejected.

In case any of the timestamp values provided by the client are older 
(reference an prior point in time) than the value on the server, the
entire transaction MUST be rejected.

When a transaction is rejected due to an entag or timestamp value
mismatch, the server must return an rpc-error with 
error-tag data-missing, error-type protocol, 
error-severity error and error-info containing a current-value tag with 
current value of the property that failed the precondition.

> Comment, to be removed    
  Say something about timestamp of transactions that do not really
  make a change (timestamp may or may not change?)

## Updates

When a transaction


## Conditional Transactions Protocol Usage

> Comment, to be removed    
  There are two different proposals here, Attribute- and Leaf- based.
  We have to pick one.

### Attribute-based variant

The entity-tags and timestamps are conveyed by the server to the client
as XML attributes on container and list instances in the output. The 
attributes' XML namespace MUST be the ietf-netconf-transaction-id 
YANG module's namespace.

A client might send the following edit-config request, if the intent 
is to ensure there have been no changes to any interfaces on the 
server since the client last synchronized before making the requested
change.

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
  <edit-config xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
    <target>
      <candidate/>
    </target>
    <test-option>test-then-set</test-option>
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                  ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
        <interface ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
          <name>GigabitEthernet-0/0/1</name>
          <description>Downward Interface</description>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
~~~

If the timestamps sent by the client match the entags in the server, it 
might respond:

~~~
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" 
           message-id="1">
  <ietf-netconf-transaction-id:updates>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                ietf-netconf-transaction-id:entag="xyz555443322"
                ietf-netconf-transaction-id:timestamp="2020-10-05T08:16:47Z">
      <interface ietf-netconf-transaction-id:entag="zzz32132199"
                 ietf-netconf-transaction-id:timestamp="2020-10-05T08:16:47Z">
        <name>GigabitEthernet-0/0/1</name>
      </interface>
    </interfaces>
  </ietf-netconf-transaction-id:updates>
  <ok/>
</rpc-reply>
~~~

If instead one or more of the entags sent by the client do not match 
the entags on the server, it might respond:

~~~
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" 
           message-id="1">
  <rpc-error>
    <error-type>protocol</error-type>
    <error-tag>data-missing</error-tag>
    <error-severity>error</error-severity>
    <error-info>
      <current-value 
        xmlns:ietf-netconf-transaction-id=
        "urn:ietf:params:xml:ns:netconf:transaction-id:1.0">
        yzx76576511
      </current-value>
    </error-info>
  </rpc-error>
</rpc-reply>
~~~

In case the client intended to go through with the transaction 
regardless of any changes to other interface instances, but ensure
no changes have been made in the target interface, it could
send a request without any entag attribute provided for the 
interfaces container.

In this case, the transaction is only aborted in  case there have
been any changes to to GigabitEthernet-0/0/1 interface instance:

~~~
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
  <edit-config xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
    <target>
      <candidate/>
    </target>
    <test-option>test-then-set</test-option>
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface ietf-netconf-transaction-id:entag="def88884321">
          <name>GigabitEthernet-0/0/1</name>
          <description>Downward Interface</description>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
~~~





To this, a server implementing ietf-netconf-transaction-id might 
respond:

~~~
<rpc-reply message-id="1"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"
                ietf-netconf-transaction-id:entag="abc12345678"
                ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
      <interface ietf-netconf-transaction-id:entag="def88884321"
                 ietf-netconf-transaction-id:timestamp="2020-10-01T12:33:50Z">
        <name>GigabitEthernet-0/0/0</name>
        <description>Management Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
      <interface ietf-netconf-transaction-id:entag="ghi77775678"
                 ietf-netconf-transaction-id:timestamp="2020-08-12T00:16:11Z">
        <name>GigabitEthernet-0/0/1</name>
        <description>Upward Interface</description>
        <type>ianaift:ethernetCsmacd</type>
        <enabled>true</enabled>
      </interface>
    </interfaces>
  </data>
</rpc>
~~~

The "entag" attribute values in the example above are meant to 
represent hash values, computed over the configuration of each
element and its descendants.







# Configuration Retrieval with Maximum Depth










+ get last-transaction-id from somewhere Last-Modified ETag
+ return at edit-config time in rpc-reply
+ edit-config parameter If-Unmodified-Since, If-Modified-Since if-match


Appendix A.  NETCONF Error List

~~~
protocol : data-missing

   error-tag:      data-missing
   error-type:     application
   error-severity: error
   error-info:     none
   Description:    Request could not be completed because the relevant
                   data model content does not exist.  For example,
                   a "delete" operation was attempted on
                   data that does not exist.

   error-tag:      data-missing
   error-type:     protocol
   error-severity: error
   error-info:     current-value : current value of the property that 
                                   failed the precondition
   Description:    Request could not be completed because the precondition
                   specified in the request was not met.
~~~

# YANG Modules


~~~ yang
module ietf-netconf-transaction-id {
  namespace 'urn:ietf:params:xml:ns:netconf:transaction-id:1.0';
  prefix ietf-netconf-transaction-id;

  organization
    "IETF NETCONF (Network Configuration) Working Group";

  contact
    "WG Web:   <http://tools.ietf.org/wg/netconf/>
     WG List:  <netconf@ietf.org>

     Author:   Jan Lindblad
               <mailto:jlindbla@cisco.com>";

  description
    "NETCONF Transaction ID aware operations.

     Copyright (c) 2020 IETF Trust and the persons identified as
     the document authors.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Simplified BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (http://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX; see
     the RFC itself for full legal notices.";

  revision 2020-10-01 {
    description
      "Initial revision";
    reference
      "RFC XXXX: Xxxxxxxxx";
  }

  import ietf-netconf {
    prefix nc;
  }

  import ietf-yang-types {
    prefix yang;
  }

  // FIXME: Only relevant for the "leaf-based approach"
  grouping netconf-transaction-id-groping {
    leaf entag {  
      type entag-type;
    }
    leaf timestamp {  
      type yang:date-time;
    }
  }

  container netconf-transaction-id {
    uses netconf-transaction-id-groping;
  }

  typedef wildcard-type {
    // FIXME: Do we need this?
    type string {
      length 1;
      pattern '*';
    }
  }
  typedef entag-type {
    description "Base64 encoded tag value"
    type string {
      pattern '[A-Za-z0-9+/]+';
    }
  }
  typedef entag-or-wildcard-type {
    type union {
      type wildcard-type;
      type entag-type;
    }
  }

  augment /nc:edit-config/nc:input {
    container precondition {
      choice precondition {
        default none;
        leaf none {
          type empty;
        }
        leaf-list if-match-entag {
          type entag-or-wildcard-type;
        }
        leaf-list if-none-match-entag {
          // FIXME: Do we need this?
          type entag-or-wildcard-type;
        }
        leaf if-unmodified-since-timestamp {
          type yang:date-time;
        }
        leaf if-modified-since-timestamp {
          // FIXME: Do we need this?
          type yang:date-time;
        }
      }
    }
  }

  // FIXME: Only relevant for the "leaf-based approach"
  augment /nc:edit-config/nc:output {
    uses netconf-transaction-id-groping;
  }
  // FIXME: Only relevant for the "leaf-based approach"
  augment /ncds:get-config/ncds:output {
    // FIXME: Say not filter aware
    uses netconf-transaction-id-groping;
  }
}
~~~

~~~ yang
module ietf-netconf-nmda-transaction-id {
  namespace 'urn:ietf:params:xml:ns:netconf:nmda:transaction-id:1.0';
  prefix ietf-netconf-nmda-transaction-id;

  organization
    "IETF NETCONF (Network Configuration) Working Group";

  contact
    "WG Web:   <http://tools.ietf.org/wg/netconf/>
     WG List:  <netconf@ietf.org>

     Author:   Jan Lindblad
               <mailto:jlindbla@cisco.com>";

  description
    "NETCONF Transaction ID aware operations for NMDA.

     Copyright (c) 2020 IETF Trust and the persons identified as
     the document authors.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Simplified BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (http://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX; see
     the RFC itself for full legal notices.";

  revision 2020-10-01 {
    description
      "Initial revision";
    reference
      "RFC XXXX: Xxxxxxxxx";
  }

  import ietf-netconf-nmda {
    prefix ncds;
  }

  import ietf-yang-types {
    prefix yang;
  }

  grouping netconf-transaction-id-groping {
    leaf entag {  
      type entag-type;
    }
    leaf timestamp {  
      type yang:date-time;
    }
  }

  // FIXME: Only relevant for the "leaf-based approach"
  container netconf-transaction-id {
    uses netconf-transaction-id-groping;
  }

  typedef wildcard-type {
    type string {
      length 1;
      pattern '*';
    }
  }
  typedef entag-type {
    description "Base64 encoded tag value"
    type string {
      pattern '[A-Za-z0-9+/]+';
    }
  }
  typedef entag-or-wildcard-type {
    type union {
      type wildcard-type;
      type entag-type;
    }
  }

  augment /ncds:edit-data/ncds:input {
    container precondition {
      choice precondition {
        default none;
        leaf none {
          type empty;
        }
        leaf-list if-match-entag {
          type entag-or-wildcard-type;
        }
        leaf-list if-none-match-entag {
          type entag-or-wildcard-type;
        }
        leaf if-unmodified-since-timestamp {
          type yang:date-time;
        }
        leaf if-modified-since-timestamp {
          type yang:date-time;
        }
      }
    }
  }

  // FIXME: Only relevant for the "leaf-based approach"
  augment /ncds:edit-data/ncds:output {
    uses netconf-transaction-id-groping;
  }
  // FIXME: Only relevant for the "leaf-based approach"
  augment /ncds:get-data/ncds:output {
    // FIXME: Say not filter aware
    uses netconf-transaction-id-groping;
  }
}
~~~










# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.


