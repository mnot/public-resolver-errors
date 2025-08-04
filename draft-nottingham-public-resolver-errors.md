---
title: DNS Filtering Details for Applications
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
      - Melbourne
    country: Australia
    email: mnot@mnot.net
    uri: https://www.mnot.net/
 -
    ins: D. Adrian
    name: David Adrian
    organization: Google
    country: USA
    email: davidcadrian@gmail.com

--- abstract

{{!I-D.ietf-dnsop-structured-dns-error}} introduces structured error data for DNS responses that have been filtered. This draft suggests additions to that mechanism that enable applications to convey the details of some filtering incidents to their users.

--- middle


# Introduction

Internet DNS resolvers are increasingly subject to legal orders that require blocking or filtering of specific names. Because such filtering happens during DNS resolution, there is not an effective way to communicate what is happening to end users, often resulting in misattribution of the issue as a technical problem, rather than a policy intervention.

This draft defines a mechanism to communicate such details when DNS resolver filtering of a name is legally mandated, based upon the structured error data for DNS responses introduced by {{!I-D.ietf-dnsop-structured-dns-error}}.

Allowing DNS resolvers to inject user-visible messages brings unique challenges. Because DNS resolvers are often automatically configured by unknown networks and DNS responses are unauthenticated, these messages can come from untrusted parties -- including attackers (e.g., the so-called "coffee shop" attack) that leverage many users' lack of a nuanced model of the trust relationships between all of the parties that are involved in the service they are using.

Furthermore, lowering the barrier to the presentation of messages explaining why access has been denied by the DNS resolver risks encouraging the wider deployment of DNS-based censorship on the Internet.

This draft attempts to mitigate these risks by minimising the information carried in the DNS response to abstract, publicly registered identifiers -- the DNS Resolver Operator ID and the Filtering Incident ID. A consuming party (e.g., a Web browser) can selectively present messages from those operators that they believe to be using this mechanism for its stated goal -- in particular, those who using it to surface policy-driven filtering, rather than enact discretionary censorship or attack end users.

## Example

In typical use, a DNS query that is filtered might contain an Extended DNS Error Code 17 (see {{!RFC8914}}) and an EXTRA-TEXT field containing:

~~~ json
{
  "ro": "exampleResolver",
  "inc": "abc123"
}
~~~

This indicates that the "exampleResolver" resolver has generated the error, and the incident identifier is "abc123".

An application that decides to present errors from "exampleResolver" to its users would look up "exampleResolver" in a local copy of the IANA DNS Resolver Identifier Registry (see {{registry}}) and obtain the corresponding template (see {{template}}). For purposes of this example, assume that the registry entry for that value contains:

~~~
https://resolver.example.com/filtering-incidents/{inc}
~~~

That template can be expanded using the value of "inc" to:

~~~
https://resolver.example.com/filtering-incidents/abc123
~~~


The application could (but might not) then decide to convey some or all of this information to its user; for example, with a statement that conveys:

> The webpage at www.example.net was blocked due to a legal request. Your DNS resolver may have more information about the legal request here:
>
> https://resolver.example.com/filtering-incidents/abc123

Note that there is no requirement for the template to expand to a URL on any particular hostname; for example, it could be hosted by a party other than the resolver's server.


## Notational Conventions

{::boilerplate bcp14-tagged}

# Data Types

This section defines the data types used to look up the details of a filtering incident from a DNS error response. Note that these identifiers are not for presentation to end users.

## DNS Resolver Operator ID {#op-id}

A DNS Resolver Operator ID is a short, textual string that uniquely identifies the operator of a DNS resolver. It is carried in the EXTRA-TEXT field of the Extended DNS Error with the JSON field name "ro". For example:

~~~ json
{
  "ro": "exampleResolver"
}
~~~

Generators MUST only use values that are registered in the DNS Resolver Operator registry; see {{registry}}. Consumers MUST ignore unregistered values, and MAY ignore registered values.

## Filtering Incident ID {#incident-id}

A Filtering Incident ID is an opaque, string identifier for a particular filtering incident. It might be specific to a particular request, but need not be. It is carried in the EXTRA-TEXT field of the Extended DNS Error with the JSON field name "inc". For example:

~~~ json
{
  "inc": "abc123"
}
~~~

# Incident Resolution Templates {#template}

An Incident Resolution Template is a URI Template {{!RFC6570}} contained in the DNS Resolver Identifier Registry ({{registry}}) that, upon expansion, provides a URI that can be dereferenced to obtain details about the filtering incident.

It MUST be a Level 1 or Level 2 template (see {{Section 1.2 of RFC6570}}). It has the following variables available to it:

ro:
: the DNS Resolver Operator ID (see {{op-id}})

inc:
: the Filtering Incident ID (see {{incident-id}})

For example:

~~~
https://resolver.example.com/filtering-incidents/{inc}
~~~

Applications MUST store a local copy of the DNS Resolver Identifier Registry for purposes of template lookup; they MUST NOT query the IANA registry upon each use.


# IANA Considerations

## EXTRA-TEXT JSON Names

IANA will register the following fields in the "EXTRA-TEXT JSON Names" sub-registry established by {{I-D.ietf-dnsop-structured-dns-error}}:

JSON Name:
: "ro"

Short Description:
: a short, textual string that uniquely identifies the operator of a DNS resolver

Mandatory:
: no

Specification:
: this document
{: spacing="compact"}


JSON Name:
: "inc"

Short Description:
: an opaque, string identifier for a particular filtering incident

Mandatory:
: no

Specification:
: this document
{: spacing="compact"}

## The DNS Resolver Identifier Registry {#registry}

IANA will establish a new registry, the "DNS Resolver Identifier Registry." Its registration policy is first-come, first-served (FCFS), although IANA may refuse registrations that it deems to be deceptive or spurious.

It contains the following fields:

Name:
: The name of the DNS resolver operator

Contact:
: an e-mail address or other appropriate contact mechanism

DNS Resolver Operator ID:
: see {{op-id}}

Incident Resolution Template:
: see {{template}}

The Incident Resolution Template can be updated by the contact at any time. However, operators SHOULD accommodate potentially long lag times for applications to update their copies of the registry.


# Security Considerations

This specification does not provide a way to authenticate that a particular filtering incident as experienced by an application was actually associated with the information presented. This means that an attacker (for example, one controlling a DNS resolver) can claim that a particular filtering incident is occurring when in fact it is not. However, a successful attack would need to reuse an existing DNS Resolver Operator ID and Filtering Incident ID that combine to expand to a URL that can be successfully dereferenced. Doing so is not currently thought to be particularly advantageous to an attacker to do so. Future iterations of this specification may introduce more robust protections.

The details of DNS responses are not available to all applications, depending on how they are architected and the information made available to them by their host. As a result, this mechanism is not reliable; some applications will not be able to display this error information.

Because the registry is first-come, first-served, Applications (such as Web browsers) will need to exercise judgement regarding which operators' error messages they display to users. This decision might be influenced by the identity of the resolver (e.g., so-called "public resolvers" are likely to use this mechanism responsibly), its history (e.g., a well-known Internet Service Provider that has been subject to legal filtering orders), or local configuration (e.g., application or operating system settings that indicate that a particular resolver is to be trusted).

--- back

# Acknowledgements

Thanks to David Adrian, Tommy Pauly, Emily Stark, and Martin Thomson for their input to this specification.
