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

{{!I-D.ietf-dnsop-structured-dns-error}} introduces structured error data for DNS responses that have been filtered. This draft suggests additions to that mechanism that enable applications to convey the details of some filtering incidents by referencing entries in a database of incidents.

--- middle


# Introduction

Internet DNS resolvers are increasingly subject to legal orders that require blocking or filtering of specific names. Because such filtering happens during DNS resolution, there is not an effective way to communicate what is happening to end users, often resulting in misattribution of the issue as a technical problem, rather than a policy intervention.

Some organizations, such as Lumen, monitor legally-mandated filtering as a
public good, and track filtering incidents in a publicly accessible databases.
Public resolvers themselves may also choose to track filtering requests over
time and make them available.

This draft defines a mechanism to communicate an identifier in a database of
filtering incidents when a DNS resolver filtering of a name is legally mandated,
based upon the structured error data for DNS responses introduced by
{{!I-D.ietf-dnsop-structured-dns-error}}.

A consuming party (e.g., a Web browser) can use the identifier to construct a
link to the specific entry in the database provider. This enables user agents to
direct users to a location with additional context about why the filtering was
required.

Allowing DNS resolvers to inject links or user-visible messages brings unique challenges.
Because DNS resolvers are often automatically configured by unknown networks and
DNS responses are unauthenticated, these messages can come from untrusted
parties -- including attackers (e.g., the so-called "coffee shop" attack) that
leverage many users' lack of a nuanced model of the trust relationships between
all of the parties that are involved in the service they are using. Furthermore,
lowering the barrier to the presentation of messages explaining why access has
been denied by the DNS resolver risks encouraging the wider deployment of
DNS-based censorship on the Internet.

This draft attempts to mitigate these risks by minimising the information
carried in the DNS response to abstract, publicly registered identifiers
associated with databases of filtering incidents---the Database Operator ID and
the Filtering Incident ID. A consuming party can choose which database
identifiers they support are are willing to direct their users to, without
enabling every DNS server to surface arbitrary links and text.

## Example

In typical use, a DNS query that is filtered might contain an Extended DNS Error Code 17 (see {{!RFC8914}}) and an EXTRA-TEXT field "fdb", which is an array of references to filtering database entries:

~~~ json
{
  "fdb": [
    {"db": "example",
    "id": "abc123"},
    {"db": "lumen",
    "id": "def456"}
  ]
}
~~~

This indicates that the filtering incident can be accessed in two different
databases, and the ID associated with each database. In this example, the data
is available in the "example" database at identifier "abc123", and in the "lumen"
database at identifier "def456".

An application that evaluates the DNS server and decides to present links to "example" to its users would look up "example" in a local copy of the DNS Filtering Incident Database Registry (see {{registry}}) and obtain the corresponding template (see {{template}}). For purposes of this example, assume that the registry entry for that value contains:

~~~
https://resolver.example.com/filtering-incidents/{id}
~~~

That template can be expanded using the value of "id" to:

~~~
https://resolver.example.com/filtering-incidents/abc123
~~~


The application could (but might not) then decide to convey some or all of this information to its user; for example, with a statement that conveys:

> The webpage at www.example.net was blocked due to a legal request. Your DNS resolver may have more information about the legal request here:
>
> https://resolver.example.com/filtering-incidents/abc123

Note that there is no requirement for the resolver to construct links to any database, nor for results from any DNS server. The resolver both choose which database providers it supports, and can evaluate whatever mechanisms it chooses to determine when and if to provide the link to the database provider.


## Notational Conventions

{::boilerplate bcp14-tagged}

# Data Types

This section defines the data types used to look up the details of a filtering incident from a DNS error response. Note that these identifiers are not for presentation to end users.

## DNS Filtering Database Entry {#entry-id}

A Filtering Database ID is a short, textual string that uniquely identifies the
operator of a database of filtering incidents. It uses the key "db".

A Filtering Incident ID is an opaque, string identifier for a particular
filtering incident. It might be specific to a particular request, but need not
be. It uses the key "id".

An object containing both a Filtering Database ID and a Filtering Incident ID is a Filtering Database Entry.

~~~ json
{
  "db": "example",
  "id": "abc123"
}
~~~

## DNS Filtering Database Entry List {#entry-list}

A DNS Filtering Database Entries list is an array of Filtering Database Entry
objects. Each entry MUST be a unique identifier for the same underlying
incident.

It is carried in the EXTRA-TEXT field of the Extended DNS Error with the JSON
field name "fdbs". For example:

~~~ json
{
  "fdbs": [ { ... }, { ... }, ... ]
}
~~~

Different clients will implement support for a varying set of database
operators. Resolvers provide a list of entries (rather than a single entry) so
that they can support as many clients with diverse database sets as possible.

# Database Entry Resolution Templates {#template}

An Incident Resolution Template is a URI Template {{!RFC6570}} contained in the DNS Resolver Identifier Registry ({{registry}}) that, upon expansion, provides a URI that can be dereferenced to obtain details about the filtering incident.

It MUST be a Level 1 or Level 2 template (see {{Section 1.2 of RFC6570}}). It has the following variables from the Filtering Database Entry (see {{entry-id}}) available to it:

db:
: the Filtering Database Operator ID

id:
: the Filtering Incident ID

For example:

~~~
https://resolver.example.com/filtering-incidents/{inc}
~~~

Applications MUST store a local copy of the DNS Resolver Identifier Registry for purposes of template lookup; they MUST NOT query the IANA registry upon each use. The registry is keyed by the Filtering Database Operator ID.


# IANA Considerations

## EXTRA-TEXT JSON Names

IANA will register the following fields in the "EXTRA-TEXT JSON Names" sub-registry established by {{I-D.ietf-dnsop-structured-dns-error}}:

JSON Name:
: "fdbs"

Short Description:
: a array of filtering database entries

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
: see {{entry-id}}

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
