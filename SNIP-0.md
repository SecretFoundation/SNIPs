---
snip: 0
title: SNIP Rationale, Types and Process
status: Active
type: Meta
author: James Waugh <james@secret.foundation>, et al.
created: 2020-09-20
---

## What is a SNIP?

SNIP stands for Secret Network Improvement Proposal. A SNIP is a design document providing information to the Secret Network community, describing a new standard or a potential change to a process or environment. The SNIP should provide a concise technical specification of the improvement and its rationale. The SNIP author is responsible for building consensus within the community and documenting opinions.

## SNIP Rationale

We intend SNIPs to be the primary mechanisms for proposing new standards and gathering technical input on relevant issues.

## SNIP Types

There are three types of SNIP:

- A **Standard SNIP** describes any change that affects Secret Network, or any change that affects interoperability of applications using Secret Network. This particular type of SNIP consists of three parts, a design document, implementation, and finally (if warranted) an update to the [Secret Network codebase](https://github.com/enigmampc/SecretNetwork).
  - **Networking** - includes proposed improvements to Secret Network protocol specifications.
  - **Interface** - includes improvements around client API/RPC specifications and standards.
  - **Standards** - application-level standards and conventions, including contract standards such as token standards, name registries, URI schemes, library/package formats, and wallet formats.
- A **Meta SNIP** describes a process related to Secret Network or proposes a change to a process. Meta SNIPs are like Standard SNIPs, but they apply to areas other than the Secret Network protocol itself. They may propose an implementation, but not in Secret Network's codebase; they often require community consensus; unlike Informational SNIPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Secret Network development.
- An **Informational SNIP** describes an Secret Network design issue, or provides general guidelines or information to the Secret Network community, but does not propose a new feature. Informational SNIPs do not necessarily represent Secret Network community consensus or a recommendation, so users and implementers are free to ignore Informational SNIPs or follow their advice.

A SNIP must meet certain minimum criteria. It must be a clear and complete description of the proposed enhancement. The enhancement must represent a net improvement. The proposed implementation, if applicable, must be solid and must not complicate the protocol unduly.

## SNIP Work Flow

### Championing an SNIP

Parties involved in the process are you, the champion or *SNIP author*, the [*SNIP editors*](https://github.com/SecretFoundation/SNIPs/graphs/contributors), and the [*Secret Network Core Developers*](https://github.com/enigmampc/SecretNetwork/graphs/contributors).

Before you begin writing a formal SNIP, you should vet your idea. First, ask the community if an idea is original to avoid wasting time on something that will be be rejected based on prior research. It is thus recommended to open a discussion thread on [the Secret Forum](https://forum.scrt/network) to do this, but you can also ask in our [Secret Chat](https://chat.scrt.network), or in [the issues of this repository](https://github.com/SecretFoundation/SNIPs/issues). 

In addition to making sure your idea is original, it will be your role as the author to make your idea clear to reviewers and interested parties, as well as inviting editors, developers and community to give feedback on the aforementioned channels.

### SNIP Process 

Following is the process that a successful SNIP will move along:

```
[ IDEA ] -> [ DRAFT ] -> [ ACCEPTED ] -> [ FINAL ]
```

Each status change is requested by the SNIP author and reviewed by the SNIP editors. Use a pull request to update the status. Please include a link to where people should continue discussing your SNIP. Requests will be processed according to these conditions:

* **Idea** -- Once the champion has asked the Secret Network community whether an idea has any chance of support, they will write a draft SNIP as a [pull request]. Consider including an implementation if this will aid people in studying the SNIP.
  * Approved -- If agreeable, a SNIP editor will assign the SNIP a number/name (generally the issue or PR number related to the SNIP) and merge your pull request. The SNIP editor will not unreasonably deny a SNIP.
  * Rejected -- Reasons for denying draft status include being too unfocused, too broad, duplication of effort, being technically unsound, not providing proper motivation or addressing backwards compatibility, or misalignment with [Secret Network](https://scrt.network).
* **Draft** -- Once the first draft has been merged, you may submit follow-up pull requests with further changes to your draft until such point as you believe the SNIP to be mature and ready to proceed to the next status. An SNIP in draft status must be implemented to be considered for promotion to the next status. 
* **Accepted** -- This status signals that material changes are unlikely and Secret Network developers should consider this SNIP for inclusion. Their process for deciding whether to encode it into their clients as part of a hard fork is not part of the SNIP process.
* **Final** -- Standards Track Core SNIPs must be implemented. When the implementation is complete and adopted by the community, the status will be changed to “Final”.

Other exceptional statuses include:

* **Active** -- Informational and Meta SNIPs may have a status of “Active” if they are never meant to be completed, e.g., SNIP-0 (this one).
* **Reverted to Draft** -- The Core Devs can decide to moves accepted SNIPs back to the Draft status at their discretion.
* **Abandoned** -- This SNIP is no longer pursued by the original authors or it may not be a (technically) preferred option anymore.
* **Rejected** -- A SNIP that is fundamentally broken or was rejected by the Core Devs and will not be implemented.
* **Superseded** -- A SNIP which was previously Final but is no longer considered state-of-the-art. Another SNIP will be in Final status and reference the Superseded SNIP.

## What belongs in a successful SNIP?

Each SNIP should have the following parts:

- Preamble - RFC 822 style headers containing metadata about the SNIP, including the SNIP number, a short descriptive title (limited to a maximum of 42 characters), and the author details. See [below](https://github.com/SecretFoundation/SNIPs/blob/master/SNIPs/snip-0.md#snip-header-preamble) for details.
- Abstract - A short (~200 word) description of the technical issue being addressed.
- Motivation - It should clearly explain the problem solved by the proposal. SNIPs without sufficient motivation may be rejected outright.
- Specification - The technical specification should describe the syntax and semantics of any new feature or standard.
- Rationale - The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.
- Backwards Compatibility - All SNIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SNIP must explain how the author proposes to deal with these incompatibilities. SNIPs without a sufficient backwards compatibility treatise may be rejected outright.
- Test Cases - Mandatory for SNIPs that are affecting consensus changes. Other SNIPs can choose to include links to test cases if applicable.
- Implementations - Must be completed before any SNIP is given status “Final”, but need not be completed before the SNIP is merged as draft. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of “rough consensus and running code” is still useful when it comes to resolving many discussions of API details.
- Security Considerations - All SNIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. SNIPs missing the "Security Considerations" section will be rejected. A SNIP cannot proceed to status "Final" without a discussion about Security Considerations that is deemed sufficient by the reviewers.
- Copyright Waiver - All SNIPs must be in the public domain. See the bottom of this SNIP for an example copyright waiver.

## SNIP Formats and Templates

SNIPs should be written in [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) format. Here is a [template](https://github.com/SecretFoundation/SNIPs/blob/master/snip-template.md).

## SNIP Header Preamble

Each SNIP must begin with an [RFC 822](https://www.ietf.org/rfc/rfc822.txt) style header preamble, preceded and followed by three hyphens (`---`). This header is also termed ["front matter" by Jekyll](https://jekyllrb.com/docs/front-matter). The headers must appear in the following order. Headers marked with "*" are optional and are described below. All other headers are required.

` snip:` *SNIP number/name* (this is determined by the SNIP editor)

` title:` *SNIP title*

` author:` *a list of the author's or authors' name(s) and/or username(s), or name(s) and email(s). Details are below.*

` * discussions-to:` *a url pointing to the official discussion thread*

` status:` *Draft, Accepted, Final, Active, Abandoned, Rejected, or Superseded*

`* review-period-end:` *date review period ends*

` type:` *Standard, Meta, or Informational*

` created:` *date of creation*

` * updated:` *comma separated list of dates*

` * requires:` *SNIP number(s)*

` * replaces:` *SNIP number(s)*

` * superseded-by:` *SNIP number(s)*

` * resolution:` *a url pointing to the resolution of this SNIP*

Headers that permit lists must separate elements with commas.

Headers requiring dates will always do so in the format of ISO 8601 (yyyy-mm-dd).

#### `author` header

The `author` header optionally lists the names, email addresses or usernames of the authors/owners of the SNIP. Those who prefer anonymity may use a username only, or a first name and a username. The format of the author header value must be:

> Random J. User &lt;address@dom.ain&gt;

or

> Random J. User (@username)

if the email address or GitHub username is included, and

> Random J. User

if the email address is not given.

#### `discussions-to` header

While a SNIP is a draft, a `discussions-to` header will indicate the mailing list or URL where the SNIP is being discussed. As mentioned above, examples for places to discuss your SNIP include our [Secret Forum](https://forum.scrt.network) and [Secret Chat](https://chat.scrt.network), an issue in this repo or in a fork of this repo.

No `discussions-to` header is necessary if the SNIP is being discussed privately with the author.

As a single exception, `discussions-to` cannot point to GitHub pull requests.

#### `type` header

The `type` header specifies the type of SNIP: Standards Track, Meta, or Informational.

#### `category` header

The `category` header specifies the SNIP's category. This is required for standards-track SNIPs only.

#### `created` header

The `created` header records the date that the SNIP was assigned a number

Both headers should be in yyyy-mm-dd format, e.g. 2020-02-13.

#### `updated` header

The `updated` header records the date(s) when the SNIP was updated with "substantial" changes. This header is only valid for SNIPs of Draft and Active status.

#### `requires` header

SNIPs may have a `requires` header, indicating the numbers of SNIPs on which this SNIP depends.

#### `superseded-by` and `replaces` headers

SNIPs may also have a `superseded-by` header indicating that a SNIP has been rendered obsolete by a later document; the value is the number of the SNIP that replaces the current document. The newer SNIP must have a `replaces` header containing the number of the SNIP that it rendered obsolete.

## Auxiliary Files

Images, diagrams and auxiliary files should be included in a subdirectory of the `assets` folder for that SNIP as follows: `assets/snip-N` (where **N** is the SNIP number). When linking to an image in the SNIP, use relative links such as `../assets/snip-0/image.png`.

## Transferring SNIP Ownership

It occasionally becomes necessary to transfer ownership of SNIPs to a new champion. In general, we'd like to retain the original author as a co-author of the transferred SNIP, but that's really up to the original author. A good reason to transfer ownership is because the original author no longer has the time or interest in updating it or following through with the SNIP process, or has fallen off the face of the 'net (i.e. is unreachable or isn't responding to email). A bad reason to transfer ownership is because you don't agree with the direction of the SNIP. We try to build consensus around an SNIP, but if that's not possible, you can always submit a competing SNIP.

If you are interested in assuming ownership of an SNIP, send a message asking to take over, addressed to both the original author and the SNIP editor. If the original author doesn't respond to email in a timely manner, the SNIP editor will make a unilateral decision.

## Style Guide

When referring to an SNIP by number, it should be written in the hyphenated form `SNIP-X` where `X` is the SNIP's assigned number.

## History

This document was derived from [Ethereum's EIP-1](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1.md), authored by Martin Becze, Hudson Jameson, et al. That document was derived from [Bitcoin's BIP-0001](https://github.com/bitcoin/bips), which in turn was derived from [Python's PEP-0001](https://www.python.org/dev/peps). In many places, text was copied and modified. Please direct all comments to the SNIP editors.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0).
