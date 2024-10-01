---
title: "The MLS Replace Proposal"
abbrev: "MLS Replace"
category: info

docname: draft-barnes-mls-replace-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Messaging Layer Security"
keyword:
 - messaging layer security
 - end-to-end encryption
 - post-compromise security
venue:
  group: "Messaging Layer Security"
  type: "Working Group"
  mail: "mls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mls/"
  github: "bifurcation/mls-replace"
  latest: "https://bifurcation.github.io/mls-replace/draft-barnes-mls-replace.html"

author:
 -
    fullname: "Richard Barnes"
    organization: Cisco
    email: "rlb@ipv.sx"
 -
    fullname: "Marta Mularczyk"
    organization: Amazon
    email: "mulmarta@amazon.com"
 -
    fullname: "Mark Xue"
    organization: Germ Network, Inc.
    email: "mark@germ.network"

normative:

informative:


--- abstract

Post-compromise security is one of the core security guarantees provided by the
Messaging Layer Security (MLS) protocol.  MLS provides post-compromise security
for a member when the member's leaf node in the MLS ratchet tree is updated,
either by that member sending a Commit message, or by an Update proposal from
that member being committed.  Unfortunately, Update proposals can only be
committed in the epoch in which they are sent, leading to missed opportunities
for post-compromise security.  This document defines a Replace proposal that
allows the fresh leaf node in an Update proposal to be applied in a future
epoch, thus enabling post-compromise security for the affected member even if
their Update proposal is received too late to be committed.

--- middle

# Introduction

Post-compromise security (PCS) is one of the core security guarantees provided
by the Messaging Layer Security (MLS) protocol.  MLS provides PCS for a member
when the member's leaf node in the MLS ratchet tree is updated, either by that
member sending a Commit message, or by an Update proposal from that member being
committed.  Unfortunately, Update proposals can only be committed in the epoch
in which they are sent, for example, if an Update proposal is sent within an
epoch, but not received by the sender of the Commit that ends the epoch until
after the Commit is sent.

This situation leads to missed opportunities for PCS.  Unlike Add and Remove
proposals, which can be "re-originated" by a committer if they are received
late, Update proposals can only be sent by the member whose leaf the Update
would replace.  If an Update proposal is not included in the Commit ending its
epoch, then it can only be discarded.  The PCS benefits of the Update are lost.

In this document, we define a Replace proposal that fills this gap.  Where the
Update proposal specifies only the new leaf node, a Replace specifies the new
leaf node, as well as the leaf index where the leaf node should be inserted and
the hash of that leaf node.  This allows any member of the group to take a leaf
node value that another member has previously produced (e.g., in an Update
proposal), and propose that the latter member's representation in the group be
replaced with the new leaf node value.  The inclusion of the old leaf node's
hash ensures that the proposal can later be used to revert a new leaf node to an
older version.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC9420}}.

# Replace Proposal

A Replace proposal replaces the indicated LeafNode in the tree with the value
provided in the proposal.

~~~
struct {
    uint32 replaced;
    LeafNode leaf_node;
    opaque old_leaf_node_hash;
} Replace;
~~~
{: #fig-replace title="Content of the Replace proposal" }

A Replace proposal is invalid if the LeafNode is invalid for an Update proposal
for the indicated leaf according to {{Section 7.3 of RFC9420}}, or if the leaf
node at index `removed` is blank.  In addition, the recipient of a Replace
proposal MUST verify the following properties of the new LeafNode and reject
the proposal as invalid if any one of them is false:

* The `leaf_node_source` field of the LeafNode is be set to `update`.

* The `old_leaf_node_hash` is the hash of the LeafNode currently in leaf with
  the `replaced` index.

A member of the group applies a Replace proposal by taking the following steps:

* Replace the LeafNode at index `removed` with the one contained in the Replace
  proposal.

* Blank the intermediate nodes along the path from the leaf at index `removed`
  to the root.

This mechanism is similar to the Update proposal described in {{Section 12.1.2
of RFC9420}}, with the difference that the index of the leaf to be replaced is
made explicit, as opposed to being indicated by the `sender` field of the
enclosing FramedContent object.  A Replace proposal with the `to_replace` field
set to the same index as the `sender` field has the same effect as an Update
sending the same leaf node value.

A Commit that contains a Replace proposal MUST contain an update path.

# Proposal List Validation

This specification extends the validation rules in {{Section 12.2 of RFC9420}}
with the following additional rules governing proposal lists that include
Replace proposals.  For a regular, i.e., not external, Commit, the list is
invalid if any of the following occurs (in addition to the rules in {{Section
12.2 of RFC9420}}):

* It contains a Replace proposal replacing the committer's leaf node.

* It contains multiple Replace proposals for the same leaf node.

* It contains both a Replace proposal and an Update or Remove proposal for the
  same leaf node.

Replace proposals are not allowed in external Commits.

# Security Considerations

> **TODO:** Unlike an Update proposal, a Replace proposal is not signed by the
> current (i.e. prior to update) key of the replaced member. This allows a
> malicious insider Bob to issue a Replace proposal for Alice that changes her
> signature key to one chosen by Bob (assuming Bob can obtain a valid
> credential for that key). This is not possible with Update proposals.
> It may be good to prevent this scenario e.g. by having a leaf node extension
> `signature_under_previous_key` required for Replace proposals that change
> the signature key.

# IANA Considerations

> **TODO:** Register new proposal type Replace

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
