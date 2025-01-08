---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Incrementally Deployable DNSSEC Delegation"
abbrev: "incremental-dnssec"
category: std

docname: draft-homburg-deleg-incremental-dnssec-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: 2025-01-06
consensus: true
v: 3
area: int
workgroup: DNS Delegation
keyword:
 - Internet-Draft
 - DNS
 - Resolver
 - Delegation
 - Authoritative
 - Deployable
 - Extensible
venue:
  group: deleg
  type: Working Group
  mail: dd@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dd/
  github: NLnetLabs/incremental-dnssec

author:
 -
    fullname: Philip Homburg
    organization: NLnet Labs
    email: philip@nlnetlabs.nl
 -
    fullname: Willem Toorop
    organization: NLnet Labs
    email: willem@nlnetlabs.nl

normative:

informative:

--- abstract

This document proposes a DNSSEC delegation mechanism that complements {{?I-D.homburg-deleg-incremental-deleg}}.
In addition this mechanism simplifies multi-signer setups by removing the need to coordinate for signers during key rollovers.

--- middle

# Introduction

This document describes a DNSSEC delegation mechanism that complements {{?I-D.homburg-deleg-incremental-deleg}}.
In particular, this mechanism makes it possible to store the contents of DS records as 'ds' parameters in SVCB-style records.
This way, a single mechanism can specify both name servers and DNSSEC delegations.

In addtion to a replacement for DS records, this document also introduces a 'dnskeyref' parameter that provides more flexibility and reduces the need to coordinate both for key rollovers and for multi-signer setups.

There are two main problems with the current DS record.
The first is a DS record refers to a key in a DNSKEY RRset.
Updating this keys (during a key rollover) requires updating the corresponding DS record.
At the moment there is no widespread mechanism to update DS records automatically.
However, even if such an update mechanism would become widespread, the signer
also has to track to propagation of the new DS record to the secondaries of the parent zone.
This is needed to make sure that all old DS records are expired in caches before moving to the next stage of a key roll.

The second problem is that DS records enforce the use of a single DNSKEY RRset (at the apex of the zone).
This means that in a multi-signer setup, signers have to coordinate when
rolling their signing keys.

There is one minor problem that can be solved as well.
Currently every signed zone has to have a DNSKEY RRset at the apex.
If a collection of zones share the same keys then this can be burden to maintain.

The solution to these problems is to introduce a dnskeyref parameter that contains the names of DNSKEY RRsets that are allowed to sign a zone.
This extra level of indirection avoids the need to involve the parent during key rolls.
Multiple DNSKEY RRsets make it possible to perform key rolls in a multi-signer setup without coordination.
Finally, this allows a single DNSKEY RRset to be used for multiple domains.

The charter for the Deleg working group has the following:

{:quote}
> The DNS protocol has limited ability for authoritative servers to signal their capabilities to recursive resolvers. In part, this stems from the lack of a mechanism for parents (often registries) to specify additional information about child delegations (often registrants) beyond NS, DS, and glue records. Further complicating matters is the similar lack of a mechanism for a registrant to signal that the operation of a delegation point is being outsourced to a different operator, leaving a challenge when operators need to update parental information that is only in the control of the child. Data is often out of synchronization between parents and children, which causes significant operational problems.

Most focus until today has been on making it possible to update the name servers of a delegation without involving the registrant.
However, DNSSEC has a similar problem.

The introduction of the dnskeyref parameter make it possible to manage DNSSEC without involving the registrant.

One extra feature that becomes possible the use of SVCB-style records is to let the zone manage downgrade attacks.
By introducing ds of dnskeyref parameters at different priority level, the zone can signal which algorithm is preferred.

{:removeinrfc}
## Incremental Deleg

This document builds on {{?I-D.homburg-deleg-incremental-deleg}} so examples
will show _deleg labels.
This document does however assume a new SVCB-style type called DELEG to be able to make clear that some semantics are different from SVCB records.

The mechanisms described in this document can work with {{?I-D.wesplaap-deleg}} as well.
However, this will create a deployment issue.
If a validator that implements this document and {{?I-D.wesplaap-deleg}} forwards requests to a recursive resolver that does not implement {{?I-D.wesplaap-deleg}} then validation will fail if the delegation uses DELEG records.

In contrast the same setup but then with {{?I-D.homburg-deleg-incremental-deleg}} will work.

## Terminology

{::boilerplate bcp14-tagged}

This document follows terminology as defined in {{?RFC9499}}.

Throughout this document we will also use terminology with the meaning as defined below:

