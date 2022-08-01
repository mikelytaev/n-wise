# Aries RFC 0000: Group DID Exchange Protocol 1.0

- Authors: [Mikhail Lytaev](mikelytaev@gmail.com)
- Status: [PROPOSED](/README.md#proposed)
- Since: 2022-07-30
- Status Note: Under research
- Supersedes:
- Start Date: 2022-06-03
- Tags: [feature](/tags.md#feature), [protocol](/tags.md#protocol)
- URI: https://didcomm.org/your_protocol_name/%VER

## Summary

This RFC defines a protocol for creating and managing relationships within a group of [agents](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0004-agents) (n-wise). In a certain sense, this RFC is a generalization of the pairwise concept and protocols [0160-connection-protocol](https://github.com/hyperledger/aries-rfcs/tree/main/features/0160-connection-protocol) and [0023-did-exchange](https://github.com/hyperledger/aries-rfcs/tree/main/features/0023-did-exchange) for an arbitrary number of agents.

## Motivation

SSI subjects and agents representing them must have a way to establish relationships with each other in a trustful manner. In the simplest case, when only two agents are involved, this goal is achieved using [0023-did-exchange] protocol (https://github.com/hyperledger/aries-rfcs/blob/main/features/0023-did-exchange/README.md) by creating and securely sharing their DID Documents directly between agents. However, it is often necessary to organize an interaction involving more than two agents. The number of participants of such an interaction may change over time, and most of the agents may be mobile ones. The simplest and most frequently used example of such interaction is group chats in instant messengers. The trusted nature of SSI technology makes it possible to use group relationships to hold legally significant unions, such as board of directors, territorial community or dissertation councils.

## Tutorial

### Name and Version

n-wise, version 1.0

URI: https://didcomm.org/n-wise/1.0

### Registry of n-wise states

The current state of n-wise is an up-to-date list of participant's DID Documents. In the case of pairwise, the state is stored by the participants and updated, if necessary, by direct notification of the other party. When there are more than two participants, the problem of synchronizing the state of this n-wise( i.e. [consensus](https://en.wikipedia.org/wiki/Consensus_(computer_science))) arising. It should be borne in mind that the state may change occasionally: users may be added or deleted, DID Documents may be modified (when keys are rotated or endpoints are changed).

In principle, any trusted repository can act as a registry of n-wise states. The following options for storing the status of participants can be distinguished:

- #### Directly on the agent's side
This approach is closest to [0023-did-exchange] protocol (https://github.com/hyperledger/aries-rfcs/blob/main/features/0023-did-exchange/README.md). However since there are more than two participants, an additional consensus procedure is required to correctly account for changes in the n-wise state. This option is suitable if the participants are represented by cloud agents which are (almost) always online. In this case, a consensus can be established between them by well-known algorithms (RAFT, Paxos, BFT). However, if most of the agents are mobile and are online only occasionally, the mentioned consensus algorithms stop working. So it is preferable to use external solutions for storing and updating states.

- #### Public or private distributed ledger

In this case, the task of recording and storing the state is taken over by a third-party distributed network. The network can verify incoming transactions by executing a smart contract, or accept all incoming transactions, so transaction validation takes place only on the participating agents.

- #### Centralized storage

This case is applicable when security requirements allow participants to trust a centralized solution.

The concept of pluggable consensus implies choosing the most appropriate way to maintain a registry of states, depending on the needs.

N-wise state updates are performed by pushing the corresponding transaction to the registry of n-wise states. To get the current n-wise state, the agent receives a list of transactions from the registry of states, verifies them and applies sequentially, starting with `genesisTx`. Incorrect transactions (without a proper signature or not containing the necessary fields) are ignored. This, n-wise can be considered as replicated state machine, that is executed on each participant's agent.

The specifics of recording and receiving transactions depend on the particular method of maintaining the n-wise registry and on the specific ledger. This RFC DOES NOT DEFINE specific state registry implementations.

### Roles

* ##### User

The SSI subject taking part in n-wise. Has the right to:

* Modify its DID Document;
* Remove himself from n-wise;
* Invite new participants.

* ##### Owner
Creator of the n-wise and the committer of `genesisTx`. In addition to the rights of the user, has the right to:
* remove other users from n-wise;
* Modify n-wise metainfo;
* transfer its role to another user.

There can only be one owner in n-wise at a time.

* ##### Inviter

The participant who initiates the invitation of a new agent.

* ##### Invitee
Агент, принимающий приглашение и подключающийся к n-wise. В случае успеха он становится участником n-wise.

> See [this note](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0003-protocols/roles-participants-etc.md) for definitions of the terms
"role", "participant", and "party".

Provides a formal name to each role in the protocol, says who and how many can
play each role, and describes constraints associated with those roles (e.g.,
"You can only issue a credential if you have a DID on the public ledger"). The
issue of qualification for roles can also be explored (e.g., "The holder of the
credential must be known to the issuer").

The formal names for each role are important because they are used when [agents
discover one another's
capabilities](https://github.com/hyperledger/aries-rfcs/tree/main/features/0031-discover-features);
an agent doesn't just claim that it supports a protocol; it makes a claim about
which *roles* in the protocol it supports. An agent that supports credential
issuance and an agent that supports credential holding may have very different
features, but they both use the _credential-issuance_ protocol. By convention,
role names use lower-kebab-case and are compared case-sensitively.

### States

This section lists the possible states that exist for each role. It also
enumerates the events (often but not always messages) that can occur, including
errors, and what should happen to state as a result. A formal representation of
this information is provided in a _state machine matrix_. It lists events as
columns, and states as rows; a cell answers the question, "If I am in state X
(=row), and event Y (=column) occurs, what happens to my state?" The [Tic Tac
Toe example](https://github.com/hyperledger/aries-rfcs/blob/main/concepts/0003-protocols/tictactoe/README.md#states) is typical.

[Choreography Diagrams](
https://www.visual-paradigm.com/guide/bpmn/bpmn-orchestration-vs-choreography-vs-collaboration/#bpmn-choreography)
from [BPMN](http://www.bpmn.org/) are good artifacts here, as are [PUML sequence diagrams](
http://plantuml.com/sequence-diagram) and [UML-style state machine diagrams](http://agilemodeling.com/artifacts/stateMachineDiagram.htm).
The matrix form is nice because it forces an exhaustive analysis of every
possible event. The diagram styles are often simpler to create and consume,
and the PUML and BPMN forms have the virtue that they can support line-by-line
diffs when checked in with source code. However, they don't offer an
easy way to see if all possible flows have been considered; what they may
NOT describe isn't obvious. This--and the freedom from fancy tools--is why
the matrix form is used in many early RFCs. We leave it up to
the community to settle on whether it wants to strongly recommend specific
diagram types.

The formal names for each state are important, as they are used in [`ack`s](https://github.com/hyperledger/aries-rfcs/tree/main/features/0015-acks)
and [`problem-report`s](https://github.com/hyperledger/aries-rfcs/tree/main/features/0035-report-problem)).
For example, a `problem-report` message declares which state the sender
arrived at because of the problem. This helps other participants
to react to errors with confidence. Formal state names are also used in the
agent test suite, in log messages, and so forth.

By convention, state names use lower-kebab-case. They are compared
case-sensitively.

State management in protocols is a deep topic. For more information, please
see [State Details and State Machines](https://github.com/hyperledger/aries-rfcs/blob/main/concepts/0003-protocols/state-details.md).

### Messages

This section describes each message in the protocol. It should also note the names and
versions of messages from other message families that are adopted by the
protocol (e.g., an [`ack`](https://github.com/hyperledger/aries-rfcs/tree/main/features/0015-acks)
or a [`problem-report`](https://github.com/hyperledger/aries-rfcs/tree/main/features/0035-report-problem)).
Typically this section is written as a narrative, showing each message
type in the context of an end-to-end sample interaction. All possible
fields may not appear; an exhaustive catalog is saved for the "Reference"
section.

Sample messages that are presented in the narrative should also be checked
in next to the markdown of the RFC, in [DIDComm Plaintext format](
https://github.com/hyperledger/aries-rfcs/tree/main/features/0044-didcomm-file-and-mime-types#didcomm-messages-dm).

The _message_ element of a message type URI are typically lower_camel_case or lower-kebab-case, matching
the style of the protocol. JSON items in messages are lower_camel_case and inconsistency in the
application of a style within a message is frowned upon by the community.

#### Adopted Messages

Many protocols should use general-purpose messages such as [`ack`](
https://github.com/hyperledger/indy-hipe/pull/77) and [`problem-report`](
https://github.com/hyperledger/indy-hipe/pull/65)) at certain points in
an interaction. This reuse is strongly encouraged because it helps us avoid
defining redundant message types--and the code to handle them--over and
over again (see [DRY principle](https://en.wikipedia.org/wiki/Don't_repeat_yourself)).

However, using messages with generic values of `@type` (e.g., `"@type":
"https://didcomm.org/notification/1.0/ack"`)
introduces a challenge for agents as they route messages to their internal
routines for handling. We expect internal handlers to be organized around
protocols, since a protocol is a discrete unit of business value as well
as a unit of testing in our agent test suite. Early work on agents has
gravitated towards pluggable, routable protocols as a unit of code
encapsulation and dependency as well. Thus the natural routing question
inside an agent, when it sees a message, is "Which protocol handler should
I route this message to, based on its @type?" A generic `ack` can't be
routed this way.

Therefore, we allow a protocol to __adopt__ messages into its namespace.
This works very much like python's `from module import symbol` syntax.
It changes the `@type` attribute of the adopted message. Suppose a `rendezvous`
protocol is identified by the URI `https://didcomm.org/rendezvous/2.0`,
and its definition announces that it has adopted generic 1.x `ack`
messages. When such `ack` messages are sent, the `@type` should now use
the alias defined inside the namespace of the `rendezvous` protocol:

[![diff on @type caused by adoption](concepts/0003-protocols/adoption.png)](https://docs.google.com/presentation/d/15UAkh_2WfDk7wlto7pSL7YU9NJr_XVMgGAOeNIRbzK8/edit#slide=id.g9e66a1f72d_0_0)

Adoption should be declared in an "Adopted" subsection of "Messages".
When adoption is specified, it should include a __minimum
adopted version__ of the adopted message type: "This protocol adopts
`ack` with version >= 1.4". All versions of the adopted message that share
the same major number should be compatible, given the [semver rules](concepts/0003-protocols/semver.md)
that apply to protocols.

### Constraints

Many protocols have constraints that help parties build trust.
For example, in buying a house, the protocol includes such things as
commission paid to realtors to guarantee their incentives, title insurance,
earnest money, and a phase of the process where a home inspection takes
place. If you are documenting a protocol that has attributes like
these, explain them here. If not, the section can be omitted.

## Reference

All of the sections of reference are optional. If none are needed, the
"Reference" section can be deleted.

### Messages Details

Unless the "Messages" section under "Tutorial" covered everything that
needs to be known about all message fields, this is where the data type,
validation rules, and semantics of each field in each message type are
details. Enumerating possible values, or providing ABNF or regexes is
encouraged. Following conventions such as [those for date-
and time-related fields](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0074-didcomm-best-practices#date-time-conventions)
can save a lot of time here.

Each message type should be associated with one or more roles in the 
protocol. That is, it should be clear which roles can send and receive
which message types.

If the "Tutorial" section covers everything about the messages, this
section should be deleted.

### Examples

This section is optional. It can be used to show alternate flows through
the protocol.

### Collateral

This section is optional. It could be used to reference files, code,
relevant standards, oracles, test suites, or other artifacts that would
be useful to an implementer. In general, collateral should be checked in
with the RFC.

### Localization

If communication in the protocol involves humans, then localization of
message content may be relevant. Default settings for localization of
all messages in the protocol can be specified in an `l10n.json` file
described here and checked in with the RFC. See ["Decorators at Message
Type Scope"](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0011-decorators#decorator-scope)
in the [Localization RFC](https://github.com/hyperledger/aries-rfcs/tree/main/features/0043-l10n).

### Codes Catalog

If the protocol has a formally defined catalog of codes (e.g., for errors
or for statuses), define them in this section. See ["Message Codes and
Catalogs"](https://github.com/hyperledger/aries-rfcs/blob/main/features/0043-l10n/README.md#message-codes-and-catalogs)
in the [Localization RFC](https://github.com/hyperledger/aries-rfcs/blob/main/features/0043-l10n).

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
choosing them?
- What is the impact of not doing this?

## Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other SSI ecosystems and what experience have
their community had?
- For other teams: What lessons can we learn from other attempts?
- Papers: Are there any published papers or great posts that discuss this?
If you have some relevant papers to refer to, this can serve as a more detailed
theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other implementers, provide readers of your proposal with a
fuller picture. If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if they are an adaptation
from other communities.

Note that while precedent set by other communities is some motivation, it
does not on its own motivate an enhancement proposal here. Please also take
into consideration that Aries sometimes intentionally diverges from common
identity features.

## Unresolved questions

- What parts of the design do you expect to resolve through the
enhancement proposal process before this gets merged?
- What parts of the design do you expect to resolve through the
implementation of this feature before stabilization?
- What related issues do you consider out of scope for this 
proposal that could be addressed in the future independently of the
solution that comes out of this doc?

## Implementations

> NOTE: This section should remain in the RFC as is on first release. Remove this note and leave the rest of the text as is. Template text in all other sections should be removed before submitting your Pull Request.

The following lists the implementations (if any) of this RFC. Please do a pull request to add your implementation. If the implementation is open source, include a link to the repo or to the implementation within the repo. Please be consistent in the "Name" field so that a mechanical processing of the RFCs can generate a list of all RFCs supported by an Aries implementation.

*Implementation Notes* [may need to include a link to test results](/README.md#accepted).

Name / Link | Implementation Notes
--- | ---
 |
