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

> Internal comment    
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

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Configuration Tagging

> Internal comment    
  This is based on RFC 8040. Is there a way to avoid duplication?

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
allowed to not implement the entity-tags and timestamps for all 
such containers and lists.  Entity-tags and timestamps as defined 
in this document MUST NOT be present on non-configuration elements.

For YANG elements where an entity-tag and timestamp is maintained, 
the server MUST return an entity-tag and timestamp when the element is
retrieved using the get-config, edit-config or edit-data operations.  

The element entity-tag and timestamp MUST be updated whenever the 
container or list itself or any descendant configuration element is
altered.  The entity-tag MUST NOT be updated due to changes in 
non-configuration elements.

## Entity-Tag Protocol Usage  (variant #1)

> Internal comment    
  There are two different proposals here. We have to pick one.




# Conditional Transactions

The "ETag" header field can be used by a RESTCONF client in
subsequent requests, within the "If-Match" and "If-None-Match" header
fields.


> Internal comment        
                        Currently we have two different approaches to solving
                        this.  We have to pick one.
                        This is the "attribute-based approach"






   This entity-tag is only affected by configuration data resources and
   MUST NOT be updated for changes to non-configuration data.  If the
   RESTCONF server is co-located with a NETCONF server, then the
   entity-tag for a configuration data resource MUST represent the
   instance within the "running" datastore.






3.4.1.2.  Entity-Tag

   The server MUST maintain a unique opaque entity-tag for the datastore
   resource and MUST return it in the "ETag" (Section 2.3 of 
   [RFC7232](https://tools.ietf.org/html/rfc7232))
   header in the response for a retrieval request.  The client MAY use
   an "If-Match" header in edit operation requests to cause the server
   to reject the request if the resource entity-tag does not match the
   specified value.

   The server MUST maintain an entity-tag for the top-level
   {+restconf}/data resource.  This entity-tag is only affected by
   configuration data resources and MUST NOT be updated for changes to
   non-configuration data.  Entity-tags for data resources are discussed
   in Section 3.5.  Note that each representation (e.g., XML vs. JSON)
   requires a different entity-tag.

   If the RESTCONF server is co-located with a NETCONF server, then this
   entity-tag MUST be for the "running" datastore.  Note that it is
   possible that other protocols can cause the entity-tag to be updated.
   Such mechanisms are out of scope for this document.

3.4.1.3.  Update Procedure

   Changes to configuration data resources affect the timestamp and
   entity-tag for that resource, any ancestor data resources, and the
   datastore resource.

   For example, an edit to disable an interface might be done by setting
   the leaf "/interfaces/interface/enabled" to "false".  The "enabled"
   data node and its ancestors (one "interface" list instance, and the
   "interfaces" container) are considered to be changed.  The datastore
   is considered to be changed when any top-level configuration data
   node is changed (e.g., "interfaces").




   The "Last-Modified" header field can be used by a RESTCONF client in
   subsequent requests, within the "If-Modified-Since" and
   "If-Unmodified-Since" header fields.

   If maintained, the resource timestamp MUST be set to the current time
   whenever the resource or any configuration resource within the
   resource is altered.  If not maintained, then the resource timestamp
   for the datastore MUST be used instead.  If the RESTCONF server is
   co-located with a NETCONF server, then the last-modified timestamp
   for a configuration data resource MUST represent the instance within
   the "running" datastore.

   This timestamp is only affected by configuration data resources and
   MUST NOT be updated for changes to non-configuration data.

3.5.2.  Entity-Tag

   For configuration data resources, the server SHOULD maintain a
   resource entity-tag for each resource and return the "ETag" header
   field when it is retrieved as the target resource with the GET or
   HEAD methods.  If maintained, the resource entity-tag MUST be updated
   whenever the resource or any configuration resource within the
   resource is altered.  If not maintained, then the resource entity-tag
   for the datastore MUST be used instead.

   The "ETag" header field can be used by a RESTCONF client in
   subsequent requests, within the "If-Match" and "If-None-Match" header
   fields.







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


