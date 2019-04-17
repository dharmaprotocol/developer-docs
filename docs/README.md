# How Does Dharma Work?

Under the hood, Dharma is powered by a suite of non-proprietary smart contracts that make it possible to borrow and lend crypto-assets on blockchains like Ethereum.  

We refer to this system of smart contracts as the **Dharma Settlement Contracts**.

The **Dharma Settlement Contracts** are open-source and permissionless to build on — though we'd like to think Dharma is the easiest way to leverage them, other clients for interacting with them have been built by third-party developers in the past, and we encourage developers to build more clients in the future.

> The core Dharma Settlement Contracts can be found on our [Github](https://github.com/dharmaprotocol/charta)

## In A Nutshell

The **Dharma Settlement Contracts** are...

**A smart contract framework for tokenized debt agreements**

*   Administer the **entire life-cycle of a loan** through smart contracts
*   Collateral is **held in escrow in smart contracts** and released to creditors upon a borrower's default
*   Creditor's stake in a loan is "tokenized" -- it can be **traded, repackaged, and programmed** like any other token

**An open, permissionless credit market**

*   A marketplace of **Relayers** who earn fees for hosting "order books" that connect borrowers and lenders
*   A marketplace of **Underwriters** who earn fees for pricing borrower default risk
*   A standardized message schema for connoting intent to borrow / lend, referred to as a **Debt Order**, enables increased liquidity through programmatic lending

**A generic and modular system**

*   Virtually any type of debt agreement can be defined with a **Terms Contract**, be it a consumer margin loan or a corporate bond
*   Developers can extend Dharma Protocol by programming new **Terms Contracts** for radically different types of debt agreements
*   Terms Contracts have a standard interface that makes it easy for developers to build **credit derivatives, structured financial products, insurance contracts, and more**

# Dharma Settlement Contracts
Let's do a deep dive into the core fundamentals of the smart contracts that undergird Dharma.
In order to do that, we need to introduce some core concepts.

## Terms Contracts

**Terms Contracts** are the core primitive used to define debt agreements in Dharma protocol.

Consider the following scenario:

> Alice wants to lend $200 to Bob, at an interest rate of 12.5%, which is to be paid back each month for 12 months. Bob agrees to these terms.

Alice and Bob need a way to record the details of this agreement so that any future disputes about loan terms can be fairly adjudicated. In the traditional financial system, Alice and Bob would sign a legal contract outlining the terms of the loan. With Dharma, we can replicate this on an immutable, permanent blockchain using a a smart contract we call a **Terms Contract**. Terms Contracts are similar to legal contracts, in that they serve as proof that a debtor and creditor (Bob and Alice) have entered into a repayment agreement with one another.

This proof is valuable if there is ever a dispute about the terms of an agreement. In the traditional financial system, legal contracts serve as proof to courts of law that both borrower and lender agreed to specific terms, and justify intervention if either party fails to adhere to their side of the agreement. Similarly, Terms Contracts serve as proof that specific terms were agreed to, and can be given to a smart contract in order to take corrective action if either side deviates from their responsibilities -- more on this later.

You can think of a Terms Contract as representing a standard template for a single type of loan. Just as you could print a new template, fill in the blanks, and have counterparties sign a standard legal template, Terms Contracts can be used repeatedly -- counterparties simply agree to the values of the loan parameters (i.e. fill in the blanks of the template) and sign the agreement (i.e. cryptographically sign the debt order). In this way, a single Terms Contract can generate an infinite number of debt agreements.

<!-- [Insert diagram of Terms Contract being used as a template for several loans] -->

When the terms of the agreement (e.g. $200 at 12.5% interest over 12 months) are signed by Alice and Bob, they get recorded on the blockchain. The terms need to be stored in a compact way (in order to save space) and be structured in way that allows the Terms Contract to parse them. To this end, the terms get packed as a string of data called the **Terms Contract Parameters**. This string of data isn't meant to be readable to humans, but it must contain all of the information that the Terms Contract needs to determine the repayment status of the loan.

An example of a Terms Contract Parameters string would be `0x04000000000de0b6b3a7640000009c40200030000000001bc16d674ec8000000`. To a specific Terms Contract, these parameters happen to describe an agreement where the borrower is requesting 1 ETH at a 4% interest rate, over a 3 week repayment duration, and is putting up 4 REP as collateral for the loan. The specific Terms Contract referenced here is Dharma's _[Collateralized Simple Interest Terms Contract](https://github.com/dharmaprotocol/charta/blob/master/contracts/examples/CollateralizedSimpleInterestTermsContract.sol)_.

<!-- [Insert diagram illustrating how Terms Contract Parameters map to different fields in the Terms Contract "template"] -->

As long as the kinds of information contained in these Terms Contract Parameters are the same for two Debt Orders, the requester can use the same Terms Contract. If entirely new functionality needs to be encoded into the parameters, then a new Terms Contract that can parse those parameters must be created.

Anyone in the world can create and deploy new Terms Contracts. These new Terms Contracts can have different parameter structures, which means that radically different types of debt agreements are possible -- it just depends on how the new Terms Contracts are programmed. This extensibility is why we say that Dharma Protocol is _"generic"_.

Dharma Protocol requires that every debt agreement is associated with a Terms Contract, so that it can be the "source-of-truth" for determining the repayment status of a debt. In order to do this, Dharma Protocol expects every Terms Contract to implement the following functions:

*   **registerTermStart** - This function registers that the terms contract has begun, and is called automatically by the Debt Kernel when an order is filled. This can be used to take care of any actions that should happen at the start of the loan - such as moving collateral from a debtor to a secure Collateralizer contract.
*   **registerRepayment** - This function is called automatically by the Repayment Router contract upon any loan repayments.
*   **getExpectedRepaymentValue** - Given a loan's agreement ID and a [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time), this function will tell you how many tokens are expected to have been repaid for that loan by the given time. Note that this will use the blockchain's timestamp (which is also a Unix timestamp) as a reference, which is specific to the current block being mined -- therefore it won't be an exact time, but approximately correct within 12 minutes.
*   **getValueRepaidToDate** - Given a loan agreement ID, this function will return the total number of tokens that have been repaid at the current time.
*   **getTermEndTimestamp**- Given a loan agreement ID, this function will return a Unix timestamp representing the time at which the debt agreement ends.

## Debt Tokens

When a creditor and debtor decide to engage in a debt agreement on Dharma Protocol, they perform a cryptographic handshake, and submit a signed message called a **Debt Order** to the Dharma smart contracts (this is described further below). If the **Debt Order** is valid, the Dharma smart contracts will transfer the principal amount from the creditor to the debtor, and mint a _unique_ token to the creditor.

<!-- [Insert illustration here of Dharma debt token being minted and transferred to the creditor] -->

This token is called a **Debt Token**. Ownership of this Debt Token entitles the creditor to receive repayments from the loan. Dharma Debt Tokens are inherently tradable, so if the creditor trades the Debt Token with someone else, future repayments will be routed to the new owner (NB: this new owner need not be a person -- it can be another smart contract!).

<!-- [Insert illustration here of 1. Repayment happening, 2. Debt Token getting transferred and 3. New repayment happening and going to new person] -->

Each Debt Token has an `agreementId` -- a unique 32-byte identifier that is generated deterministically from the associated debt's terms. Each `agreementId` is mapped on-chain to the address of a deployed **Terms Contract** and a set of **Terms Contract Parameters** specific to that Debt Token.

This means that anyone who's interested in what the terms and repayment status of a Debt Token are can look up what **Terms Contract** and **Terms Contract Parameters** are associated with that Debt Token's `agreementId`. By knowing the Debt Token's terms contract and associated parameters, one can programmatically answer any and all of the following questions...

> How much of token X is the borrower expected to repay? What's the interest rate of this loan? How much of token Y is held in collateral? How much has already been repaid?

...and so on and so forth. This makes it easy for exchanges to list and display Debt Tokens -- _all_ the information they need is readily available on-chain.

<!-- [Insert diagram illustrating DebtTokens having an `agreementID`, and that `agreementId` being mapped to the Terms contract and parameters] -->

Debt Tokens are _non-fungible_, which means that no two tokens are interchangeable. In contrast, a holder of 1 ETH is basically indifferent between holding that 1 ETH or any other 1 ETH, which is why ETH is said to be "fungible". As _unique, non-fungible_ tokens, Dharma Debt Tokens comply with the ERC721 token standard.

## Repayments

Let's go back to our friends Alice and Bob. Suppose Alice and Bob engage in a Dharma debt agreement using a **Terms Contract** at address [0x5de25...](https://etherscan.io/address/0x5de2538838b4eb7fa2dbdea09d642b88546e5f20), and that the corresponding **Terms Contract Parameters** state that Bob owes 1.1 ETH every month for 3 months.

Alice now owns a Debt Token with an `agreementId` of `0x75fca...`, and Bob owes, in total, 3.3 ETH.

Let's say a month has passed, and Bob has an outstanding 1.1 ETH he must repay as soon as possible. In order to do this, Bob calls the `makeRepayment` function on one of the Dharma smart contracts (namely, the `RepaymentRouter`). The function transfers 1.1 ETH from Bob to whoever holds the Debt Token with an `agreementId` of `0x75fca...` -- at this point, Alice -- and registers the repayment with the **Terms Contract** at address [0x5de25…](https://etherscan.io/address/0x5de2538838b4eb7fa2dbdea09d642b88546e5f20). Anyone who now queries the **Terms Contract** at address [0x5de25…](https://etherscan.io/address/0x5de2538838b4eb7fa2dbdea09d642b88546e5f20) about Alice and Bob's agreement will see that Bob has repaid ⅓ of this outstanding debt, and that he has not missed his required installment payment.

<!-- [Insert diagram of repayment being transferred from Bob to Alice and repayment router reporting payment to terms contract] -->

Now, suppose another month has passed, and Alice has sold Debt Token `0x75fca...` to her friend Charlie. Bob once again is expected to repay an additional installment of 1.1 ETH, and again calls the `makeRepayment` function on a Dharma smart contract. Just as in the first repayment, the Dharma smart contracts transfer 1.1 ETH from Bob to the holder of Debt Token `0x75fca..` -- Alice's friend Charlie -- and registers the repayment with the Terms Contract.

<!-- [Insert diagram of repayment being transferred from Bob to Charlie and repayment router reporting payment to terms contract] -->

Note that Bob doesn't need to know _anything_ about who or what owns Debt Token `0x75fca…` at the time of repayment -- his repayment is automatically routed to the Debt Token's current owner by the Dharma smart contracts.

## Defaults

A _default_ is when a borrower fails to fulfill their obligation to make a loan repayment according to the terms of their loan. What happens when a borrower defaults depends on whether or not the loan is collateralized, and what the terms of the loan are.

Typically, a loan is considered to have _entered default_ when the expected amount repaid exceeds the _actual_ amount repaid at any given point in time. Programmatically, this can be determined using methods on the standardized Terms Contract interface:

```javascript
if (termsContract.getExpectedRepaymentValue(block.timestamp) > termsContract.getValueRepaid()) {
    isInDefault = true;
}
```

Having a standardized way of checking whether a loan is _in default_ is very useful -- particularly for developers who want to build mechanisms for discouraging defaults.

We'll discuss a few here, but it's important to note that these are not the _only_ ways developers can prevent defaults -- if you have a novel idea for how to discourage defaults in _your_ system, you can easily build it yourself and hook it into a Dharma Terms Contract.

### Collateralized Loans

_Collateralization_ is when a borrower locks up an asset as recourse to the lender in the event that the borrower defaults on the loan. In the traditional financial system, collateralized assets are held by human intermediaries called _escrow agents_. In Dharma Protocol, collateral is held on-chain by smart contracts that operate programmatically. Broadly speaking, if the borrower defaults, the lender gains the ability to seize the borrower's collateral.

Let's look at an example: suppose Alice lends Bob 300 DAI using Dharma, and Bob has put up 1 ETH as collateral. They are using the `CollateralizedSimpleInterestTermsContract` to define the terms of the agreement, and the terms of the loan are that Bob owes Alice 330 DAI in 1 week. Currently, Bob has Alice's 300 DAI and is free to use it however he pleases. Meanwhile, Bob's 1 ETH is being held by the `CollateralizedSimpleInterestTermsContract` as collateral.

Now, let's say Bob forgets to pay Alice the 330 DAI back in 1 week -- or, he pays only 300 DAI, and not the full, expected 330 DAI. Bob and Alice's loan has now entered default. As a result, Alice can now call the `seizeCollateral` method of the contract in which Bob's collateral is held. If she does this, she'll be sent Bob's 1 ETH to compensate for her losses.

The only way for Bob to prevent Alice from being able to claim his collateral is to repay any remaining balance on the loan. As soon as he has done so, the loan will no longer be in default and his collateral will no longer be eligible for seizure.

Alternatively, let's say the loan has been fully repaid and Bob no longer owes Alice any money. Bob can now call the `withdrawCollateral` method of the contract in which his collateral is stored, and his 1 ETH will be returned to him.

_Note: The process described above is specific to the way the CollateralizedSimpleInterestTermsContract works -- different varieties / mechanisms for collateralization can easily be built into new TermsContract if a developer desires more custom conditions._

### Uncollateralized Loans

!> _Warning: unsecured / uncollateralized loans are an experimental feature set of Dharma, and are **not** trustless. Only borrow or invest in uncollateralized loans if you really know what you're doing and are willing to risk losing your money with little to no recourse._

_Uncollateralized_ loans are loans in which a borrower did _not_ lock up any assets in order to get the loan. For the lender, this is _much_ riskier, because the borrower could refuse to make repayments with no on-chain financial ramifications. (NB: uncollateralized loans are also called _unsecured_ loans.)

There are many ways to make uncollateralized loans less risky for lenders like Alice, but we'll talk about one mechanism in particular here: **Underwriters.**

In Dharma, an Underwriter is a **trusted** third party that facilitates loans between borrowers and lenders. Underwriters are responsible for making an on-chain prediction as to the probability that a borrower will repay his loan in full. Underwriters are also responsible for _servicing the loans_ that they underwrite, which means they must do everything in their reasonable power to make sure the borrower pays back.

You might be wondering -- why should the Underwriter put _any_ effort into making sure the loan is repaid when they've already gotten their fee? The answer is simple: they have a **reputation** to maintain. The predictions they make about the borrower's probability of repayment are **permanently** recorded on the blockchain and associated with the Underwriter's public key -- that way, _anyone_ can see all of the Underwriter's historical predictions, and how well those predictions matched up to reality.

If a lender sees a Debt Order where an Underwriter has predicted there to be 2% chance of default, that lender can go on a site like [Loanscan](https://loanscan.io), look up all loans associated with that Underwriter, and see how often loans that were predicted to be "safe" were _actually_ repaid.

This incentivizes Underwriters to do a good job at:

1.  Accurately predicting the likelihood that a borrower will default
2.  Preventing a borrower from defaulting by doing things like reminding her about repayments, signing a legal contract with her that stipulates some "real-world" consequences to defaulting, etc.

To learn more about how Underwriters work in Dharma protocol, read the following section on Underwriters in these docs, or, alternatively, read [this non-technical primer on Dharma](https://blog.dharma.io/dharma-protocol-in-a-nutshell-a7abcc716429).

To learn more about our thoughts on unsecured lending, read [this blog post](https://blog.dharma.io/current-and-future-approaches-to-unsecured-lending-in-dharma-protocol-8199eea720de).

!> _Warning: Again, we must reiterate that unsecured loans are an experimental feature set of Dharma. There are several flaws in the current model for underwriting loans in Dharma that make it so that underwriters **must** be known, trusted entities. To learn more about these shortcomings (and the ways we plan on addressing them), we recommend studying [this post](https://blog.dharma.io/current-and-future-approaches-to-unsecured-lending-in-dharma-protocol-8199eea720de)._

## Debt Orders

Let's say you want to request a loan on Dharma. The first thing you need to do in order to tap into the Dharma Credit Market is construct a **Debt Order.**

A **Debt Order** is a signed message that represents a person's desire to borrow some amount of tokens at a specified set of terms and conditions. Simplistically, you can think of a **Debt Order** as a data packet that contains the following information:

1.  The **borrower's** Ethereum address
2.  The desired **[principal amount](https://www.investopedia.com/terms/p/principal.asp)**
3.  The address of the **[Terms Contract](/primers/terms-contracts)** that will define the "template" terms of this debt agreement (i.e. interest rate, duration, etc.)
4.  The **Terms Contract Parameters** that will fill-in-the-blanks for the loan's "template"
5.  The **fees** that will be paid to the **Relayer and Underwriter** (more on this later)

Think of a **Debt Order** as being a "magic" string of letters and numbers. Once the **Debt Order** is _fillable_(we'll discuss in a moment what this means),anybody who has knowledge of this "magic" string of letters and numbers can _fill_ it by submitting it to a specific Dharma Protocol smart contract (called the **Debt Kernel**). By filling a **Debt Order**, the creditor is agreeing to the terms specified in the **Debt Order**. Returning to our template analogy, the creditor is accepting not just the logic of the loan agreement, but also the specific values in the template's blank spaces (i.e. the interest rate, duration, principal amount, etc.). More importantly, a couple actions are triggered by a Debt Order being _filled:_

1.  The desired principal amount is transferred from the lender to the borrower
2.  The lender is given a [Debt Token](/primers/debt-tokens) representing the borrower's obligation to repay the lender
3.  Fees are paid out to the **Relayer and Underwriter** (if they are involved in the transaction...again, more on this later)

<!-- [Insert diagram of the loan request being filled, a token being given to the lender, and principal being transferred to the borrower] -->

A **Debt Order** is only considered _fillable_ once it is _signed_ by both the borrower and lender. If an **Underwriter** is involved in the transaction, their signature is required as well.

There are two ways a Debt Order can be _signed_ by a borrower, lender, or underwriter.

The first is straightforward -- the borrower / lender / underwriter produces an ECDSA signature of the **Debt Order**. The second is a bit counterintuitive -- the borrower / lender / underwriter can forego signing the **Debt Order** if _their_ address is the one that submits the **Debt Order** to the Dharma smart contracts. This is OK, given that the submitter of the **Debt Order** _must_ provide an ECDSA signature of the _whole transaction_ to the Ethereum network in order for it to be accepted into the blockchain. We know that the **Debt Order** is a part of this whole transaction, so, effectively, the submitter is signing the **Debt Order** by proxy (along with a whole bunch of other stuff).

## Relayers

**Relayers** are a crucial part of the Dharma Credit Market. Their existence solves a key problem: if a borrower needs to find a lender to _fill_ their **Debt Order**, how do they find a prospective lender?

One way for a debtor to find a creditor is to post a **Debt Order** to a centralized order book, where creditors can find and invest in these requested loans. We call the party who aggregates Debt Orders into an open order book a **Relayer**. **Relayers** can receive a fee for their service automatically through Dharma Protocol, by stipulating that fee in the **Debt Order** before it is signed by the Debtor.

There are no explicit requirements for what a **Relayer** has to look like. Some **Relayers** (like [Bloqboard](https://dapp.bloqboard.com), for instance) have websites with simple UIs that resemble interactive bulletin boards for **Debt Orders.** Others may simply be databases with thin REST APIs wrapped around them for programmatic consumption. Some may even be entirely on-chain -- it's _really_ up to the **Relayers.**

Most importantly, though, **Relayers** earn fees when **Debt Orders** are filled, meaning entrepreneurs can build functioning businesses on Dharma Protocol with minimal technical know-how of blockchain tech. We believe that **Relayers** in the Dharma Credit Market have a meaningful opportunity to, at a minimum, earn passive income by hosting a simple website, and, at a maximum, build very profitable blockchain businesses by providing a crucial service to the decentralized financial ecosystem.

## Underwriters

In Dharma Protocol, an **Underwriter** is a formal (but optional) part of the process of generating a **Debt Order**. Their existence solves another key problem in the Dharma Credit Market: how can lenders evaluate [the credit risk](https://www.investopedia.com/terms/c/creditrisk.asp) if all they know about a borrower is their Ethereum address?

An **Underwriter** is a _trusted_ party that performs the following functions:

*   Negotiates the terms of the loan with the borrower
*   Produces a **Debt Order**, based on these terms
*   Commits to a _risk rating_ of the borrower, which is the likelihood that the borrower will repay his loan in full
*   Cryptographically signs the **Debt Order**, including their risk rating for that order -- you can think of this as equivalent to "co-signing" the loan
*   Doing everything within their reasonable power to ensure timely repayment according to the loan's terms. This is known as _debt servicing_

Armed with a **Debt Order** that is "co-signed" by an **Underwriter**, a borrower is now able to broadcast this **Debt Order** to as many different **Relayers** as she likes. Since a known, **trusted** third party (i.e. the **Underwriter**) has attested to the borrower's creditworthiness, her **Debt Order** is much more likely to be _filled_ by lenders who are browsing through a **Relayer's** order book, because those lenders no longer have to put blind faith in an unknown Ethereum address.

When a **Debt Order** that was underwritten by an **Underwriter** is _filled_, that underwriter earns a fee. **More importantly, however, the Underwriter's risk rating for the loan is immutably written into the blockchain**. This means that **Underwriters** have an incentive to do a great job rating different loans. Because anyone in the world can examine an **Underwriter's** historical performance based on their on-chain attestations, if a given **Underwriter** has a _bad_ track record of assessing borrower creditworthiness, lenders will be less likely to fill their loans. If they do a _good_ job, the opposite would happen.

The functions performed by an **Underwriter** are not particularly out-of-the-norm for what most online lenders do in their day-to-day loan underwriting and servicing operations.

We think that the Dharma Credit Market will become an alternative, cheaper route for aspiring online lending platforms to bootstrap their operations and earn similar margins as they would in traditional markets by becoming an **Underwriter** -- all-the-while never holding balance sheet risk and avoiding the upfront time and capital costs that come with raising debt capital from traditional investors.

> _Note: it is crucially important to note that Underwriters **must** be trusted entities. The Underwriting mechanism of Dharma protocol is by no means trustless -- in order to learn more about this, and to get a sneak peak of how we plan on making Underwriting in Dharma more trustless, we [recommend this blog post](https://blog.dharma.io/current-and-future-approaches-to-unsecured-lending-in-dharma-protocol-8199eea720de)._
