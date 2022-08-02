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

This RFC defines a protocol for creating and managing relationships within a group of SSI subjects (n-wise). In a certain sense, this RFC is a generalization of the pairwise concept and protocols [0160-connection-protocol](https://github.com/hyperledger/aries-rfcs/tree/main/features/0160-connection-protocol) and [0023-did-exchange](https://github.com/hyperledger/aries-rfcs/tree/main/features/0023-did-exchange) for an arbitrary number of parties.

## Motivation

SSI subjects and [agents](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0004-agents) representing them must have a way to establish relationships with each other in a trustful manner. In the simplest case, when only two agents are involved, this goal is achieved using [0023-did-exchange] protocol (https://github.com/hyperledger/aries-rfcs/blob/main/features/0023-did-exchange/README.md) by creating and securely sharing their DID Documents directly between agents. However, it is often necessary to organize an interaction involving more than two agents. The number of parties of such an interaction may change over time, and most of the agents may be mobile ones. The simplest and most frequently used example of such interaction is group chats in instant messengers. The trusted nature of SSI technology makes it possible to use group relationships to hold legally significant unions, such as board of directors, territorial community or dissertation councils.

## Tutorial

### Name and Version

n-wise, version 1.0

URI: https://didcomm.org/n-wise/1.0

### Registry of n-wise states

The current state of n-wise is an up-to-date list of parties DID Documents. In the case of pairwise, the state is stored by the participants and updated, if necessary, by direct notification of the other party. When there are more than two participants, the problem of synchronizing the state of this n-wise( i.e. [consensus](https://en.wikipedia.org/wiki/Consensus_(computer_science))) arising. It should be borne in mind that the state may change occasionally: users may be added or deleted, DID Documents may be modified (when keys are rotated or endpoints are changed).

In principle, any trusted repository can act as a registry of n-wise states. The following options for storing the n-wise state can be distinguished:

- #### Directly on the agent's side
This approach is closest to [0023-did-exchange] protocol (https://github.com/hyperledger/aries-rfcs/blob/main/features/0023-did-exchange/README.md). However since there are more than two participants, an additional consensus procedure is required to correctly account for changes in the n-wise state. This option is suitable if the participants are represented by cloud agents which are (almost) always online. In this case, a consensus can be established between them by well-known algorithms (RAFT, Paxos, BFT). However, if most of the agents are mobile and are online only occasionally, the mentioned consensus algorithms stop working. So it is preferable to use external solutions for storing and updating states.

- #### Public or private distributed ledger

In this case, the task of recording and storing the state is taken over by a third-party distributed network. The network can verify incoming transactions by executing a smart contract, or accept all incoming transactions, so transaction validation takes place only on the participating agents.

- #### Centralized storage

This case is applicable when security requirements allow participants to trust a centralized solution.

The concept of pluggable consensus implies choosing the most appropriate way to maintain a registry of states, depending on the needs.

N-wise state updates are performed by pushing the corresponding transaction to the registry of n-wise states. To get the current n-wise state, the agent receives a list of transactions from the registry of states, verifies them and applies sequentially, starting with `genesisTx`. Incorrect transactions (without a proper signature or not containing the necessary fields) are ignored. This, n-wise can be considered as replicated state machine, that is executed on each participant.

The specifics of recording and receiving transactions depend on the particular method of maintaining the n-wise registry and on the specific ledger. This RFC DOES NOT DEFINE specific state registry implementations.

### Roles

* ##### User

The party of n-wise. Has the right to:

* Modify its DID Document;
* Remove himself from n-wise;
* Invite new parties.

* ##### Owner
In addition to user rights, has the right to:
* remove other users from n-wise;
* Modify n-wise meta information;
* transfer its role to another user.

There can only be one owner in n-wise at a time.

* ##### Creator

Creator of the n-wise and the committer of `genesisTx`. The creator automatically becomes is the owner of n-wise after creation.

* ##### Inviter

The n-wise party who initiates the invitation of a new agent.

* ##### Invitee

The party accepting the invitation and connecting to n-wise. If successful, the party becomes a user of n-wise.

### N-wise creation

The creation begins with the initialization of the n-wise registry. This RFC DOES NOT SPECIFY the procedure for n-wise registry creation. After creating the registry, the creator pushes `genesisTx` transaction. The creator automatically obtains the owner role. The creator MUST generate a new unique DID and DID Document for n-wise.

### Invitation of the new party

Any n-wise party can create an invitation to join n-wise. First, Inviter generates a pair of public and private invitation keys according to Ed25519. The public key of the invitation is pushed to the registry using `invitationTx` transaction. Then the `Invitation` message with invitation private key is sent out-of-band to the Invitee. The invitation key pair is unique for each Invitee and can only be used once.

### Accepting the invitation

Once the `Invitation' received, the Invite generates a new unique BID and BID Document for the n-wise and pushes `AddParticipantTx` transaction to the registry.
It is NOT ALLOWED to reuse DID from other relationships.

The process of adding a new participant is shown in the figure below

![](add_participant.png)

### Updating DID Document

Updating the user's DID Document is required for the key rotation or endpoint updating. To update the associated DID Document, user pushes the `updateParticipantTx` transaction to the registry.

> See [this note](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0003-protocols/roles-participants-etc.md) for definitions of the terms
"role", "participant", and "party".

### Removing a party form n-wise

Removing is performed using the `removeParticipantTx` transaction.
The user can delete himself (the corresponding transaction is signed by the user's public key). The owner can delete any user (the corresponding transaction is signed by the owner's public key).

Updating n-wise meta information

Meta information can be updated by the owner using the `updateMetadataTx` transaction.

### Transferring the owner role to other party

The owner can transfer control of n-wise to another n-wise party. The old owner loses the corresponding privileges and becomes a regular user. The operation is performed using the `NewOwnerTx` transaction.

### Notification on n-wise state update

Just after pushing the transaction to the n-wise registry, the participant MUST send `LedgerUpdateNotify` message to all other parties.
The participant who received `LedgerUpdateNotify` SHOULD get the current status from the n-wise registry.

### DIDComm messaging within n-wise

It is allowed to exchange DIDComm messages of any type within n-wise.
The belonging of the sender to a certain n-wise is determined by the sender's verkey.

This RFC DOES NOT DEFINE a procedure of exchanging messages within n-wise. In the simplest case, this can be implemented as sending a message to each participant in turn. In case of a large number of parties, it is advisable to consider using a centralized coordinator who would be responsible for the ordering and guaranteed sending of messages from the sender to the rest of parties.

## Reference

### N-wise registry transactions

N-wise state is modified using transactions.

```json
{
  "type": "transaction type",
  ...
  "proof" {
    "type": "JcsEd25519Signature2020",
    "verificationMethod": "did:alice#key1",
    "signatureValue": "..."

  }
}

#### Attributes
* `type` - required attribute, type of transaction;
* `proof` - required attribute, transaction signature in [JSON-LD Proof](https://w3c-ccg.github.io/data-integrity-spec/) format;
* `verificationMethod` - required attribute, depends on the specific type of transaction and is defined below.

### GenesisTx

'GenesisTx' is a mandatory initial transaction that defines the basic properties of n-wise.

```json
{
  "type": "genesisTx",
  "label": "Council",
  "creatorNickName": "Alice",
  "creatorDid": "did:alice",
  "creatorDidDoc": {
   ..
  },
  "ledgerType": "iota@1.0",
  "metaInfo" {
    ...
  }
}
```

#### Attributes
* `label` - required attribute, n-wise name;
* `creatorNickname` - required attribute, creator nickname;
* `creatorDid` - required attribute, - DID of the creator;
* `creatorDidDoc` - required attribute, - DID Document of the creator;
* `ledgerType` - required attribute, n-wise registry type;
* `metaInfo` optional attribute, - additional n-wise meta information; the format is determined by a particular n-wise state implementation;

`genesisTx` transaction MUST be signed by the creator's public key defined in his DID Document.

### InvitationTx

This transaction adds invitation public keys to the n-wise registry.

```json
{
  "type": "invitationTx",
  "publicKey": [
    {
      "id": "invitationVerkeyForBob",
      "type": "Ed25519VerificationKey2018",
      "publicKeyBase58": "arekhj893yh3489qh"
    }
  ]
}

```
#### Attributes
* `publicKey` - required attribute, array of invitation public keys;
* `id` - required attribute, invitation public key ID;
* `type` - required attribute, key type;
* `publicKeyBase58` - required attribute, Base58 encoded public key.

invitationTx` MUST be signed by the user's public key defined in its DID Document.

### Invitation message

The message is intended to invite a new participant. It is sent via an arbitrary communication channel (pairwise, qr code, e-mail, etc.).
 
```json
{
  "@id": "5678876542345",
  "@type": "https://didcomm.org/n-wise/1.0/invitation",
  "label": "Invitaion to join n-wise",
  "invitationKeyId": "invitationVerkeyForBob",
  "invitationPrivateKeyBase58": "qAue25rghuFRhrue....",
  "ledgerType": "iota@1.0",
  "ledger~attach": [
    {
      "@id": "attachment id",
      "mime-type": "application/json",
      "data": {
        "base64": "<bytes for base64>"
      }
    }  
  ]
}

```
#### Attributes
* `label` - optional attribute, human readable invitation text;
* `invitationKeyId` - required attribute, invitation key ID;
* `invitationPrivateKeyBase58`- required attribute, Base58 encoded invitation private key;
* `ledgerType` - required attribute, n-wise registry type;
* `ledger~attach` - optional attribute, attachment with meta information, required for connection to n-wise registry. Defined by a particular registry implementation.

### AddParticipantTx

The transaction is designed to add a new user to n-wise.

```json
{
  "id": "addParticipantTx",
  "nickname": "Bob",
  "did": "did:bob",
  "didDoc": {
    ...
  }
  
}

```
#### Attributes
* `nickname` - required attribute, user nickname;
* `did` - required attribute, user's DID;
* `didDoc` - required attribute, user's DID Document.

`AddParticipantTx` transaction MUST be signed by invitation private key  `invitationPrivateKeyBase58`, received with `Invitation` message. As pushing the `AddParticipantTx` transaction, the corresponding invitation key pair is considered deactivated (other invitations cannot be signed by it).

The transaction executor MUST verify if the invitation key was indeed previously added. Execution of the transaction entails the addition of a new party.

## UpdateParticipantTx

The transaction is intended to update information about the participant.

```json
{
  "type": "updateParticipantTx",
  "did": "did:bob",
  "nickname": "Updated Bob",
  "didDoc" {
    ...
  }
}

```
#### Attributes
* `did` - requred attribute - DID of the updating user;
* `nickname` - optional attribute, new user's nickname;
* `didDoc` - optional attribute, - new user's DID document.

Transaction MUST be signed by the public key of the user being updated The specified public key MUST be defined in the previous version of the DID Document.

Execution of the transaction entails updating information about the participant.

### RemoveParticipantTx

The transaction is designed to remove a party from n-wise.

```json
{
  "type": "removeParticipantTx",
  "did": "did:bob"
}
```
#### Attributes
* `did` - requred attribute - DID of the removing user;

The execution of the transaction entails the removal of the user and his DID Document from the list of n-wise parties.

The transaction MUST be signed by the public key of the user who is going to be removed from n-wise, or with the public key of the owner.

### UpdateMetadataTx

The transaction is intended to update the meta-information about n-wise.

```json
{
	"type": "updateMetadataTx",
	"label": "Updated Council"
	"metaInfo": {
	  ...
	}
}
```

#### Attributes
* `label` - optional attribute, new n-wise name;
* `metaInfo` - - optional attribute, new n-wise meta-information.

The transaction MUST be signed by the owner's public key.

### NewOwnerTx

The transaction is intended to transfer owner rights to another user. The old owner simultaneously becomes a regular user.

```json
{
	"type": "newOwnerTx",
	"did": "did:bob"
}
```

#### Attributes
* `did` required attribute, new owner's DID.

The transaction MUST be signed by the owner's public key.

### LedgerUpdateNotify

The message is intended to notify participants about the  modifications of the n-wise state.

```json
{
  "@id": "4287428424",
  "@type": "https://didcomm.org/n-wise/1.0/ledger-update-notify"
}
```

## Drawbacks

External DLT is required.



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