{: vspace="0"}
Incremental deleg:
: The delegation mechanism as specified in {{?I-D.homburg-deleg-incremental-deleg}}.

Incremental delegation:
: A delegation as specified in {{?I-D.homburg-deleg-incremental-deleg}}.

AliasMode:
: An SVCB mode as specified in {{?RFC9460}}.

ServiceMode:
: An SVCB mode as specified in {{?RFC9460}}.

SvcParamKey:
: An SVCB field as specified in {{?RFC9460}}.

SvcParamValue:
: An SVCB field as specified in {{?RFC9460}}.

# New parameters

Two new SVCB-style parameters are introduced: ds and dnskeyref.

To limit validation resources and avoid accidentally delegating security, these two parameters are not valid after following an SVCB-style record in AliasMode.

For DELEG records, these parameters are allowed to be added to DELEG records in
AliasMode.
To increase flexibility, a DELEG record in ServiceMode can be created with just ds or dnskeyref parameters but without name server delegation.
In that case the TargetName has to be set to ".".
Unlike SVCB records, for DELEG we allow mixing of AliasMode and ServiceMode records.

## The ds parameter

The ds parameter puts the equivalent of multiple DS records in one parameter.

To simplify parsing, the presentation format is a space separated list of strings where each string is in the form \<key-tag>:\<algorithm>:\<igest-type>:\<digest>.
The parts key-tag, algorithm, digest-type and digest have their meaning and
encoding as described for the presentation format of the DS record (See {{?RFC4034}}, Section 5.3).

A parameters start with a 2-octet field that contains the SvcParamKey following
by a 2-octet field containing the length of the SvcParamValue.

For the ds parameter, the SvcParamValue field consists of a sequence of
DS record RDATA octet sequences prefixed by a 1-octet length field.
The length field contains the length of the DS record RDATA.

The encoding of the RDATA octet sequence is the same as for an equivalent
DS record (with the exact same owner name as the SVCB-style record that
contains the ds parameter).

## The dnskeyref parameter

The dnskeyref parameter names one or more DNSKEY RRsets.
The presentation format of the value is a space separated list of DNS names.

The wire format starts with a 2-octet field that contains that SvcParamKey followed by a 2-octet filed than contains the length of SvcParamValue.

For dnskeyref, the SvcParamValue consists of just a concontenated list of uncompressed DNS names in wire format.
DNS names are self terminating so there is no need for extra length fields.

# Validator behavior

When following a delegation, a validator MUST first look for an Incremental delegation.
If presence or absence is not DNSSEC Secure (i.e., the status is Insecure, Bogus or Indeterminate) then the child is not secure and the algorithm stops here.

If absence of an Incremental delegation is proven to be Secure then then validator MAY continue validating using DS records.

If a secure Incremental delegation is found then the validator MUST ignore any DS records and solely rely on what is found in the Incremental deleg records.

If the Incremental deleg records contain no ds or dnskeyref parameters or these do not lead to any key with an algorithm that the validator supports then the delegation is considered insecure.

A validator MUST NOT use any ds or dnskeyref parameters found after following an SVCB-style record in AliasMode.
Restricting ds and dnskeyref to the top level is essential in preventing a DNS operator (who is supposed to only serve a zone, not sign it) from adding additional ds or dnskeyref parameters.

To support protection against downgrade attacks, a validator SHOULD consider only SVCB-style records with the highest priority that either have a ds parameter that has a least one algrithm supported by the validator or in the case of dnskeyref, a DNSKEY RRset that contains at least one key with an algorithm that is supported.

After selecting priority level, the validator uses any ds parameters to validate the DNSKEY RRset the apex of the child zone.
The DNSKEY RRsets that dnskeyref paramters refer need to be DNSSEC Secure to be used.
Any DNSKEY RRsets that are not DNSSEC Secure according to the validator MUST be ignored.

To limit validator resources, when validating a name that refers to a DNSKEY RRset, the validator should only use ds parameters (and DS records) and ignore any dnskeyref parameters.

Now that there are multiple DNSKEY RRsets that may be used to sign an RRset, the validator uses the Signer's Name in a signature to select a DNSKEY RRset to validate the signature. If the name does not match any of the DNSKEY RRset selected in the previous step then the signature MUST be ignored.

# Signer behavior

Signer MUST put ds and dnskeyref parameters only in DELEG records at an Incremental delegation and not in SVCB-style records that follow a DELEG record in AliasMode.

The ds parameter affects signers in one small way: the priority system of SVCB-style records can be used to provide protection again downgrades.
The signer can put a ds parameter with a more secure algorithm in a DELEG record with a higher priority and expect fallback to the less secure algorithm only if the validator does not understand the more secure algorithm.

