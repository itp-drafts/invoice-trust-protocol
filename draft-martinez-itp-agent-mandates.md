---
title: "Invoice Trust Protocol (ITP): Agent Mandates"
abbrev: "ITP Agent Mandates"
category: exp
docname: draft-martinez-itp-agent-mandates-latest
submissiontype: IETF
number:
date: 2026-10-28
consensus: true
v: 3
keyword:
 - electronic invoicing
 - software agents
 - delegation
 - SD-JWT
venue:
  group: "DISPATCH"
  type: "Working Group"
  mail: "dispatch@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dispatch/"
author:
 -
    fullname: "Carlos Martínez García-Villarrubia"
    asciiFullname: "Carlos Martinez Garcia-Villarrubia"
    initials: "C."
    surname: "Martínez García-Villarrubia"
    asciiSurname: "Martinez Garcia-Villarrubia"
    organization: "BeeL. (Honey Solutions, S.L.)"
    city: Tarragona
    country: Spain
    email: carlos@martinezgv.com

normative:
  I-D.martinez-itp-core:
    title: "Invoice Trust Protocol (ITP): Discovery, Delivery, and Verification of Electronic Invoices"
    author:
      - ins: C. Martínez García-Villarrubia
        name: Carlos Martínez García-Villarrubia
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-martinez-itp-core-00
  I-D.martinez-itp-lifecycle:
    title: "Invoice Trust Protocol (ITP): Lifecycle Status and Payment Information"
    author:
      - ins: C. Martínez García-Villarrubia
        name: Carlos Martínez García-Villarrubia
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-martinez-itp-lifecycle-00
  I-D.ietf-oauth-sd-jwt-vc:

informative:
  OPENID4VP:
    title: "OpenID for Verifiable Presentations 1.0"
    author:
      - org: OpenID Foundation
    date: 2025
    target: https://openid.net/specs/openid-4-verifiable-presentations-1_0.html
  AP2:
    title: "Agent Payments Protocol (AP2)"
    author:
      - org: Google LLC
    date: 2025
    target: https://ap2-protocol.org/ap2/specification/

--- abstract

This document lets a business authorize a software agent to issue,
accept, reject, or dispute invoices, and to report payments, through
the Invoice Trust Protocol (ITP), within limits that the business sets.
An authorization is a mandate: a Selective Disclosure JSON Web Token
(SD-JWT) credential issued by the
business to a key held by the agent and enforced by a named ITP node
that records each use.

--- middle

# Introduction

Some purchasing and accounts-payable software lets automated agents
send invoices and status messages.  A counterparty that receives one
needs to know which business the agent acts for and whether the action
is within the agent's authority.  The business that deployed the
agent needs its limits enforced even when the agent misbehaves, for example because text in an invoice it processed changed its behavior.

The core of the Invoice Trust Protocol (ITP) {{I-D.martinez-itp-core}} defines the "agent" signer
role and the "agent" member of the envelope core but leaves their
content to this document.  This document defines:

* the mandate, an SD-JWT VC {{I-D.ietf-oauth-sd-jwt-vc}} that binds an
  agent key to a participant and to a set of permitted actions and
  limits ({{mandate}});

* how an agent signs envelopes and status messages and proves
  possession of its key for each one ({{use}});

* enforcement of limits by a single node, which prevents the same
  budget from being spent through several nodes ({{enforcement}});

* attenuated delegation from one agent to another ({{delegation}});
  and

* approval by a person above a threshold ({{approval}}).

Payment through HTTP-based payment protocols, which raises anti-money-laundering, sanctions-screening, and payee-verification questions, is out of scope.

OAuth 2.0 Rich Authorization Requests {{?RFC9396}} can express limits
such as amounts, and OAuth 2.0 Token Exchange {{?RFC8693}} can record
a chain of delegation.  Both describe what an authorization server
grants to a client for access to a resource.  A mandate is instead
issued by a participant with its own keys, presented to counterparties
that do not share an authorization server with it, and enforced by a
node that records each use.  The Agent Payments Protocol {{AP2}} also
uses mandates, as verifiable digital credentials, for payments by
agents; ITP mandates cover invoices and their statuses and do not give
access to payment instruments.  Frameworks for agent identity such as
{{?I-D.ietf-wimse-aims}} identify an agent as a workload and
authenticate it to services; how an agent authenticates to its
enforcing node is outside this document, and the key bound by a
mandate can be one that such a framework manages.  A mandate states
which business the agent acts for and within which limits, which a
workload identity does not.

