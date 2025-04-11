NIP-AB
======

The server responds with 401, and a message that indicates that a login is required.

When the client wants to access a restricted resource and does not have the correct header, the server **must** returns
a
401, with an Access Request Challenge Message attached.

The message **should** contain a hint of under what conditions access would be granted, otherwise the client **must**
assume that a regular login would be accepted.

If a membership is required, a list of accepted memberships **should** be presented to the client.

Access Request Challenge Message:

    {
        kind: 5670
        tags: [
            e: (optional, multiple) <Membership Definition Message Id>
        ]
    }

The client presents the user an option to select a required membership, then responds with a correct Access Request
Message base64 encoded into the Authorization: Bearer header

Access Request Message:

    {
        kind: 5671
        tags: [
            e: <the id of the Access Request Challenge Message>
        ]
        content: (optional) = [ <list of messages that proves the membership> ]
    }

If the client does not have a required membership, the client **may** present the user with the option to buy one.

Membership Definition Message:

    {
        kind: 5672
        tags: [
            t: <Title of the membership>
        ]
        content: (optional) = <Description of the membership>
    } 

Membership Request Message:

    {
        kind: 5673
        tags: [
            e: <tag of the Membership Definition Message>
        ]
    }   

Membership Issuance Message:

    {
        kind: 5674
        tags: [
            e: (multiple) <tag of the Membership Definition Message>
            e: (optional, multiple) <tag of the Membership Request Message>
            p: (multiple) <tag of the new member>
        ]
        content: { 
            start: (optional) <UTC timestamp of the start of the membership>
            duration: (optional) <UTC timesteps that the membership lasts>
            end: (optional) <UTC timestamp of the end of the membership>
            membersince: (optional) <UTC timestamp of the first issuance of the membership as assured by the issuer>
            proofofpurchase: (optional) <A Lightning invoice to be payed>
        }
    }

#### Proof of Purchase

A Membership Issuance Message **can** contain a proofofpurchese variable, in this case the membership first needs to be
activated by issuing a Membership Activation Message.

Membership Activation Message:

    {
        kind: 5675
        tags: [
            e: <tag of the Membership Issuance Message>
        ]
        content: <The proof that the purchase has happened>
    }

### Example

In Wonderland, Bob decides to form an exclusive club for tea fans, where he would like to share his secret videos about
making tea. He wants to have two tiers in it, "Bobs TeaClub Silver", and "Bobs TeaClub Gold". Members in the gold club *
*must** also have access to the silver club. To market his Club he issues two Kind 0 messages representing the different
tiers in the club. Access to the gold club is invitation only.

    {
        kind: 5672
        tags: [
            t: "Bobs TeaClub Silver"
        ]
        content: = "Learn how to make great tea"
    } 

    {
        kind: 0
        tags: [
            e: <tag of the Membership Definition Message for silver>
            e: <tag of the Membership Definition Message for gold>
            A: 'restricted'
        ]
        content: {
            name: "Bobs TeaClub Silver"
        }
    }

Alice wants to join Bobs club, so she issues a, this can either be published open in Wonderland, or sent via a NIP44.

    }
        {
        kind: 5673
        tags: [
            e: <tag of the Membership Definition Message for silver>
        ]
    }

Bob answers with a:

    {
        kind: 5674
        tags: [
            e: <tag of the Membership Definition Message for silver>
            e: <tag of the Membership Request Message Alice issues>
            p: <Alice pubkey>
        ]
        content: { 
            duration: <30-days>
            proofofpurchase: <lnd link>
        }
    }

Alice pays and answers with a:

    {
        kind: 5675
        tags: [
            e: <tag of the Membership Issuance Message>
        ]
        content: <The recite>
    }

Alice can now log in to the relay. The relay issues a 

    {
        kind: 5670
        tags: [
            e: (optional, multiple) <Membership Definition Message Id>
        ]
    }

Alice then answers with a: 

    {
        kind: 5671
        tags: [
            e: <the id of the Access Request Challenge Message>
        ]
        content: (optional) = [ <list of messages that proves the membership> ]
    }
