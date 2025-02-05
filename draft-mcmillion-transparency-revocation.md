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
domains in a highly efficient way without relying on third-party services.
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
internet, which consumes a significant amount of outbound bandwidth. Similarly,
monitoring certificate issuance in CT requires downloading the entire contents
of all logs, which is several terabytes of data at minimum.

The fundamental purpose for publishing TLS certificates is to allow site operators
to identify and and revoke those certificates which are mis-issued. However,
revocation systems have historically been plagued by a requirement to "fail
open". That is, revocation checks would stop being enforced in certain (often,
easily engineered) scenarios. For example, if the server didn't proactively
offer proof of non-revocation or if a third-party service was inaccessible for
too long, this might cause revocation checks to be disabled. One promising
exception to this general principle, OCSP Must-Staple {{?RFC7633}}, has
unfortunately been the victim of waning support for OCSP by Certificate
Authorities.

This motivates a need for a new system of publishing certificates that's
resistant to collusion and dramatically more efficient to operate, and a need
for a new system of revoking certificates that can be reliably enforced.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Architecture

The system has several roles, which we describe in more detail below. Parties
are allowed to assume multiple roles.

**Certificate Authority:**
: A service that performs domain-control validation and authenticates
  certificates and revocations.

**Transparency Log:**
: A service that provides an append-only, publicly-auditable log of certificates
  and revocations issued by a wide range of Certificate Authorities.

**Site Operator:**
: The individual or organization responsible for the operation and maintenance
  of a website, as identified by a single domain name.

**User Agent:**
: A software application, typically but not necessarily a web browser, that acts
  on behalf of a user to access and interact with websites.

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
"transparency log" that publishes data, which may or may not have additional
structure to support efficient searches for specific data. Transparency Logs
must be verified to be linear, in terms of being append-only, and
internally-consistent with respect to any rules that the log has about its
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
co-signers' signatures on a claim. The transparency guarantees of
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
revocation, effectiveness depends on whether or not **all** certificates in the PKI are
required to be short-lived. If end-users enforce that all certificates are
short-lived, and issuance is transparent, then revocation is provided by the
transparency system as claimed. If certificates may be issued with longer
lifespans, then a second revocation mechanism for these certificates is
necessary. Additionally, when considering solutions other than short-lived
certificates where the CA initiates revocation by declining to sign some
statement, it's clear that the same potential for split-view attacks exists as
discussed above.

**Transparency Logs must implement the Key Transparency protocol.** As stated at
the beginning of this section, the goal of any transparency system is to ensure
that data shown to one participant is equally visible to any other participant.
An important aspect of this requirement is that site operators must be able to
contact a Transparency Log and verifiably receive all of the certificates and
revocations that are relevant to them.

Most transparency systems require downloading the entirety of the log's contents
to ensure that all potentially relevant entries are found. This quickly becomes
prohibitively expensive for all parties. Currently roughly 7.5 million
certificates are issued per day, with an average size of 3 kilobytes. This means
that a site operator would need to download almost 700 gigabytes of certificates
to cover a single month of issuance. Outbound bandwidth typically costs between
1 to 9 cents per gigabyte, which means that providing this data to a single
site operator would cost the Transparency Log between $6 to $60. For any
reasonable estimate of the number of site operators on the internet, this
represents an exceptional burden on a Transparency Log.

In previously deployed systems, because of this exceptional cost, Site Operators
have elected not to do this work themselves and instead outsourced it to
third-party monitors. Third-party monitors represent a problematic break in the
security guarantees of the system, as there are no enforceable requirements on
their behavior. They are not audited for correct behavior like Certificate
Authorities are, and there are no technical mechanisms to prevent misbehavior
like a Transparency Log would have. This has had the real-world impact of
undermining the claimed transparency guarantees of these systems {{ct-in-wild}}.

Key Transparency {{!I-D.draft-ietf-keytrans-protocol}} augments a Transparency Log
with additional structure to allow efficient and verifiable searches for
specific data. This allows Site Operators to download only the entries of the
Transparency Log that're relevant to them, while still being able to guarantee
that no certificates have been issued for their domains that they are unaware
of. As a result of the significantly reduced need for outbound bandwidth,
operating such a Transparency Log would cost around one million times less than
it would otherwise.

## Summary

In summary, the system described in this document works as follows:

- Site Operators obtain a certificate from a Certificate Authority and submit it
  to one of many trusted Transparency Logs to obtain a proof of inclusion.
