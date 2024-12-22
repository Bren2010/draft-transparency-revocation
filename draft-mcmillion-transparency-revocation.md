---
title: "Reliable Transparency and Revocation Mechanisms"
category: info

docname: draft-mcmillion-transparency-revocation-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: SEC
workgroup: TLS Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "Bren2010/draft-transparency-revocation"
  latest: "https://Bren2010.github.io/draft-transparency-revocation/draft-mcmillion-transparency-revocation.html"

author:
 -
    fullname: "Brendan McMillion"
    email: "brendanmcmillion@gmail.com"

normative:

informative:
  ct-in-wild:
    target: https://dl.acm.org/doi/pdf/10.1145/3319535.3345653
    title: "Certificate Transparency in the Wild: Exploring the Reliability of Monitors"
    date: 2019-11-06
    author:
      - name: Bingyu Li


--- abstract

This document describes reliable mechanisms for the publication and revocation
of Transport Layer Security (TLS) certificates.  This reliability takes several
forms. First, it provides browsers a strong guarantee that all certificates they
accept are truly published and unrevoked at the time they're accepted. Second,
it allows website operators to monitor for mis-issuances related to their
domains in a highly efficient way, without relying on third-party services.
Third, it provides a high degree of operational redundancy to minimize the risk
of cascading outages.

--- middle

# Introduction

The Certificate Transparency (CT) ecosystem created by {{?RFC6962}} has been
immensely valuable to security on the internet. However, various cracks in the
design have become apparent over time:

The security that CT provides to verifiers is based on an assumption of
non-collusion between multiple parties. Historically, this assumption has been
challenging to maintain, as it degrades quickly without active management. The
compromise of a single Transparency Log or the unexpected acquisition of a
single business is often sufficient to allow the possibility of undetectable
mis-issued certificates. This is compounded by the fact that multiple parties in
the CT ecosystem play multiple roles (such as Certificate Authorities that are
also Transparency Log operators that are also browser vendors), which makes
reasoning about the possibility of collusion even more tricky.

It is also becoming far too expensive to both operate a CT log, and to monitor
CT logs. Logs are required to serve their entire contents to anyone on the
internet, which consumes a significant amount of outbound bandwidth. Inversely,
monitoring certificate issuance in CT requires downloading the entire contents
of all logs, which is several terabytes of data at minimum.

The flip side of publishing certificates is reliably revoking the mis-issued
certificates that are identified by site operators. However, revocation systems
have historically been plagued by a requirement to "fail open". That is,
revocation checks would stop being enforced in certain (often, easily
engineered) scenarios. For example, if the server didn't proactively offer proof
of non-revocation or if a third-party service was inaccessible for too long,
this might cause revocation checks to be disabled. One promising exception to
this general principle, OCSP Must-Staple {{?RFC7633}}, has unfortunately been
the victim of waning support for OCSP by Certificate Authorities.

This motivates a need for a new system of publishing certificates that's
resistant to collusion and dramatically more efficient to operate, and a need
for a new system of revoking certificates that can be reliably enforced.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Architecture

The system has several roles, which we describe in more detail below. Parties
are allowed to assume multiple roles.

**Certificate Authority:**
: A trusted service that performs domain-control validation and authenticates
  certificates and revocations.

**Transparency Log:**
: An untrusted service that provides an append-only, publicly-auditable log of
  certificates and revocations issued by a wide range of Certificate
  Authorities.

**Site Operator:**
: The individual or organization responsible for the operation and maintenance
  of a website, identified by a single domain name.

**User Agent:**
: A software application, typically a web browser, that acts on behalf of a user
  to access and interact with websites.

## Requirements

The following baseline requirements for the system are as follows:

1. For a certificate to be mis-issued and not eventually published: two trusted
   parties must collude and all verifiers that observed the certificate must be
   stateless
2. Violations of the transparency guarantee must have a high potential for
   after-the-fact detection by stateful verifiers.
3. It must be reasonably efficient for site operators to audit all trusted
   Certificate Authorities for mis-issuances affecting their domain names.
4. The system must not have any single points of failure, other than those in
   the web PKI as it exists today.
5. End-users must be able to connect to a server without having immediate
   connectivity to third-party services.
6. The domain names of websites visited by the end-user must not be leaked,
   other than how they are in the web PKI as it exists today.
7. The system must be reasonable for non-browser user agents to deploy.
8. The system must have a reasonable path to scale indefinitely.

These requirements and their main consequences are discussed in more detail in
the following subsection.

## Discussion

**Transparency requires stateful verification.** The fundamental goal of any
transparency system is to ensure that data shown to one participant is equally
visible to any other participant. This generally involves the use of a
"transparency log" that publishes data, and may or may not have additional
structure to support efficient searches for specific data. Transparency Logs
must be verified to be linear, in terms of being append-only, and
internally-consistent with respect to any rules that the log may have about its
content or structure. Without relying on a third-party to verify these
properties, which may collude with the Transparency Log, end-users must retain
state to verify these properties themselves. In addition to ensuring that the
various views of the log that an end-user has seen are linear and
internally-consistent, end-users must also *gossip* their view of the log with a
wide range of other parties. Gossiping allows end-users to detect when they've
been shown a fork of the log that could potentially contain malicious data
that other participants in the system are unaware of.

