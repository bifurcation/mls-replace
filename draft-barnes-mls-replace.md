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
Update proposal specifies only the new leaf node, a Replace specifies both the
leaf node and the leaf index where the leaf node should be inserted.  This
allows any member of the group to take a leaf node value that another member has
previously produced (e.g., in an Update proposal), and propose that the latter
member's representation in the group be replaced with the new leaf node value.

The new flexibility that this mechanism introduces can be abused by a malicious
member to undermine PCS.  The malicious member can issue a Replace proposal that
rolls back a victim member's leaf to an earlier, compromised version.  To
prevent these attacks, we also introduce a `leaf_node_epoch` extension that
allows a LeafNode to be annotated with the epoch in which it was created.  This
allows the recipients of a Replace proposal to ensure that the new leaf node was
created after the one it replaces.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC9420}}.

# Leaf Node Epoch Extension

The `leaf_node_epoch` extension simply describes the epoch in which a LeafNode
was created:

~~~
uint64 leaf_node_epoch;
~~~
{: #fig-leaf-node-epoch title="Content of the `leaf_node_epoch` extension" }

# Replace Proposal

A Replace proposal replaces the indicated LeafNode in the tree with the value
provided in the proposal.

~~~
struct {
    uint32 replaced;
    LeafNode leaf_node;
} Replace;
~~~
{: #fig-replace title="Content of the Replace proposal" }

A Replace proposal is invalid if the LeafNode is invalid for an Update proposal
for the indicated leaf according to {{Section 7.3 of RFC9420}}, or if the leaf
node at index `removed` is blank.  In addition, the recipient of a Replace
proposal MUST verify the following properties of the new LeafNode and reject
the proposal as invalid if any one of them is false:

* The `leaf_node_source` field of the LeafNode is be set to `update`.

* The `extensions` field of the LeafNode contains an extension of type
  `leaf_node_epoch`.

* The epoch value in the `leaf_node_epoch` field is less than or equal to the
  current epoch.

* If the current LeafNode contains a `leaf_node_epoch` extension, then the
  epoch value in the `leaf_node_epoch` field is greater than or equal to the
  value in the current LeafNode.

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

As noted above, the Replace proposal introduces a risk that a malicioius group
member could roll back a victim group member to an earlier, compromised
LeafNode.  Thus while the Replace and `leaf_node_epoch` mechanisms are
notionally separate, using Replace proposals requires the use of the
`leaf_node_epoch` extension in order to avoid roll-back attacks.  In a group
where Replace proposals might be used, the application SHOULD ensure that all
leaf nodes contain a `leaf_node_epoch` extension.

> **TODO:** It might be worth defining a flag that says `leaf_node_epoch` is
> required.  That would be a good use of an MLS flags extension, along the lines
> of [I-D.ietf-tls-tlsflags] (real reference not working right now), which would
> be a nice thing to include in the MLS extensions document.

# IANA Considerations

> **TODO:** Register new proposal type Replace

> **TODO:** Register new extension type leaf_node_epoch

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
