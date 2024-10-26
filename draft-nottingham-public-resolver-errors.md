---
title: Extensions for DNS Public Resolvers
abbrev:
docname: draft-nottingham-public-resolver-errors-latest
date:
category: std

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, tocindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
 -
    ins: M. Nottingham
    name: Mark Nottingham
    organization: Cloudflare
    postal:
      - Prahran
    country: Australia
    email: mnot@mnot.net
    uri: https://www.mnot.net/


--- abstract



--- middle


# Introduction

{{!I-D.ietf-dnsop-structured-dns-error}} introduces structured error data for DNS responses that have been filtered. This draft suggests additions to that mechanism.

These additions are informed by specific concerns that Web browsers have about providing information about DNS filtering events to end users. In particular, they are intended to address the risks associated with trusting information inserted by DNS resolvers into responses. Presenting information sourced from unauthenticated network elements to end users opens a variety of attacks. Given the variety of network deployments on the Internet, such information needs to be considered as attacker-controlled.

This proposal mitigates these risks by minimising the amount and type of information carried into the DNS response to a "DNS Resolver Operator ID" and a "Filtering Incident ID." Neither is presented to end users: instead, they can be used to obtain (using HTTPS) a document that carries details of the specific filtering incident, for presentation to end users.

This mechanism is not intended to scale to large numbers of DNS operators. Instead, it is expected that in typical use, the DNS Resolver Operator ID will be used to selectively present information from DNS resolvers operators that clients deem to be serving a public good role (e.g., publicly available open resolvers), to aid those parties in serving the public interest by making their operation more transparent.

## Notational Conventions

{::boilerplate bcp14-tagged}

# DNS Resolver Operator ID {#op-id}

A DNS Resolver Operator ID is a short, textual string that uniquely identifies the operator of a DNS resolver. It is carried in the EXTRA-TEXT field of the Extended DNS Error with the JSON field name "ro". For example:

~~~ json
{
  "ro": "exampleResolver"
}
~~~

The value of the "ro" field MUST be registered in the DNS Resolver Operator registry; see {{registry}}. Unregistered values MUST be ignored, and registered values MAY be ignored.

# Filtering Incident ID {#incident-id}

A Filtering Incident ID is an opaque, string identifier for a particular filtering incident. It might be specific to a particular request, but need not be. It is carried in the EXTRA-TEXT field of the Extended DNS Error with the JSON field name "inc". For example:

~~~ json
{
  "inc": "abc123"
}
~~~

# Incident Resolution Templates {#template}

An Incident Resolution Template is a URI Template {{!RFC6570}} that, upon expansion, provides a URI that can be dereferenced to obtain a Filtering Incident Description document (see {{format}}).

It MUST be a Level 1 or Level 2 template (see {{Section 1.2 of RFC6570}}). It has the following variables available to it:

* ro: the DNS Resolver Operator ID (see {op-id})
* inc: the Filtering Incident ID (see {incident-id})

For example:

~~
https://resolver.example.com/filtering-incidents/{inc}
~~

When dereferencing this URL, HTTP content negotiation for language SHOULD be used; see {{!Section 12 of RFC9110}}.

# The Filtering Incident Description Format {#format}

The Filtering Incident Description Format is a JSON {{!RFC8259}} document format for returning information about a particular filtering incident. Its root is an Object, with the following members:

* inc: String; the Filtering Incident ID
* resolver: String; a short textual name for the resolver operator (RECOMMENDED to be no longer than 64 characters)
* authority: String; a short textual name for the authority that required the filtering (RECOMMENDED to be no longer than 64 characters)
* description: String; a short textual description of the incident (RECOMMENDED to be no longer than 256 characters)

All members above are mandatory. New members can be added by updating this specification.

# IANA Considerations

## EXTRA-TEXT JSON Names

IANA will register the following fields in the "EXTRA-TEXT JSON Names" sub-registry established by {{I-D.ietf-dnsop-structured-dns-error}}:

* JSON Name: "ro"
* Short Description: a short, textual string that uniquely identifies the operator of a DNS resolver
* Mandatory: no
* Specification: [this document]

* JSON Name: "inc"
* Short Description: an opaque, string identifier for a particular filtering incident
* Mandatory: no
* Specification: [this document]

## The DNS Resolver Identifier Registry {#registry}

IANA will establish a new registry, the "DNS Resolver Identifier Registry." Its registration policy is first-come, first-served (FCFS), although IANA may refuse registrations that are deemed to be deceptive or spurious.

It contains the following fields:

* Name: The name of the DNS resolver operator
* Contact: an e-mail address or other appropriate contact mechanism
* DNS Resolver Operator ID: see {{op-id}}
* Incident Resolution Template: see {{template}}

# Security Considerations

This specification does not provide a way to authenticate that a particular filtering incident as experienced by a client was actually associated with the DNS resolver operator claimed. This means that an attacker (for example, one controlling a DNS resolver) can claim that a filtering incident is associated with an operator when it in fact was not.

However, to be successful an attacker would need to reuse an existing incident identifier that is supported by a DNS resolver operator recognised by the client. It is not currently thought to be particularly advantageous to an attacker to do so.


--- back