Transparency systems based on stateless verification are more accurately termed
"co-signing" schemes. That's because, with stateless verification, the security
of any such construction reduces solely to successful verification of the
transparency log's signature on a claim. The transparency guarantees of
co-signing schemes can be undermined by collusion between the co-signers. In the
web PKI, the co-signers would be a CA and one or more transparency logs. The
diverse and rapidly changing nature of the web makes collusion between
participants easy to achieve and difficult to detect. As such, co-signing
schemes generally unsuitable for use in the web PKI without requiring a
multitude of co-signers. However, transmitting a multitude of signatures from
larger, PQ-secure signature schemes can measurably degrade the performance of
TLS connections. Additionally, co-signing schemes provide no way to detect
collusion after the fact without resorting to stateful verification.

**Servers must provide proof directly to end-users that their certificate
satisfies transparency requirements and isn't revoked.** If proof of
transparency and non-revocation isn't provided by the server, it must be fetched
from one or more third-party services. The primary issue with this is that it
ties the server's global availability to the global availability of these
third-party services, for which the server operator has no way to preempt or
resolve deficiencies. As a result of this, proposals for transparency and
revocation that rely on connectivity to third-party services have historically
been required to fail open. That is, if the third-party service is inaccessible
for too long of a period of time, the end-user stops enforcing these security
properties altogether.

A second issue is that, since we're primarily interested in describing a system
that works equally well regardless of the end-user's software vendor, these
third-party services might not be permitted to restrict access to a subset of
end-users. They would receive regular requests from all internet-connected
devices, which would present significant scaling and centralization concerns.

**Servers must refresh their certificates regularly and automatically.** This is
a direct consequence of the decision that servers must be responsible for
providing end-users proof that their certificates are not revoked. If a server
is not required to refresh its certificate, it can attempt to delay the end-user
from learning about a change of its revocation status.

**Revocation must be provided by the transparency system.** A CA can initiate
revocation either by declining to sign new statements related to a certificate
(for example, by not renewing a short-lived certificate or not creating an OCSP
staple), or by signing a new "statement of revocation" for the certificate.

In the case where CAs initiate revocation by signing a new "statement of
revocation", proving that a certificate is not revoked consists of proving that
such a statement does not exist. Proving that a statement does not exist
requires exhaustive knowledge of all such statements. Any method of conveying
this exhaustive knowledge, if it is not a transparency system, admits the
possibility of split-view attacks which are not eventually detected. Split-view
attacks in this context would allow a CA to mis-issue a certificate, claim to
revoke it, and then maintain the certificate's utility by presenting different
views of its revocation status to attack victims than to other participants. The
possibility of such a split-view attack would render revocation fundamentally
insufficient for correcting mis-issuance, and would create a need for a second
(more effective) revocation mechanism.

In the case where CAs initiate revocation by declining to sign new statements,
this makes the CA a single point-of-failure for websites relying on it. A
prolonged CA outage would have the effect of revoking all certificates and
causing a cascading outage. Proposals for revocation that fall into this
category have historically mitigated this risk by providing slower revocation,
bounded by the longest conceivable outage that a CA may have (typically at least
one week).

When specifically considering short-lived certificates as an approach to
revocation, effectiveness depends on whether or not **all** certificates are
required to be short-lived. If end-users enforce that all certificates are
short-lived, and issuance is transparent, then revocation is provided by the
transparency system as claimed. If certificates may be issued with longer
lifespans, then a second revocation mechanism for these certificates is
necessary. Considering solutions other than short-lived certificates, where the
CA initiates revocation by declining to sign some statement, it's clear that the
same potential for split-view attacks exists as discussed above.

**Transparency logs must implement the Key Transparency protocol.** As stated at
the beginning of this section, the goal of any transparency system is to ensure
that data shown to one participant is equally visible to any other participant.
An important aspect of this requirement is that site operators must be able to
contact a transparency log and verifiably receive all of the certificates and
revocations that are relevant to them.

Most transparency systems require downloading the entirety of the log's contents
to ensure that all potentially relevant entries are found. This quickly becomes
prohibitively expensive for all parties. Currently roughly 7.5 million
certificates are issued per day, with an average size of 3 kilobytes. This means
that a site operator would need to download almost 700 gigabytes of certificates
to cover a single month of issuance. Outbound bandwidth typically costs between
1 to 9 cents per gigabyte, which means that providing this data to a single
site operator would cost the transparency log between $6 to $60. For any
reasonable estimate of the number of site operators on the internet, this
represents an exceptional burden on a transparency log.

In previously deployed systems, because of this exceptional cost, site operators
have elected not to do this work themselves and instead outsourced it to
third-party monitors. Third-party monitors represent a problematic break in the
security guarantees of the system, as there are no enforceable requirements on
their behavior. They are not audited for correct behavior like certificate
authorities are, and there are no technical mechanisms to prevent misbehavior
like a transparency log would have. This has had the real-world impact of
undermining the claimed transparency guarantees of these systems {{ct-in-wild}}.

Key Transparency {{!I-D.draft-ietf-keytrans-protocol}} augments a transparency log
with additional structure to allow efficient and verifiable searches for
specific data. This allows server operators to download only the contents of the
transparency log that's relevant to them, while still being able to guarantee
that no certificates have been issued for their domains that they are unaware
of. As a result of the significantly reduced need for outbound bandwidth,
operating such a transparency log would cost around one million times less than
it would otherwise.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