The dnskeyref parameter requires that the validator puts the name of the DNSKEY RRset in the Signer's Name field of the signature.
This is a small but significant change to how signers operate.
The DNSKEY RRset MUST be DNSSEC Secure and the validation chain MUST only involve ds parameters or DS records.
The dnskeyref parameter also supports downgrade protection but has a number of additinal features that can be used by signers.

The first is that because dnskeyref directly refers to a DNSKEY RRset there is no longer any need for Key Signing Keys (KSK).
Those DNSKEY RRsets only contain Zone Signing Keys (ZSK).
This means that KSK rollovers are no longer necessary.
It is also no longer necessary to coordinate with the parent zone for key rollovers.

The second is that dnskeyref can refer to a DNSKEY RRset that is used to sign multiple zones.
Zones do no need to have their own copies of key material.
Instead a single DNSKEY RRset can be used for as many zones as needed.

Third, in a multi-signer setup, each signer can maintain its own DNSKEY RRset independent of other signers.
This means that each signer can perform key rollovers without any need to coordinate with other signers.

# Examples

## One ds parameter that is part of a delegation

~~~~
$ORIGIN example.
@                  IN  SOA   ns zonemaster ...
customer1._deleg   IN  DELEG 1 ( ns.customer1
                ipv4hint=198.51.100.1,203.0.113.1
                ipv6hint=2001:db8:1::1,2001:db8:2::1
                ds=60485:5:1:2BB183AF5F22588179A53B0A98631FAD1A292118
                               )
~~~~
{: title="One ds parameter that is part of a delegation"}

## One separate DELEG record with a ds parameter
 
~~~~
$ORIGIN example.
@                  IN  SOA   ns zonemaster ...
customer2._deleg   IN  DELEG 0 ns.operator1
                   IN  DELEG 1 ( .
                    ds="60485:5:1:2BB183AF5F22588179A53B0A98631FAD1A292118
 370:13:2:BE74359954660069D5C63D200C39F5603827D7DD02B56F120EE9F3A86764247C"
                               )
~~~~
{: title="One separate DELEG record with a ds parameter"}

## One separate DELEG record with a dnskeyref parameter
 
~~~~
$ORIGIN example.
@                  IN  SOA   ns zonemaster ...
customer3._deleg   IN  DELEG 0 ns.operator1
                   IN  DELEG 1 ( .
                    dnskeyref=customer3.keys.operator1
                               )
~~~~
{: title="One separate DELEG record with a dnskeyref parameter"}

## Two operators with dnskeyref parameters
 
~~~~
$ORIGIN example.
@                  IN  SOA   ns zonemaster ...
customer4._deleg   IN  DELEG 1 ( ns.operator1
                    dnskeyref=customer4.keys.operator1
                                  )
                   IN  DELEG 1 ( ns.operator2
                    dnskeyref=customer-keys.operator3
                                  )
~~~~
{: title="Two operators with dnskeyref parameters"}

# Security Considerations

The ds parameter is a slight improvement over the DS record because it allows
SVCB priorities to be used for downgrade protection.
Otherwise, the semantics are the same.
This relies on the restriction that ds (and dnskeyref) parameters can only appear the top level and not after following an SVCB-style record in AliasMode.

The dnskeyref parameter breaks the traditional chain semantics of DNSSEC.
With the ds parameter (or the DS record) the validity of a signature depends only on a chain of cryptographic primitives.
With dnskeyref, a DNS lookup is inserted.
This DNS lookup is protected by a traditional DNSSEC chain, but the presence of the name means that it is no longer a cryptographic chain.

This trade-off between struct security and flexibility is left to the zone operator.

# IANA Considerations

Per {{?RFC9460}}, IANA is requested to add the following entry to the DNS "Service Parameter Keys (SvcParamKeys)" registry:


|--------|-----------|------------------------|-------------------|-------------------|
| Number | Name      | Meaning                | Reference         | Change Controller |
|--------|-----------|------------------------|-------------------|-------------------|
| TBD    | ds        | DNSSEC delegations     | \[this document\] | IETF |            |
|--------|-----------|------------------------|-------------------|-------------------|
| TBD    | dnskeyref | Names of DNSKEY RRsets | \[this document\] | IETF |            |
|--------|-----------|------------------------|-------------------|-------------------|
{: title="Entry in the Service Parameter Keys (SvcParamKeys) registry"}

--- back

# Acknowledgments
{:numbered="false"}

