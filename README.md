RTBkit
======

RTBkit is an open-source software package that takes the hard work out of creating a Real-Time Bidder.

http://rtbkit.org/

[Getting Started](https://github.com/rtbkit/rtbkit/wiki/Getting Started) (Installation and build instructions)


# How to change shadow account data:

1. Start redis cli:
```shell
> redis-cli
> get "banker-<campaignID>:pacerAccount:PostAuctionLoop.slaveBanker"
```
The PostAuctionLoop shadow account stored in redis would look like:
```json
{
    "adjustmentLineItems":{},
    "adjustmentsIn":{},
    "adjustmentsOut":{},
    "allocatedIn":{},
    "allocatedOut":{},
    "budgetDecreases":{},
    "budgetIncreases":{"USD/1M":99550},
    "commitmentsMade":{},
    "commitmentsRetired":{"USD/1M":450},
    "lineItems":{},
    "md":{"objectType":"Account","version":1},
    "recycledIn":{},
    "recycledOut":{},
    "spent":{},
    "status":"active",
    "type":"spend"
}
```

2. To submit shadow account JSON using rtbkit's `POST,PUT /v1/accounts/<accountName>/shadow` banker REST API, one have to come up with a shadow account JSON that looks like this:
```json
{
    "balance":{"USD/1M":98700},
    "commitmentsMade":{"USD/1M":10000},
    "commitmentsRetired":{"USD/1M":10200},
    "lineItems":{},
    "md":{"objectType":"ShadowAccount","version":1},
    "netBudget":{"USD/1M":99500},
    "spent":{"USD/1M":1000}
}
```
Among which, `netBudget` and `balanace` can be calculated based on the PostAuctionLoop shadow account info aquired from redis by
```C++
netBudget = (budgetIncreases - budgetDecreases
            + allocatedIn - allocatedOut
            + recycledIn - recycledOut
            + adjustmentsIn - adjustmentsOut);

balance = netBudget + commitmentsRetired
            - commitmentsMade - spent;
```
`spent`, `commitmentsMade` and `commitmentsRetired` can be changed. 
> NOTE that both `commitmentsMade` and `commitmentsRetired` can only be increased but cannot be decreased from the redis PostAuctionLoop shadow account JSON value.

3. Before submitting the new shadow account data to rtbkit banker, make sure its `credit` equals to `debit`:
```C++
credit = (budgetIncreases + recycledIn + commitmentsRetired
            + adjustmentsIn + allocatedIn);

debit = (budgetDecreases + recycledOut + commitmentsMade + spent
            + adjustmentsOut + balance
            + allocatedOut);
```

4. Now use curl or node command line to submit the data, an example of curl:
```shell
curl --header "Content-Type: application/json" --request POST --data '{ "balance":{"USD/1M":98700}, "commitmentsMade":{"USD/1M":10000}, "commitmentsRetired":{"USD/1M":10200}, "lineItems":{}, "md":{"objectType":"ShadowAccount","version":1}, "netBudget":{"USD/1M":99500}, "spent":{"USD/1M":1000} } ' http://localhost:9985/v1/accounts/5b7dc9e4afd43403b0e70068:pacerAccount:PostAuctionLoop.slaveBanker/shadow
```