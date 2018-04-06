# NEM Voting Standard

## Table of contents

1. [Introduction](#introduction)
2. [General Structure](#structure)
3. [Poll Creation](#creation)
4. [Voting](#voting)
5. [Counting Votes](#counting)
6. [Scalability](#scalability)
7. [Old Poll Structure](#old)
8. [Future](#future)
9. [Reference Implementation](#code)

## Introduction <a name="introduction"></a>

The NEM voting standard allows anybody to create a poll on the NEM blockchain for other people to vote on. Poll creation and voting is implemented on NanoWallet, and results can be seen both in NanoWallet and some NEM block explorers.

The system leverages the already existing importance score from every account to weigh votes.

There is also the possibility of creating a poll with a whitelist where all casted votes weigh equally, and in the future if the NIS allows for it, a mosaic vote is going to be implemented where votes count proportional to the amount of a certain mosaic held by the voter.

In this document we describe the the technical details of the standard that everyone should use for creating, voting and counting polls, so that everybody gets the same results.

## General structure <a name="structure"></a>

The voting structure consists of a series of NEM accounts and messages sent to them as a way of storing immutable information on the blockchain.

A poll is formed by (n + 1) accounts, where n is the number of options for the poll. These accounts are:
- The Poll Account (PA): The poll account is the main account for a poll. It contains all the info for a poll in various messages (described below)m such as the title, the description, the option addresses, etc.
- The Option Accounts(OA): Each poll has an option account for every possible option to vote on. Option accounts are empty and they contain no messages. The poll account contains all the addresses to the corresponding option accounts.

To vote on a poll a voter must send a transaction with 0 xem to the option account they wish to vote for, this must be done before the end of the poll, and the weight of their vote will be equal to the importance score from their account at the last harvested block before the poll ends.

With the two types of account described we can create a well formed poll and vote on it. If we input the poll address on the NanoWallet voting module manually we can count the votes and see all the information. But the poll will not appear in the initial menu in NanoWallet, nor in the Block explorers, since it is not on the public poll index. This is ok if you want to host a private poll, but as we will see there is a better way of doing it with private poll indexes.

Poll Indexes contain a list of Polls in the form of messages, that we will call poll headers. Poll headers contain basic info for a poll, like the title, the type of vote counting, and most importantly the poll accoiunt's address. This is sent to a poll index account at the time of the poll creation so that the poll is easily found and tracked. A public poll index account exists for general purpose public polls, but poll indexes can be created by users, both public and private (which creates a poll index where only the creator can submit polls).

The final poll structure would look something like this:

![poll structure](structure.png)

Where boxes represent accounts, black arrows represent address pointers and colored arrows represent vote messages.

## Poll Creation <a name="creation"></a>

We are going to generate some accounts to create a poll. This accounts should not be owned by anybody, they are just used to store information, so when we crete a new account, we don't need to save the private key for it, it will just be discarded.

The information sent to the accounts is encoded as plain strings or JSON depending on the type of data. The messages are sent as unquoted strings.

Let's walk through the steps to create a poll:

### 0. Poll Index (optional)
First of all if you intend to publish your poll to a personal index, you should create it first.
Generate a new account and save the address.
The address is sent as a message to the poll index creator by himself, so that there will be a way to list all indexes created by an account. This mesage has the structure:

`createdPollIndex:TATWKUGFW5RABZZGHP3AXMISRHTTCZI643VFMA62`

where TAT...A62 is the poll index address.

Then we send a message to the poll index account to declare it as a poll index and to configure it as public or private. With structure:

`pollIndex:{"private":false}`

where the private field can be "true" or "false". If the index is private only those polls sent by the creator will be valid on the index, we have to send the creator address like this:

`pollIndex:{"private":true,"address":"TATWKUGFW5RABZZGHP3AXMISRHTTCZI643VFMA62"}`

where the parameter address is the address of the index creator.

### 1. Generate Accounts
We need to generate one Poll Account and one account for every option, in no particular order. We don't need the private key because the accounts don't need to be owned by anyone, so we will save some time by generating the public key directly. A public key consists of 32 bytes that can be transformed into an address. We are interested in generating all the accounts from the poll information in a deterministic way, so that we can verify if a certain option account pertains to a certain poll (This will be useful for vote redirection cheating attempts, as discussed later).

The Account generation process will go like this:

Poll Account:

For the poll account we need two strings: the creator public key as an hex string and the poll title. Then we get the public key by doing SHA3-256(publicKey | pollTitle), where | represents string concatenation. This gives us a public key that we can convert easily to an address, as described in the original NEM technical specification.

Option Accounts:

The option accounts will be deterministically derived from the Poll Address in a similar way. Each option will have as public key SHA3-256(pollAddress | optionString) where opionString is the string describing each option. From there we get the addresses.

### 2. Send Information to Accounts
Now all the accounts we need have been created, we need to populate them with information, in the form of messages. We will first send the poll information to the Poll Account.

The messages in a NEM transaction have a length limit, which limits the details you can add to a single message, because of this we will split the information in different messages to optimize the quantity of information sent, while minimizing the fee cost of a poll creation.

Three messages are sent to the Poll Account for a regular poll:

1. **formData:**

This message contains the data that defines the poll parameters. An example message is:

`formData:{"title":"title","doe":1607772120000,"multiple":false,"type":0}`

where:
- title: The title of the poll.
- doe: Time of the poll ending when the votes will be definitive (in milliseconds since thursday, 1 January 1970)
- multiple: true or false. If set to true people is allowed to split their importance between different options.
- type: the counting type of polls:
    - 0 -> POI
    - 1 -> whitelist
    - 2 -> mosaic (not implemented)

2. **description:**

This message contains only the description of the poll, which we want to be able to be as long as possible:

`description:this is the description`

3. **options:**

This message contains the poll's options and their respective option account addresses. It consists of an array of the option strings and a dictionary that maps the option strings to their respective option account.

The string array is redundant and could be removed to save space but it was left in there for compatibility with the old poll structure (explained below). You can add quite a lot of options before you surpass the message limit, so this is not a critical problem. The format is this:

`options:{"strings":["yes","no"],"link":{"yes":"TC2BOQO2JVBZMVSFTUILCSQBGOUAZIOCZXTHAP6S","no":"TCKMEQVM32F7BL6IHU2QF4S6JJVXIMDFXN6PXBN6"}}`

the strings parameter contains an array of the option strings. The link parameter is a map from strings to their respective option addresses.

4. **whitelist (only for whitelist polls):**

The whitelist message contains a whitelist with people who is allowed to vote. This is very limited by the message length and right now does not have very practical uses. The better option would be to distribute a mosaic for everyone you want to be allowed to vote and then create a mosaic poll. But as of the current NIS api there is no historical information on mosaics, so the mosaic vote counting can't be properly implemented in a decentralized way.

`whitelist:["TCCXQPJNPXAZFKV2IZHIFLAGTSN42WPNAQI6XGK3"]`

the message contains an array with the whitelisted addresses.

5. **Poll Header:**

Once the poll has been created and there have been no errors we can add it to a poll index for discoverability. For this purpose we create a poll header with basic data for the poll and send it to a poll index account.

The message looks like this:

`poll:{"title":"title","type":0,"doe":1607772120000,"address":"TBR6KPJ2PMUXVWIDLYAUAY52XBU7KDOVTWYLBTUN"}`

all of the data here is redundant and is already stored in the poll account, but it is replicated here for easy loading of basic information without having to query the poll account.

Now the poll is formed and ready for voting after all the transactions have been confirmed.

The default public poll indexes right now are these:

Testnet -> TAVGTNCVGALLUPZC4JTLKR2WX25RQM2QOK5BHBKC
Mainnet -> NAZN26HYB7C5HVYVJ4SL3KBTDT773NZBAOMGRFZB

## Voting <a name="voting"></a>

A vote from a simple account consists of a transaction with 0xem and 0 mosaics to the desired option account. It is important that there is no xem or mosaics included, or it will not be counted. A message can be added, but it is not added when voting from NanoWallet.

Voting from a multisig account is also a transaction with 0xem and 0 mosaics, but it includes a message by default, which tells cosigners what poll and option this transaction is for. The message in NanoWallet looks like this:

`vote for poll TBR6KPJ2PMUXVWIDLYAUAY52XBU7KDOVTWYLBTUN with option "yes"`

but the format is not important, since the message is just informative for cosigners, and is ignored by the system. Nothing stops a multisig cosigner from creating a manual vote with a misleading message, but when you have a multisig account with somebody a certain amount of trust is expected.

## Vote counting <a name="counting"></a>

The result of a poll is not stored anywhere, since that would mean you have to trust the server storing it to have calculated the results correctly. To guarantee decentralization the results can be calculated by anybody anytime. The client asks the NIS for the poll information and for all of the transactions sent to the option accounts. If everybody uses the same protocol for counting votes then they get the same results.

We will describe now the specification for the correct protocol implemented in NanoWallet, that should be used by everybody in order to get the correct results. Example code will also be provided in this repo.

1. **Get the poll information**

First query the api for the first messages on the poll account that start with "options:" and "formData:". It is important that the first ones are used, since anybody can send a new message to the poll account. From the options message we get all the option account's addresses. From the formData message we get important information like the poll ending, whether it is multiple option, and the type of poll. From the options message we get all the option addresses.

2. **Get all the votes and apply filter**

Now that we have all the option addresses we ask the API for all the transactions sent to them and store them separately, so that we have all the transactions sent to each option account, we are only interested on transactions of type 257 (normal transactions) and 4100 (multisig transactions).

If the date of ending of the poll is in the future we are performing a provisional vote counting. The results can be unreliable since the importances of accounts change overtime, so the final result can vary a lot from the provisional results. In the other hand if the date of ending is in the past then we will be performing a definitive count. The importances that will be taken are those that accounts had at the last harvested block before the poll end.

In the case of a finished poll we must first find the last harvested block before the doe, this can be done with the API indirectly by approximation and then searching for the exact block. We will call this block LB (for last block). Later in this document we will describe a good algorithm to find a block by its timestamp efficiently.

Once we have found LB we filter out all those transactions that were included in a block > LB. It is important that we do this filter by the included block and not by the transaction timestamp, since it can be modified manually, and we don't want anything that happens after LB to affect the results of the poll.

Then we apply a second filter, where we ignore all the transactions that send xem or mosaics, and we remain only with 0xem / 0 mosaics transactions. This is to avoid people from voting from an exchange, as explained in detail below. This filter is applied to both ended and not ended polls.

Now we don't need the transaction info anymore, we just need the sender addresses, so we transform each transction to it's sender address for easier manipulation.

For whitelist polls an additional filter is applied that accepts only votes sent by accounts in the whitelist.

3. **Handle duplicated votes**

In the standard protocol we allow for a user to vote more than once on the same option, but the vote will only count once. When there are votes from the same account to different options, then it depends on the type of poll. For single option polls all the votes from the user are nullified for that poll. In the case of multiple option polls the importance of the voter is divided between the number of options the account voted on. So an account that has 4% importance and votes on two options will add 2% to both of the options, and so on.

To do this first we take each array of addresses, representing the votes for each option and we sort them so that if there are repeated addresses they will be adjacent. Then we iterate for the whole array and delete repetitions, leaving just one of each account.

To nullify multiple votes on single option polls we merge all the option arrays into a single array, such that the resulting array contains all voter addresses for all the accounts and they remain sorted. Since the addresses are unique for each option after the previous filter, we can conclude that if there is a repetition in the total array then that address voted in two different options, and all of its votes are nullified.

4. **Final results**

For a whitelist poll the results are just the number of addresses on each option, since after the filters only valid votes are left. Fot importance polls it is a little more complicated:

First we get a list of all the accounts that have valid votes, and then we ask the api for all of their importances at block LB, or in current block if the poll has not ended. Requesting all the importances at the same time for a big enough poll will trigger the spam protection in the NIS, check the scalability section for a better solution. Also, not all nodes support historical data. On the mainnet http://hugealice.nem.ninja is the only one that does as far as I know, so it should be used for historical data requests.

If it is a multiple answer poll, we divide the importance score of each account by the number of votes from that account.

Then finally we sum the importances of all the addresses on every option, and that gives us the result. The result is given in fractions of 1, so we multiply it by 100 to get the % score.

## Attacks <a name="attacks"></a>

In this section we will describe a list of known possible attacks and the solution the system has for it, where possible.

### Double voting

Double voting is the act of voting twice with the same account, or transfering importance to another account and voting again.

Voting on different options with the same account won't work since the vote counting protocol will invalidate such attempts.

Transfering importance to another account to vote more than once won't work either, since the final result of a poll is calculated from the importances in a single block at the poll ending, not at the time of the vote.

### Voting from an exchange

Cryptocurrency exchanges have some of the accounts with the most importance on the network. If an exchange user makes a withdrawal from an exchange and introduces an option account as their address, the huge exchange wallet will send a transaction to an option account.

Initially the voting module in NanoWallet had a list of exchange addresses that were blacklisted from voting on any poll, nut the list is hard to mantain and can be unreliable.

Latest versions implement a smarter system that filters out any transaction that transfers xem or mosaics. Since all exchanges have a minimum amount for withdrawal, you cannot withdraw 0 xem and thus you can not create a valid vote from the exchange's address.

### Vote redirection

Let's imagine Alice creates a poll P1, with two options: "yes" and "no". Alice is very interested that people vote on "yes", and not interested in people voting "no", so alice comes up with an ingenious way of getting people to vote on her poll for the "yes" option without knowing: She creates a second poll P2 that incites people to vote on a single option. For example the poll could be "Is NEM awesome?". Alice can be pretty confident that lots of people are going to vote on "yes" for P2, so when she submits this poll instead of generating the option addresses correctly she changes the "yes" address to be the same as the "yes" address on P1. So now when Bob votes on P2 for the option "yes" he thinks that he is voting for only "P2" but unknowingly he is voting for "P1" too. Three possibilities arise when Bob votes:

1. Bob already voted on P1 with option "yes": Then the vote for P2 will not have an effect on P1 since multiple votes on the same option are only counted as a single vote.
2. Bob already voted on P1 with option "no": Then the vote for P2 will count as a "yes" vote on P1, so both votes will be invalidated and Bob's "no" vote will not count.
3. Bob has not voted on P1: Then Bob will unknowingly send a valid "yes" vote for P1.

All three of these scenarios are positive for Alice, they increase the "yes" votes, decrease the valid "no" votes or stay the same for P1. The problem is Bob does not intend to send a vote for P1 when voting for P2 so Alice is cheating.

The solution to this attack is to always validate a poll before voting on it. Validating consists of checking that all the option addresses have been correctly derived from the Poll account as described in the poll creation section. This is secure since to break validation Alice would need to find a collision on SHA-3, which would break much more important things than the NEM voting system, the safety of NEM itself relies on SHA-3, as do many other cryptocurrencies.

## Scalability <a name="scalability"></a>

### Poll index scalability

As the number of polls increases the time taken to load all the poll headers from the poll index increases. In the future if the poll index gets too big to be loaded in a reasonable time some decision will have to be made. For example changing the default poll index to a new one or limiting the maximum amount of polls loaded.

### Finding the last valid block for a poll efficiently

The poll information message that is stored in the Poll Account, as described in this document contains a timestamp in milliseconds since the unix epoch. For a finished poll we need to find the last block that was harvested before the end timestamp of the poll. All transactions contain a timestamp, but this timestamp shouldn't be relied upon. It is better to use the block at which they were confirmed. When we want to do this we find a problem: there is no easy way to know which was the last block before a given timestamp. So we will essentially have to request a bunch of blocks to the nis until we find the target. This is temporally costly, so we will try as much as we can to reduce the amount of blocks we request.

To solve this we need to take into account some information:
1. The target block is such that its timestamp is less or equal than the target timestamp, and the timestamp of the next block is above it.
2. All the blocks are sorted, so the timestamp of a block will be always higher than that of the previous block
3. The interval between blocks is not completely random, blocks take an average of 60 seconds to be confirmed.

The most crucial information of all is the fact that we know the blocks are sorted. If they were not sorted this problem would be intractable since the best we could do is to check every single one until we find it.

The first algorithm that comes to mind when searching a sorted list is a binary search. In a binary search you try the middle element, if it is lower than the target then you know the target is not on the left, if it is higher then it is not in the right. We do this until we find the exact block. The maximum amount of blocks that we will have to request to the nis in the worst case is log2(h) + 1 where h is the current height of the blockchain. As of the time of writing this the height is 1546730, so the worst case is we request for 21 blocks. This is ok and will take about 20 seconds. But we can do much better. We are ignoring an important piece of information: the average block time is 60 seconds, so instead of just asking for the middle element each time it is a far better strategy to estimate a block by calculating the block that should be expected to have the target timestamp if the blocks were always exactly 60 seconds. This will not always immediately get the right block because block times fluctuate, but it will get you much closer than blindly guessing the middle one. So we estimate a block and check which side the target must be on, and then reestimate from the current block. If the estimation is beyond a point we know is impossible, then we set it at the first possible block, so the two walls get tighter, just as in the binary search, but with better guessing. With this method we reduce the amount of requested blocks by a lot, averaging about 6 blocks requested each time. This is executed in about 7 seconds which is really good considering the huge search space we are given.

To save another request or two it is also a good idea to save the blocks we already requested, so if we need it again we won't have that additional request. This will save at most 2 seconds or so, but it is something.

### Importance score requests

For counting poll results we need to request from the NIS server the importance scores of all the voters at a certain block. There is an API call that will give you this information for one account. But the problem is that when the number of votes increases, the number of requests gets too high. The NIS has a spam protection that triggers when an ip exceeds 25 requests per second. When the NanoWallet implementation first rolled out in the mainnet, and big polls started to take place, we ran into this problem, and the NEM developers offered as a solution a new API call. It is a single POST request that gets an array of addresses and returns an array of importances, in just one request.

The POST request has the form:

http://alice.dd-dns.de:7890/account/historical/get/batch

with supplied json object:
```javascript
{
   "accounts": [
      { "account": "TALICEROONSJCPHC63F52V6FY3SDMSVAEUGHMB7C" },
      { "account": "TALIC37D2B7KRFHGXRJAQO67YWOUWWA36OU46HSG" }
   ],
   "startHeight": 100000,
   "endHeight": 100001,
   "incrementBy": 1
}
```
It can actually return the importance for different blocks, between startHeight and endHeight, in increments of incrementBy, but for voting that is not needed so startHeight and endHeight will be the same.

The given parameters would return an object like:
```javascript
{

    "data": [
        {
            "data": [
                {
                    "pageRank": 0.0035903320547790232,
                    "address": "TALICEROONSJCPHC63F52V6FY3SDMSVAEUGHMB7C",
                    "balance": 50386727995377,
                    "importance": 0.0059685675660915375,
                    "vestedBalance": 50382877508084,
                    "unvestedBalance": 3850487293,
                    "height": 100000
                },
                {
                    "pageRank": 0.0035903320547790232,
                    "address": "TALICEROONSJCPHC63F52V6FY3SDMSVAEUGHMB7C",
                    "balance": 50386727995377,
                    "importance": 0.0059685675660915375,
                    "vestedBalance": 50382877508084,
                    "unvestedBalance": 3850487293,
                    "height": 100001
                }
            ]
        },
        {
            "data": [
                {
                    "pageRank": 0.0035903320547790232,
                    "address": "TALIC37D2B7KRFHGXRJAQO67YWOUWWA36OU46HSG",
                    "balance": 50203823850265,
                    "importance": 0.005948444666726026,
                    "vestedBalance": 50200067389240,
                    "unvestedBalance": 3756461025,
                    "height": 100000
                },
                {
                    "pageRank": 0.0035903320547790232,
                    "address": "TALIC37D2B7KRFHGXRJAQO67YWOUWWA36OU46HSG",
                    "balance": 50203823850265,
                    "importance": 0.005948444666726026,
                    "vestedBalance": 50200067389240,
                    "unvestedBalance": 3756461025,
                    "height": 100001
                }
            ]
        }
    ]

}
```

## Old poll structure <a name="old"></a>

In the first version of the voting system the structure of the options message that stores the options and their respective addresses was different. It looked like this:

`options:{"strings":["yes","no"],"addresses":["TC2BOQO2JVBZMVSFTUILCSQBGOUAZIOCZXTHAP6S","TCKMEQVM32F7BL6IHU2QF4S6JJVXIMDFXN6PXBN6"]}`

Where the first string of the strings array was supposed to correspond to the first address of the addresses array, and it worked for a while, until for some reason the addresses got swapped between different versions of NanoWallet and people sent votes to an option he didn't intend to. The structure was changed to be clearer and give no margin for error, by linking the strings directly to their addresses with a map:

`options:{"strings":["yes","no"],"link":{"yes":"TC2BOQO2JVBZMVSFTUILCSQBGOUAZIOCZXTHAP6S","no":"TCKMEQVM32F7BL6IHU2QF4S6JJVXIMDFXN6PXBN6"}}`

Current versions of NanoWallet still have compatibility for old structure polls, but due to the unreliability, it is strongly advised to avoid it at all costs and create polls always with the new structure. To make sure polls are created correctly it is important to always use the latest version of NanoWallet to create polls.

## Future <a name="future"></a>

### Mosaic voting

Right now the most important feature that is lacking is the ability to create mosaic polls. Mosaic polls would be weighted by the amount of a certain mosaic that the voter owns. This would allow for very interesting possibilities, and a better way of creating whitelist polls. It would work essentially as Proof of Stake, and since xem is a mosaic itself you could use it as the weighing token.

Right now this could only be implemented with xem, since it is the only mosaic with historical data available in the NIS.

Historical data is the basis of the voting system. It is what allows the system to be trully decentralized and trust-less, since it allows everybody to verify the results themselves. This is what makes the voting system meaningful.

Mosaic polls could be created and votes could be counted for the current block, but not for a past one, so a result should be stored somewhere and people who weren't there would have to trust the results. This is not the blockchain way, so it stays unimplemented until this function becomes available in the NIS. Maybe in Catapult, who knows...

### Liquid democracy

Liquid democracy is a democratic system where people can delegate their vote to a representative, who is allowed to vote for them.

This is expected to be implemented in the next big update for the voting system.

There are some ideas about how this can be implemented, but no definitive one is decided yet. If somebody has a suggestion or an idea for an easy, efficient and secure way to implement delegation please contact me and we can discuss the idea.

## Implementation Example <a name="code"></a>

You can find in this repo a simple implementation in node.js of the vote counting process. to execute it navigate to the js folder then:

`$ npm install`

`$ node count-votes.js NBDK5MNPM7G72GYFN3QXYYKQMFXD4YTYJAGMQXUG`

where NB..UG can be any poll address from the mainnet or testnet.

The program will log to the terminal the details and results of the poll.

The reference implementation of the NEM Voting System is the nem-voting npm module hosted at https://github.com/shierve/nem-voting