- User Agents that contact the Site Operator's server over TLS include compact
  transparency-related state in their ClientHello. The server provides its
  certificate and proof of inclusion (potentially modified based on the agent's
  advertised state) in the Certificate message. The User Agent verifies that the
  proof of inclusion aligns with its state, is sufficiently recent, and
  indicates the certificate is unrevoked.
- As time goes on, the current proof of inclusion will become stale. Site
  Operators refresh their proof of inclusion by requesting a new one from the
  Transparency Log.
  - If the first Transparency Log is offline, the Site Operator may failover to
    any of several other qualified Transparency Logs.
- At any time in the background, the Site Operator may query any of the trusted
  Transparency Logs and verifiably learn about all new certificates issued for
  their domain. Since the Key Transparency protocol specifies a "correct"
  location for a domain's certificates to be stored, which User Agents enforce
  when verifying proofs of inclusion, requesting all certificates related to a
  domain always remains highly efficient.

The remainder of this document describes these steps in more detail.

# Transparency Log

Transparency Logs are online services that maintain a tree data structure and
provide access to it through the endpoints described below. Transparency Logs
are generally only contacted by Site Operators. Site Operators regularly issue
requests to the Transparency Log's endpoints to either obtain fresh proofs of
inclusion for their certificates, or to monitor for mis-issuances affecting
their domain names.

While a Transparency Log may be operated by a Certificate Authority,
Transparency Logs SHOULD accept certificates issued by a broad set of the
current widely-trusted Certificate Authorities. This ensures that, if one
Transparency Log has an outage, there are several other Transparency Logs that
Site Operators can fallback on for the purpose of fetching a fresh proof of
inclusion.

## Structure

The data structure maintained by a Transparency Log is identical to the Combined
Tree described in {{!I-D.draft-ietf-keytrans-protocol}}, with two exceptions:
the search keys that are used to navigate the Prefix Tree, and the
data committed to by each Prefix Tree leaf node, are different.

The search key used to navigate the Prefix Tree is the hash of the Common Name
field of a TLS certificate. The hash function used corresponds to the
ciphersuite hash function. No VRF is used, or version counter is included in the
hash function input, as they would be in {{!I-D.draft-ietf-keytrans-protocol}}.

Rather than a privacy-preserving commitment, each Prefix Tree leaf contains the
hash of a `DomainCertificates` structure:

~~~ tls
struct {
  opaque root<Hash.Nh>;
  uint32 first_valid;
  uint32 invalid_entries<0..2^8-1>;
} DomainCertificates;
~~~