## The Experiment

This document is Experimental.  The experiment is whether at least two
independent implementations of ITP nodes enforce mandates issued by
each other's participants, using the mandate in {{example-claims}}, and
whether the limits defined here are enough for the purchasing and
accounts-payable uses that implementers report.

Results will be reported on the mailing list where this document is
discussed and in later revisions of it.  The experiment ends 24 months
after publication, together with that of {{I-D.martinez-itp-core}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms defined in {{I-D.martinez-itp-core}} and
{{I-D.martinez-itp-lifecycle}} are used with the same meaning.  In
addition:

Principal:
: The participant that issues a mandate.

Agent:
: Software that holds the private key bound by a mandate and acts for
  the principal.

Enforcing node:
: The ITP node named in the mandate that checks and records every use
  of it.

# Mandates {#mandate}

A mandate is an SD-JWT VC {{I-D.ietf-oauth-sd-jwt-vc}}, with "typ"
"dc+sd-jwt", whose issuer-signed JWT is signed with a key from the
principal's accepted Participant Document ({{I-D.martinez-itp-core}}, "Trust and Keys") that is not listed in "delegates".  An
operator that holds only delegated keys therefore cannot authorize an
agent on its own.  This is the issuer key discovery mechanism for
mandates in the sense of Section 2.5 of {{I-D.ietf-oauth-sd-jwt-vc}};
JWT VC Issuer Metadata is not used.  The claims are:

iss (required):
: The principal's PID in URN form ("urn:ietf:params:itp:pid:...").

sub (required):
: An identifier of the agent chosen by the principal, unique among
  its mandates.  It SHOULD NOT identify a natural person.

vct (required):
: "urn:ietf:params:itp:mandate".

cnf (required):
: The agent's public key, as a "jwk" confirmation method {{!RFC7800}}.
  The JWK contains "alg".

aud (required):
: The Node Document URL of the enforcing node.  Counterparties and
  other relying parties process the mandate without being named in
  "aud"; for mandates, this overrides the rejection rule of
  {{Section 4.1.3 of RFC7519}}.  That node MUST be
  listed in "senders" of the principal's Participant Document, MUST
  have a key listed in "delegates" of the principal's Participant
  Document with that node as "node", and MUST be the
  node through which the principal signs status messages, so that the
  principal's chain of status messages does not fork.

iat, nbf, exp (required):
: As in {{!RFC7519}}.  Mandates SHOULD NOT be valid for more than 30
  days.

status (required):
: A reference to a status list entry {{!I-D.ietf-oauth-status-list}},
  so that the principal can revoke the mandate at once.  The Status
  List Token is a JWT signed with a key of the principal that is not
  listed in "delegates", or with a key listed in "kids" of the
  principal's "senders" entry for the enforcing node.  For a delegated
  mandate, the enforcing node provides the status list entry.

actions (required):
: An array of values from the ITP Mandate Actions registry
  ({{iana-actions}}); the initial values are in {{actions-table}}.

limits (required):
: An object with "currency" and any of: "per_document" (maximum
  amount of one document or payment), "per_period" (maximum total
  within "period", which is then required), "period" (a duration of whole days or weeks in the "duration"
  grammar of {{Appendix A of !RFC3339}}, for example "P30D" or "P1W"), "counterparties"
  (an array of PIDs the agent may deal with; a counterparty matches
  when a PID verified for its accepted Participant Document equals an
  entry), "document_types"
  (values of the ITP Document Types registry), and "approval_above"
  (an amount above which every action needs approval by a person,
  {{approval}}).  Amounts are "amount" values as defined in
  {{I-D.martinez-itp-core}}.  Limits are compared with, and
  reservations made for, the absolute value of the amount of an
  action, so that a negative total, as in some credit notes, never
  increases the remaining budget.

max_delegation (optional):
: The number of further delegations allowed ({{delegation}}).  The
  default is 0.

software (optional):
: An object that describes the software in which the agent runs, as
  stated by the principal: "software_id" and "software_version", with
  the meanings given to these names in {{Section 2 of ?RFC7591}}, and
  "provider", the name of the organization that supplies the software.
  It lets the principal and the enforcing node find and revoke every
  mandate held by a given product or version, for example after a
  vulnerability in it becomes known.  It is not evidence that the
  agent runs that software: relying parties MUST NOT base trust
  decisions on it.  The model or other components that the software
  uses change more often than mandates and are not described here; a
  deployment that needs them records them in its own logs.

Claims not registered in the "JSON Web Token Claims" registry, such
as "actions", "limits", "max_delegation", "software", "principal",
and "parent",
are private claim names ({{Section 4.3 of RFC7519}}) whose meaning is
given by this document for the "vct" values
"urn:ietf:params:itp:mandate" and
"urn:ietf:params:itp:mandate-delegated" only.

| Action | Permits |
|---|---|
| issue | Issuing envelopes of the document types in "limits" |
| accept | Status "accepted" |
| reject | Status "rejected" |
| dispute | Status "disputed" |
| pay | Status "payment-sent" |
{: #actions-table title="Initial mandate actions"}

None of these actions gives access to a payment account.  The "pay"
action lets an agent report a payment it has made by other means, and
reserves the amount against its limits.

Claims other than "iss", "vct", "cnf", "aud", the time claims, and
"status" MAY be selectively disclosable, so that a principal can avoid
showing all its limits to every counterparty.  A counterparty sees the
disclosures that the agent presents and relies on the enforcing node
for the rest.  Because the presentation in the "agent" member ({{use}}) is the one
that counterparties receive, the principal, or for a delegated mandate
the agent that issued it, gives the enforcing node the mandate with
all its disclosures, by means outside this document.

# Using a Mandate {#use}

An agent signs an envelope or status message with a signature whose
protected header has "itp_role" "agent", "crit" including "itp_role",
"alg", "typ", and "kid" set to the JWK SHA-256 Thumbprint
{{!RFC7638}} of the key in "cnf".
The payload then contains:

~~~ json
{
  "agent": {
    "mandate": "<SD-JWT with selected disclosures and KB-JWT>",
    "enforced_by": "https://node.example.com/itp",
    "use": "01a18210-0000-7000-8000-000000000001"
  }
}
~~~

"use" is a UUIDv7 chosen by the agent for this use.  The enforcing node
rejects it if its timestamp differs from the node's clock by more than
300 seconds, and keeps used values for at least 600 seconds to reject
reuse.  The Key Binding JWT ({{!RFC9901}}) of the presented SD-JWT has "aud" set to
the enforcing node and "nonce" set to a value that the enforcing node issued to the agent
and accepts only once, so it proves freshness; how the agent obtains
the nonce, like how it authenticates to the node, is outside this
document.  The Key Binding JWT otherwise follows {{Section 4.3 of RFC9901}}.  Its "sd_hash" covers the presented SD-JWT.  The agent
signature, made with the same key, covers the payload that contains
the presentation, and so binds the presentation to the action.

Status messages signed by an agent use the General JSON
Serialization, so that they can carry the agent signature and the
signatures added by the enforcing node.

The agent sets every member of the envelope core, including "id",
"issuer_node", and "transparency_salt", or of the status message,
including "seq" and "prev", before it signs.  The enforcing node, as
the principal's issuing node, refuses with "mandateInvalid" a payload
in which they do not meet {{I-D.martinez-itp-core}}, for example a
status message that does not continue the principal's chain.

An "agent" signature is never accepted in place of an issuer
signature ({{I-D.martinez-itp-core}}).  For issuance, the enforcing
node adds a "delegated-issuer" signature after it has checked the
mandate, so the envelope still has an issuer-level signature, and an
"operator-seal" signature that records the use.  For status messages
it adds the same two signatures.

# Enforcement {#enforcement}

Only the enforcing node named in "aud" can authorize a use of a
mandate.  Before it signs or forwards anything for the agent, it MUST:

1. verify the mandate signature against a key of the principal's
   accepted Participant Document that is not listed in "delegates" or, for a delegated mandate, against the
   "cnf" key of its parent, recursively up to a mandate signed by the
   principal, and check the validity period and status of each
   mandate in the chain;

2. verify the Key Binding JWT and its nonce, and the agent signature,
   and check "use";

3. check that the principal (the "iss" of the root mandate) is the
   "issuer" of the envelope or the "by" of the status message, that
   the principal has the signer role that
   {{I-D.martinez-itp-lifecycle}} gives for the status, that the action
   is in "actions", and that the document and counterparty meet
   "limits", for every mandate in the chain; the
   counterparty is the "recipient" for "issue" and, for status
   messages, whichever of "seller" and "buyer" is not the principal;

4. refuse the action with "mandateInvalid" if the currency of its
   amount (the envelope "currency", or the "currency" of the status
   message, which defaults to it) differs
   from "limits.currency", and otherwise atomically reserve the amount
   against "per_period" of every mandate in the chain; a principal
   that acts in several currencies issues one mandate per currency;
   and

5. if the amount exceeds "approval_above", hold the action until an
   approval is received ({{approval}}), answering with status 202 as
   for asynchronous results in {{I-D.martinez-itp-core}}.

The enforcing node also holds for approval an "accept" or "pay"
action on an envelope signed with a recently bound key
({{I-D.martinez-itp-core}}), or from an issuer that is not in
"limits.counterparties" and has not issued an envelope to the
principal before, whatever the amount.

The enforcing node keeps a tamper-evident record of each use, with
its "use" identifier, the "sub" and "cnf" key thumbprint of each
mandate in the chain, the principal, the action, the digest of the
envelope or status message, the decision, its reason and time, any
approval, and the "software" claim if present, for at least as long
as the principal retains the documents concerned.  This is the record
that lets the principal and auditors establish which agent did what,
in line with the audit records that agent identity frameworks such as
{{?I-D.ietf-wimse-aims}} require.  A request whose
envelope core or status message payload, with the "agent" member
removed, equals that of a recorded use is a retry, which the agent sends with
a new "use" value and nonce: the node returns
the result of the recorded use and reserves nothing more.  A request
with the envelope or status message "id" of a recorded use and a payload that, with the "agent" member removed, differs from that of the recorded use is refused
with "mandateInvalid".  The enforcing node's
"operator-seal" signature covers the payload, and so the "agent"
member and the "use" identifier.

Reservations are kept per mandate, envelope, and use.  "issue"
reserves the "total_with_tax" of the envelope and is released when the state of
the envelope ({{I-D.martinez-itp-lifecycle}}) shows delivery
"technical-rejected", fiscal "clearance-rejected", cancellation
"cancelled", or acceptance "rejected" or "deemed-rejected".  "accept"
reserves the "amount" of the "accepted" message or, if it has none,
the "total_with_tax" of the envelope, and is released when acceptance becomes
"rejected" or "deemed-rejected", which includes the case in which the
"accepted" message is void ({{I-D.martinez-itp-lifecycle}}).  "pay"
reserves the amount of the "payment-sent" message and is released only
by a "payment-failed" that refers to it and is signed by the principal
or by a bridge; a "payment-failed" signed by the payee does not
release it.  "reject" and "dispute" reserve nothing.  A "pay" reservation for an
envelope on which the same mandate holds an "accept" reservation
reduces that "accept" reservation by the amount paid, so that accepting
and paying one invoice is counted once.  A reservation is
never released because time has passed, since a payment may already
have been made.  "per_period" is a window that ends at the time of the
reservation, and a reservation counts toward the period in which the
use occurred.  A reservation is held while none of its release
conditions holds in the current state, and is held again if the state
changes so that none holds.

A counterparty node that receives an envelope or status message with
an "agent" member MUST reject it with "mandateInvalid" unless: it
carries an "operator-seal" made with a key listed in "kids" of the
principal's "senders" entry for "enforced_by"; that node is the "aud" of the mandate; the root
mandate's "iss" (or "principal") is the URN of the envelope's
"issuer" or of the status message's "by"; the agent signature
verifies with the key in "cnf"; the root mandate verifies with a key
of the principal's accepted Participant Document that is not listed in
"delegates", each delegated mandate verifies with the "cnf" key of its
parent, and every mandate in the chain was valid at the signing time;
and the action is among the disclosed
"actions".

Because each mandate has a single enforcing node, the same budget
cannot be spent twice by sending through two nodes.  A principal that
wants an agent to work through several nodes issues one mandate per
node, each with its own limits.

# Delegation {#delegation}

An agent whose mandate has "max_delegation" greater than 0 can issue
a mandate to another agent, signed with the key in its own "cnf".  A
delegated mandate has "vct" "urn:ietf:params:itp:mandate-delegated",
"iss" set to "urn:ietf:params:oauth:jwk-thumbprint:sha-256:" followed
by the thumbprint of the parent's "cnf" key {{!RFC9278}}, and a
"principal" claim with the principal's PID in URN form.  It has the
same "principal" and "aud" as its parent, "actions" and "limits" that
are equal to or narrower than the parent's, an "exp" no later than the
parent's, and "max_delegation" lower than the parent's.  A limit that
is absent from the delegated mandate is inherited from the parent.

The delegated mandate carries its parent, with all its disclosures, in
a "parent" claim that is never selectively disclosable, so that the
enforcing node can check that the limits are narrower.  The enforcing
node checks the whole chain and counts every use against every
mandate in it, and SHOULD refuse chains longer than five mandates.

# Approval by a Person {#approval}

When "approval_above" is exceeded, or when the principal's policy at
the enforcing node requires it, the node asks the principal for
approval.  An approval is a JWS in the Flattened JSON Serialization with "typ" "itp-approval+jws", signed
with a key of a natural person whom the principal has authorized and
registered with the enforcing node; how that registration is done is
local configuration.  Its payload contains "mandate_sub", "action",
"envelope_digest" (or the status message digest), "amount",
"currency", and "iat".  The principal can instead approve through a
verifiable presentation, for example with {{OPENID4VP}}, whose
transaction data (Appendix B.3.3 of {{OPENID4VP}}) carries the same
members, so that the presentation is bound to the action.  The enforcing
node keeps the approval with the record of the use.  ITP status
messages are never treated as approvals.  The enforcing node shows the
approver the action as stated in the signed envelope core or status
message, and the document, and not a description written by the
agent, since an agent manipulated by content it processed can
describe an action misleadingly.

# Implementation Status

Note to the RFC Editor: please remove this section before
publication, as described in {{?RFC7942}}.

Organization: BeeL.&nbsp;(Honey Solutions, S.L.).  Description: support for
this document in the reference implementation listed in
{{I-D.martinez-itp-core}}.  Maturity: in development.  License:
Apache License 2.0 (intended).  Contact: the author.

No independent implementation is known at the time of writing.

# Security Considerations

The security considerations of {{I-D.martinez-itp-core}} and
{{I-D.martinez-itp-lifecycle}} apply.

A mandate limits what a compromised agent can do.  Text in invoices,
reason fields, and attachments is written by counterparties and can
try to change an agent's behavior.  Because enforcement happens at
the node, outside the agent, such text cannot raise the agent's
limits; a manipulated agent can still act within its limits.  For example, it can split an amount into several
documents or payments below "per_document" and "approval_above",
which only "per_period" bounds, and it can reject or dispute any
envelope, since these actions reserve nothing.  Principals that need
tighter control use "counterparties", short validity periods, or a
lower "approval_above", and enforcing nodes MAY limit the rate of
actions per mandate.

Mandates are signed only with keys that the principal holds, so an
operator cannot issue mandates to itself.  The enforcing node is
nevertheless trusted with the limits: a compromised enforcing node can
approve any action that the principal's "delegated-issuer" key can
sign.  Principals review the uses it records and monitor the
transparency logs where its envelopes are registered.

Revocation takes effect when relying parties refresh the status list.
Enforcing nodes SHOULD check the status of every mandate in the chain
at each use, using a Status List Token no older than five minutes, and
an enforcing node that provides the status list applies a revocation
as soon as it records it; counterparties rely on
the enforcing node for timely revocation.

The Key Binding JWT names the enforcing node and contains a nonce issued by that node, and the agent signature covers the payload that contains it, which prevents replay of either.  The reservation is atomic, so concurrent
requests cannot exceed "per_period", and it is released only by a signed status, so the amount of a payment already made cannot be reused.

Selective disclosure hides limits from counterparties, so a
counterparty cannot check them itself and depends on the
"operator-seal" of the enforcing node.  Counterparties that do not
trust that node can refuse envelopes signed by agents.

# Privacy Considerations

A mandate reveals the principal's PID and, through disclosed claims,
part of its purchasing policy.  A disclosed "software" claim reveals
which product the principal uses; principals can keep it undisclosed
to counterparties, since only the enforcing node needs it.  The enforcing node learns every action
of the agent, which it needs to enforce limits.  A delegated mandate carries
its parent with all its disclosures, so a counterparty that receives
it sees every limit of the parent.  Approvals identify
the person who approved, and are kept only by the enforcing node and
the principal.

# Internationalization Considerations

Mandates and approvals contain no human-readable text.  Amounts and currencies follow {{I-D.martinez-itp-core}}.

# IANA Considerations

## Media Type

IANA is requested to register the following media type in the "Media
Types" registry, following {{!RFC6838}}:

Type name:
: application

Subtype name:
: itp-approval+jws

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary; JWS JSON Serialization, which is UTF-8 JSON

Security considerations:
: Signed JSON without active content.  Integrity and origin come from
  the JWS signature, checked as described in {{approval}}.  See the
  Security Considerations of this document.

Interoperability considerations:
: N/A

Published specification:
: This document

Applications that use this media type:
: Implementations of ITP agent mandates

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names: N/A; Magic number(s): N/A; File
  extension(s): N/A; Macintosh file type code(s): N/A

Person and email address to contact for further information:
: IETF (iesg@ietf.org)

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: Carlos Martinez Garcia-Villarrubia

Change controller:
: IETF

## ITP URN Values

IANA is requested to register "mandate" ("Credential type of ITP agent
mandates") and "mandate-delegated" ("Credential type of delegated ITP
agent mandates") in the ITP URN Values registry, with this document as
the reference.

## ITP Error Types

IANA is requested to register "mandateInvalid", with HTTP status 403,
the description "Agent mandate missing, invalid, or exceeded", and
this document as the reference, in the ITP Error Types registry.

## ITP Signer Roles

IANA is requested to add this document as a reference for the value
"agent" in the ITP Signer Roles registry created by
{{I-D.martinez-itp-core}}.

## ITP Mandate Actions {#iana-actions}

IANA is requested to create the "ITP Mandate Actions" registry in the
"Invoice Trust Protocol (ITP)" registry group, with the "Specification
Required" policy {{!RFC8126}}.  Each entry has a name, a description
of what it permits, and a reference.  The initial entries are those of
{{actions-table}}.  Designated experts check that a new action says
exactly which statuses or envelope members it permits, that it cannot
be read as granting access to a payment account, and that its
specification has Security Considerations.

--- back

# Worked Example {#worked-example}

This appendix is informative.  It follows one use of the mandate in
{{example-claims}}.

1. A business registered for VAT in the United Kingdom deploys an
   accounts-payable agent.  In its node's administration interface it
   creates the agent's key pair, or registers a key that the agent
   software generated, and signs a mandate for it with a key of its
   Participant Document that is not listed in "delegates".  The mandate allows "accept", "reject", and
   "dispute", up to 5000.00 GBP per document and 40000.00 GBP in 30
   days, for one supplier, with approval by a person above 2500.00
   GBP.  The business's node is the enforcing node.

2. The supplier delivers an envelope with an invoice of 3000.00 GBP
   to that node ({{I-D.martinez-itp-core}}).

3. The agent reads the invoice and decides to accept it.  It obtains a
   nonce from the enforcing node, creates the "accepted" status
   message with the "agent" member ({{use}}), and signs it with its
   key.

4. The enforcing node runs the checks in {{enforcement}}.  The
   mandate is valid, the action is allowed, the supplier is in
   "counterparties", and 3000.00 GBP fits both limits, so the node
   reserves the amount.  Because it exceeds "approval_above", the node
   answers with status 202 and asks an authorized person to approve,
   showing the signed status message and the invoice.

5. After the approval ({{approval}}), the node adds its
   "delegated-issuer" and "operator-seal" signatures, records the
   use, and sends the status message to the supplier's node.

6. The supplier's node checks the "agent" member as described at the
   end of {{enforcement}} and counts the status message as an
   acceptance by the business ({{I-D.martinez-itp-lifecycle}}).

Had the invoice contained text that led the agent to accept 50000.00
GBP, step 4 would have refused the action, because the limits are
checked by the node and not by the agent.

# Example Mandate Claims {#example-claims}

~~~ json
{
  "iss": "urn:ietf:params:itp:pid:tax:GB:vat:000000000",
  "sub": "ap-agent-3",
  "vct": "urn:ietf:params:itp:mandate",
  "cnf": {"jwk": {"kty": "OKP", "crv": "Ed25519", "alg": "Ed25519",
                  "x": "<agent public key>"}},
  "aud": "https://node.example.com/itp",
  "iat": 1793180000, "nbf": 1793180000, "exp": 1795772000,
  "status": {"status_list": {"idx": 412,
    "uri": "https://node.example.com/itp/status/1"}},
  "actions": ["accept", "reject", "dispute"],
  "software": {"software_id": "4NRB1-0XZABZI9E6-5SM3R",
               "software_version": "2.1",
               "provider": "Example AP Ltd"},
  "limits": {"currency": "GBP", "per_document": "5000.00",
             "per_period": "40000.00", "period": "P30D",
             "approval_above": "2500.00",
             "counterparties": ["eas:0088:0000000000000"]}
}
~~~

# Acknowledgments
{:numbered="false"}

The author used AI-based tools while drafting and reviewing this
document.  The author checked all text and references and is
responsible for the content.
