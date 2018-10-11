---
title: Rhombus API Reference

# language_tabs: # must be one of https://git.io/vQNgJ
#  - csharp

# toc_footers:
# - <a href='#'>Sign Up for a Developer Key</a>
# - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

# includes:
#  - errors

search: true
---

# Rhombus API Reference

Rhombus divides oracles into two broad buckets: how the oracle is activated and
how the oracle replies. Our API for running oracles has predictable,
resource-oriented functions.

# Activation Methods

## Periodic oracles

When a contract needs regularly updated information, a periodic Rhombus oracle
is often the best solution. The oracle checks the data sources at regular
intervals and updates the on-chain information whenever significant changes to
that information have occurred.

<aside class="example">
<span class="example-text">Example</span>
<h3>
Periodic gold prices for an exchange contract selling gold backed
tokens.
</h3>

<p>
Every 5 minutes a Rhombus Oracle aggregates several gold price suppliers and
returns the median price to an exchange contract. If there is an anomaly in
the prices recovered, the exchange is informed of that fact so that it may
stop trading if necessary.
</p>

</aside>

<aside class="notice">
  The user could save even more gas costs by requesting that the reply is only
  sent when a certain price variation occurs, e.g. more than 1%.
</aside>

## Asynchronous oracles

A contract may require information on an ad-hoc basis. A client would initiate
these requests from their smart contract. A request may contain a number of
parameters but should also contain a Unique ID (UID) tied to their oracle.
Additionally, a nonce may be supplied to ensure that replies match requests.

<aside class="example">
<span class="example-text">Example</span>
<h3>Resolving a Prediction Market</h3>

<p>
At some day and time in the future, a prediction market will want to settle
the outstanding bets for a given proposition, like gas price. When the time
comes, it mines an event that activates a Rhombus Oracle which delivers the
current gas price to the contract.
</p>
</aside>

# Reply Methods

## Lighthouse contract

Rhombus has a pre-written contract that we can deploy to decouple your oracle
from your contract. They are fast to deploy and easily observed before
committing to integrating the data.

Features of a lighthouse contract

- One or more data items as specified
- A nonce - incremental or returned serial number
- Time of update written into contract
- Optional aging data expiry mechanism
- Optional “poke” mechanism when updated

<aside class="example">
<span class="example-text">Example</span>
<h3>A lighthouse contract that contains hourly asset price data</h3>

<p>
A market maker wants to observe the performance of a feed before integrating.
After a month passes, they are confident in its stability, consistency, and
performance. They integrate the price data into several of their smart
contracts.
</p>

</aside>

## Direct delivery

> **Example:** The gold spot price must be written directly to the gold exchange contract

```csharp
function setSpotPrice(uint spotPrice, uint nonce);
```

> `setSpotPrice` function signature is specified prior to delivery so the
> Rhombus oracle can call it properly.

Where the Rhombus oracle is the sole information provider, it may be more
efficient to write directly to your contract. The oracle can provide one or more
data items as well as a nonce which helps serialize replies.

# Getting Started

Below we'll demonstrate sample integrations with both Direct and Lighthouse
delivery oracles.

## Direct Delivery

> Direct Delivery Example
> The example below

```csharp
pragma solidity ^0.4.24;

import "https://github.com/RhombusNetwork/rhombus-demos/blob/master/rhombus/rhombusClient.sol";

contract FundRaiser is RhombusClient {

    struct Donation {
        address donor;
        uint    amountInUSD;
    }

    Donation[] public donations;
    address public constant alice =  0x31EFd75bc0b5fbafc6015Bd50590f4fDab6a3F22;

    event Raised(address donor, uint USDraised);

    // On receiving a donation (in ether), we ask the Rhombus ETH to USD oracle
    // to convert the amount but forward the amount to our favourite charity,
    // Alice's Food Kitchen.

    function () public payable {
        if (msg.value == 0)
            return;
        Donation memory thisDonation;
        thisDonation.donor = msg.sender;
        uint pos = donations.push(thisDonation);
        emitDoubleUint(0, pos-1, msg.value);
        alice.transfer(msg.value);
    }

    // On receiving the USD value, the donation amount is updated and an event
    // emitted. This function can only be called with valid, unused nonces.

    function postUSDvalue(uint index, uint USDvalue) public onlyRhombus {
        require(index < donations.length, "Invalid Index");
        Donation storage d = donations[index];
        require(d.amountInUSD == 0, "Record already updated");
        d.amountInUSD = USDvalue;
        emit Raised(d.donor, d.amountInUSD);
    }

}
```

Alice's food kitchen has fallen on hard times so she launches a fundraiser on
the ethereum blockchain.

Alice want's to thank each participant for their donation not in ETH terms but
in USD terms. Rhombus, sympathetic to her needs, has agreed to provide an ETH/USD
oracle.

On receiving a donation, the donor's address is logged and the oracle is asked
to convert the eth value to USD. The donation is forwarded to Alice.

The oracle executes a direct reply to the contract's postUSDvalue function which
causes the donations log to be updated and a Raised event to be emitted which
Alice can monitor to publish a thank you note.

## Lighthouse Delivery

> Lighthouse Contract Example
> 10 cents per minute

```csharp
pragma solidity ^0.4.24;

import "https://github.com/RhombusNetwork/rhombus-demos/blob/master/lighthouse/Ilighthouse.sol";

contract TenCentsAMinute {

    ILighthouse  public myLighthouse;
    mapping(address => uint) public balances;
    mapping(address => uint) public expiry;
    uint public spentFees;
    address public constant alice =  0x31EFd75bc0b5fbafc6015Bd50590f4fDab6a3F22;

    constructor(ILighthouse _myLighthouse) public {
        myLighthouse = _myLighthouse;
    }

    function() public payable {
        balances[msg.sender] += msg.value;
    }

    function buyTime(uint minutesToBuy) public {
        require(minutesToBuy != 0, "nothing to buy");
        uint tenCentsOfETH;
        bool ok;
        (tenCentsOfETH,ok) = myLighthouse.peekData();
        uint fee = tenCentsOfETH * minutesToBuy;
        require(fee / minutesToBuy == tenCentsOfETH,"Overflow");
        require(fee < balances[msg.sender],"Not enough funds");
        expiry[msg.sender] = now + minutesToBuy * 1 minutes;
        spentFees += fee;
    }

    function inPaidTime() public view returns (bool) {
        return now < expiry[msg.sender];
    }

    function withdrawFees() public {
        require(alice == msg.sender, "Unauthorised");
        alice.transfer(spentFees);
        spentFees = 0;
    }
}
```

Alice's petting zoo also operates a video feed service that allows you to watch
videos of the animals for a nominal fee of ten cents a minute.

Since Alice is an ethereum fan, she controls the payments using an ethereum
smart contract but that means she gets payments in ether not cents.

Rhombus, well known supporters of petting zoos worldwide, agree to provide a
lighthouse supplying regularly updated data for the value of ten cents in eth.

The contract that Alice wrote provides a function inPaidTime() that can be
called by the camera's streaming code to query whether the user has paid for
their video time.