The `root` field contains the root hash of a Log Tree, which will be referred to
as the **Certificate Subtree** to avoid confusion with the top-level Log Tree.
The Certificate Subtree contains all certificate chains that may be presented
for a particular domain name (the certificate's Common Name), in the order they
were logged. The leaves of the Certificate Subtree are represented as
`SubtreeLogLeaf` structures, used in place of `LogLeaf`:

~~~ tls
opaque Certificate<0..2^16-1>;

struct {
  Certificate chain<0..2^8-1>;
} SubtreeLogLeaf;
~~~

The `first_valid` field contains the index of the first entry in the Certificate
Subtree where no certificate in the chain is revoked or expired. The
`invalid_entries` field contains the list of indices of all entries to the right
of `first_valid` where one or more of the certificates in the chain have been
revoked.

Transparency Logs SHOULD determine whether a certificate chain is expired by
comparing the Not After field of each certificate in the chain to the timestamp
of the Log Tree's rightmost leaf. However, Transparency Logs do not have a
proactive responsibility to keep the `first_valid` field updated; it is simply
provided as a mechanism to drain `invalid_entries`.

When computing `PrefixLeaf`, the hash of the leaf certificates' Common Name is
stored in the `vrf_output` field and the hash of `DomainCertificates` is stored
in the `commitment` field.

### Subtree Inclusion Proofs

It is often necessary in later parts of this document to provide proofs of
inclusion for entries in the Certificate Subtree. Such proofs are provided as
follows:

~~~ tls
struct {
  uint32 position;
  uint32 size;
  InclusionProof inclusion;
  uint32 first_valid;
  uint32 invalid_entries<0..2^8-1>;
} SubtreeInclusionProof;
~~~

The `position` field contains the index of the leaf for which inclusion is being
proven, while the `size` field contains the total number of leaves in the
Certificate Subtree. The proof in `inclusion` allows a recipient to recompute
the root hash of the Certificate Subtree, given the correct value for the leaf
at `position`.

The `first_valid` and `invalid_entries` fields duplicate the contents of the
DomainCertificates structure. This allows recipients to verify that the leaf at
`position` is not revoked, and also allows them to recompute the hash of the
DomainCertificates structure stored at a given leaf of the Prefix Tree.

Note that this document follows the pattern established in
{{!I-D.draft-ietf-keytrans-protocol}} of requiring each element of an
InclusionProof to be a full subtree. An InclusionProof may also function as a
"consistency proof" if the recipient is known to have observed a previous
version of the tree.

### Tree Heads

Transparency Logs are generally expected to add only a small number of new
entries to their Log Tree per day. This keeps proof sizes small and also ensures
that, when User Agents advertise having observed a particular tree head, there
are a large number of potential hosts that could've conveyed the tree head. To
support providing proofs of inclusion for new submissions quickly, Transparency
Logs provide proofs of inclusion against **provisional** heads. A provisional
head is conceptually a work-in-progress log entry that will, after a bounded
amount of time, be added to the rightmost edge of the Log Tree.

~~~ tls
struct {
  uint64 tree_size;
  uint32 counter;
  opaque signature<0..2^16-1>;
} ProvisionalTreeHead;
~~~

The `tree_size` field is equal to the `tree_size` field of the last
(non-provisional) TreeHead that was published plus 1.  The `counter` field
starts at zero and is incremented by 1 for each subsequent ProvisionalTreeHead
that's published without the creation of a new TreeHead.

The `signature` field of both TreeHead and ProvisionalTreeHead structures is
computed over a serialized TreeHeadTBS structure:

~~~ tls
struct {
  CipherSuite ciphersuite;
  opaque signature_public_key<0..2^16-1>;
  optional<uint64> maximum_lifetime;
} Configuration;

enum {
  reserved(0),
  standard(1),
  provisional(2),
  (255)
} TreeHeadType;

struct {
  Configuration config;

  TreeHeadType tree_head_type;
  select (TreeHeadTBS.tree_head_type) {
    case standard:
      uint64 tree_size;
    case provisional:
      uint64 tree_size;
      uint32 counter;
  }
  opaque root<Hash.Nh>;
} TreeHeadTBS;
~~~

## Endpoints

Transparency Logs expose the following endpoints over HTTP or HTTPS. Which
endpoint is being accessed is determined by the request's method and path.
Request and response bodies are specific structures, which are encoded according
to TLS format and then base64 encoded.

### Get Puzzle

GET /get-puzzle

There is no request body. The response body is:

~~~ tls
struct {
  uint16 difficulty;
  opaque server_input<0..2^8-1>;
} Puzzle;
~~~

Puzzles are used to prevent abuse of the Transparency Log's endpoints. A
solution to a puzzle is a `PuzzleSolution` structure, with the `solution` field
populated such that hashing the `PuzzleSolution` structure with the ciphersuite
hash function results in a value with `difficulty` leading zero bits.

~~~ tls
struct {
  uint64 solution;
  opaque server_input<0..2^8-1>;
} PuzzleSolution;
~~~

Clients request a puzzle before making a query to any subsequent endpoint and
provide the `PuzzleSolution` in the request body. Clients SHOULD NOT attempt to
build a reserve of puzzle solutions, or use a puzzle more than once.

### Add Certificate

POST /add-certificate

~~~ tls
struct {
  PuzzleSolution solution;
  Certificate chain<0..2^8-1>;
} AddCertificateRequest;
~~~

The request body is an `AddCertificateRequest` structure. The first certificate
in the `chain` field is a leaf certificate, the subsequent certificate is the
one that issued the leaf certificate, and so on. The final element of the array
is issued by one of the roots trusted by the Transparency Log.

~~~ tls
struct {
  TreeHead tree_head;
  CombinedTreeProof search;
  InclusionProof inclusion;
} AddCertificateResponse;
~~~



### Refreshing an Inclusion Proof
### Monitoring a Label

# TLS Extension

The following three subsections define the ClientHello, ServerHello, and Certificate
message portions of a TLS 1.3 extension. This extension allows the host server
to provide an inclusion proof for its certificate chain from a Transparency Log
that the User Agent supports.

## ClientHello

User Agents include the extension in their ClientHello to communicate which
Transparency Logs they support and whether or not they have previously observed
a provisional inclusion proof from the host.

~~~ tls
struct {
  opaque value<0..2^8-1>;
} BearerToken;

struct {
  uint16 transparency_log_id;
  TreeHeadType tree_head_type;
  select (SupportedTransparencyLog.tree_head_type) {
    case standard:
      uint64 tree_size;
    case provisional:
      BearerToken bearer_token;
  }
} SupportedTransparencyLog;

struct {
  SupportedTransparencyLog supported<0..2^8-1>;
} TransparencyRequest;
~~~

The extension has type "transparency_revocation" and consists of a serialized
`TransparencyRequest` structure in the "extension_data" field.

User Agents include an entry in the `supported` array for each Transparency Log
that they support receiving inclusion proofs from, containing the Transparency
Log's assigned unique identifier in `transparency_log_id`. The `supported` array
MUST be sorted in ascending order by `transparency_log_id`, and each
`transparency_log_id` MUST only be advertised once.

If the User Agent was shown a provisional inclusion proof in a previous
connection to the host, they will also have received a bearer token and a
pre-shared key. In all subsequent connections to the host, for as long as the
User Agent has not yet seen the provisional proof integrated into a subsequent
tree head, they advertise a `provisional` TreeHeadType and include the
provided bearer token in `bearer_token`. If the host indicates the Transparency
Log in its ServerHello extension, the User Agent will set the PSK input to the
TLS key schedule to be the pre-shared key.

If the User Agent does not need to resolve a provisional inclusion proof, they
instead advertise a `standard` TreeHeadType and include the greatest tree size
of the Transparency Log in `tree_size` where the current time (according to the
User Agent's local clock) minus the rightmost log entry's timestamp is greater
than 8 hours and less than 168 hours. If the User Agent is not aware of such a
tree size, the `tree_size` field is set to 0.

## ServerHello

As was mentioned in the previous section, hosts that receive a TLS 1.3
connection where the ClientHello has an extension of type
"transparency_revocation" with a properly formed `extension_data` field have the
option of providing an inclusion proof for their certificate chain. When the
inclusion proof builds on top of a previously-shown provisional inclusion proof,
the PSK input to the TLS key schedule must be set appropriately for the
connection to succeed.

If the host intends to provide a proof of inclusion (provisional or not) from a
Transparency Log where the client has advertised a `provisional` TreeHeadType
in its ClientHello, the host MUST have an extension of type
"transparency_revocation" in its ServerHello. The `extension_data` is the unique
identifier of the Transparency Log:

~~~ tls
uint16 transparency_log_id;
~~~

## Certificate

In the following section, describing the Certificate message extension, the host
server can either:

1. Respond with the same provisional inclusion proof, in which case the bearer
   token and pre-shared key remain unchanged.
2. Respond with a new provisional inclusion proof, in which case the host will
   set a new bearer token and pre-shared key.
3. Respond with a standard (non-provisional) inclusion proof, in which case the
   bearer token and pre-shared key would no longer be provided by the User
   Agent.


This is done by adding an extension of type "transparency_revocation" to the
first CertificateEntry structure in `certificate_list` where the
`extension_data` field is a serialized TransparencyProof structure:

~~~ tls
enum {
  reserved(0),
  standard(1),
  provisional(2),
  same_head(3),
  (255)
} TransparencyProofType;

struct {
  TreeHead tree_head;
  CombinedTreeProof combined;
  SubtreeInclusionProof subtree;
} StandardProof;

struct {
  ProvisionalTreeHead tree_head;
  CombinedTreeProof combined;

  PrefixProof prefix;
  SubtreeInclusionProof subtree;

  BearerToken bearer_token;
  opaque pre_shared_key<16>;
} ProvisionalProof;

struct {
  PrefixProof prefix;
  SubtreeInclusionProof subtree;
} SameHeadProof;

struct {
  uint16 transparency_log_id;

  TransparencyProofType proof_type;
  select(TransparencyProof.proof_type) {
    case standard:
      StandardProof proof;
    case provisional:
      ProvisionalProof proof;
    case same_head:
      SameHeadProof proof;
  };
} TransparencyProof;
~~~

The `transparency_log_id` field specifies the Transparency Log that the proof
comes from. The `proof_type` field specifies the type of proof that follows:

- `standard`: The proof of inclusion is against a log entry that is currently
  published by the Transparency Log, but more recent than the User Agent may be
  aware of.
- `provisional`: The proof of inclusion is against a log entry that is not yet
  published by the Transparency Log, but will be published within a bounded
  amount of time.
- `same_head`: The proof of inclusion is either against the same provisional
  head that originated `TransparencyRequest.bearer_token` (if present), or
  otherwise against the same version of the Transparency Log tree advertised in
  `TransparencyRequest.supported`

User Agents verify the proof by the following steps:

1. Verify that `transparency_log_id` is a known Transparency Log that was
   advertised in the ClientHello extension.
2. If `proof_type` is `standard`:
   1. Verify that `SubtreeInclusionProof.pos`

# Certificate Authority

## Poison Extension

A trivial downgrade attack is possible where a host refuses to acknowledge the
presence of the "transparency_revocation" TLS extension in an attempt to circumvent
the stronger transparency or non-revocation requirements. Customers of a
Certificate Authority can mitigate such an attack by including a special
non-critical poison extension (OID TODO, whose extnValue OCTET STRING contains
ASN.1 NULL data (0x05 0x00)) in all certificates they issue.

User Agents that advertise the "transparency_revocation" extension in their
ClientHello MUST reject a certificate that contains the extension if it is not
provided with an appropriate proof of inclusion.

## Configuration Distribution

TODO ACME extension where CA provides information about which Transparency Logs
will accept a certificate? To avoid server implementation lock-in.

# Security Considerations

## ClientHello Extension

Given that ClientHello extensions are sent unencrypted, this portion of the
extension was designed to avoid unnecessary privacy leaks. In particular, care
was taken to avoid leaking what certificate(s) the User Agent may have been
shown in previous connections and what other hosts the User Agent may have
contacted recently.

User Agents advertise a recently observed tree size for each Transparency Log
that they support receiving proofs of inclusion from. Since User Agents will
generally only "observe" various tree sizes of a Transparency Log by
communicating with hosts that provide proofs from that Transparency Log, and
since different hosts will update their proofs at different times, this may
cause a privacy leak. Specifically, it could happen that a User Agent
communicates with a host that uses proofs from a very recently-created tree
size. If the User Agent advertised this very recently-created tree size to other
hosts, it would reveal who they previously communicated with.

To mitigate this, User Agents only advertise tree sizes where the timestamp of
the rightmost log entry is at least 8 hours old. This time delay ensures that
the proof of inclusion provided by almost any host could've conveyed the same
tree size, creating a large anonymity set.

When a User Agent observes a provisional proof of inclusion from a host, they
retain condensed information about it to allow them to later verify that the
information it contained was properly integrated into the Transparency Log. The
primary avenue for obtaining this verification is advertising knowledge of the
provisional proof back to the host that it came from, hoping to get the
necessary information in-band.

Since provisional proofs of inclusion must be issued quickly, they don't have
time to build up a large anonymity set with other hosts. Instead of having User
Agents advertise knowledge of a specific provisional proof in their ClientHello,
they instead use a bearer token that was provided by the host. This bearer token
is provided in the encrypted Certificate message the first time that the
provisional proof is used by the host. Similarly, when the bearer token is
redeemed (i.e., when the host shows that the provisional proof was correctly
integrated into the Transparency Log), this information is provided in the
encrypted Certificate message. As such, assuming that the bearer token and
pre-shared key are generated by the host in a secure way, a passive network
observer never sees anything that would identify the certificate shown to the
User Agent.

Each bearer token is additionally associated with a pre-shared key which is
provided to the TLS key schedule. This prevents an active attacker from
establishing a TLS connection to the host, advertising an observed bearer token,
and learning which certificate is provided.

Finally, note that it is not a goal to prevent an attacker from learning whether
a User Agent has previously contacted a host *at all* before. The protocol
explicitly relies on the User Agent's stored state to send as little data over
the wire as possible. A passive observer of network traffic could trivially
determine from the size of the encrypted portion of the handshake messages
whether such state was present or not, and therefore whether the host had been
contacted before. Similarly, it is not a goal to prevent a host from identifying
the same User Agent over many connections.

## Downgrade Prevention

The stronger transparency and non-revocation guarantees this protocol provides
would be irrelevant if a malicious actor could cause the TLS client to disable
them at-will. An attacker may have, for example, a revoked certificate to which
they know the private key. This would allow them to fully intercept a User
Agent's connection to a host and attempt to impersonate the host. In a
downgraded version of TLS, the User Agent may not enforce revocation at all and
therefore the attacker's interception would succeed.

Site Operators that have deployed this protocol and wish to prevent capable TLS
clients from being downgraded can include a poison extension in their TLS
certificates, as described in {{poison-extension}}. The poison extension will be
silently ignored by TLS clients that genuinely do not support it, but will cause
updated clients to abort the protocol in downgrade scenarios.

Site Operators can monitor the existing {{RFC6962}} ecosystem to detect
certificates issued for their domains that lack the poison extension,
potentially permitting downgrades. However, the creation of such a certificate
without the Site Operator's consent would imply mis-issuance by a Certificate
Authority rather than abuse of a compromised/revoked certificate.


# IANA Considerations

- Codepoint for TLS extension "transparency_revocation"
- Registry for transparency_log_id(?)
- OID for poison extension

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
