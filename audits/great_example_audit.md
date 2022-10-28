# Issues

## **[H-1]** Counterparty can deposit less tokens than necessary

In the `deposit` function of `EscrowManager` at lines 45-48, you allow the counterparty to choose how much token they want to deposit, and then you set `depositComplete` to true without checking whether enough was deposited to meet the requirement.

```solidity
function deposit(uint256 escrowId, uint256 amountToken) external {
    ...
    ERC20(escrow.token).transferFrom(msg.sender, address(this), amountToken);
    escrow.depositComplete = true
}
```
and later you allow the counterparty to withdraw funds based on this flag:
```solidity
if (depositComplete && block.timestamp >= escrow.createdAt + 1 day) {
    escrow.counterparty.call{
        value: escrow.creatorDepositAmount
    }("");
    ...
}
```

This means that the counterparty can deposit only a small amount of funds, but still withdraw the full amount of the creator's funds.

Consider checking that `amountToken > escrow.counterpartyDepositAmount` in `deposit` and revert otherwise.

## **[H-2]** `refund` when called by creator is vulnerable to reentrancy

In the `refund` function on lines 60-68, you have:
```solidity
    if (msg.sender == escrow.creator) {
      require(!escrow.creatorRefunded, "Already refunded");
      (bool success, ) = escrow.creator.call{
        value: escrow.creatorDepositAmount
      }("");
      require(success, "Refund to creator failed");

      escrow.creatorRefunded = true;
    } else {
```

An attacker could create an escrow from contract with a `receive() payable` function in which they again call the `refund()` function. Because you set `escrow.creatorRefunded = true;` _after_ making the external call, the second call to `refund()` will succeed. The attacker could exploit this in such a way that all ETH is drained from the contract.

Consider more carefully following the Checks-Effects-Interactions pattern, and updating important contract state before making any external calls.

## **[M-1]** Return value of ETH transfers is not checked

In the `withdraw` function on lines 90-96, you have:
```solidity
    if (msg.sender == escrow.counterparty) {
      require(!escrow.counterpartyWithdrew, "Already withdrew");
      escrow.counterpartyWithdrew = true;

      escrow.counterparty.call{
        value: escrow.creatorDepositAmount
      }("");
    } else {
```

It's possible, although maybe unlikely, that the counterparty for the escrow is a contract that is only able to receive ETH under certain conditions, e.g. based on the state of a boolean flag. Suppose the contract called `withdraw` on our `EscrowManager` contract at a time when it couldn't receive ETH. In that case, the transfer in `escrow.counterparty.call` would fail but your code will still mark `escrow.counterpartyWithdrew = true` which will prevent the counterparty from attempting a withdrawal in the future.

Consider checking the `success` value of the ETH transfer and reverting if it's `false`.


## **[Technical Mistake]** Creator and counterparty can be same address

The spec states:
> Creator and counterparty cannot be the same address.

In your `create` function, you don't check whether `msg.sender` and `counterparty` are the same address. I don't think this introduces any new attack vectors, but it does deviate from the requirements.

Consider adding a check like 
```
require(msg.sender != counterparty);
```

## [Q-1] Very few emitted events

You added events in a couple places, like `create` and `withdraw`, but there are a few other functions that should emit events, such as `cancel` and `refund`.

## [Q-2] Unnecessary field on `Escrow` struct

Your escrow struct has a `creatorDeposited` field which is used to track whether the creator has already deposited their part into the escrow. Since the creator has to make their deposit in the same transaction where the escrow is created, by passing ETH as `msg.value` into the call to `create`, we know this field will always be true. So you can save some gas on storage reads and writes by removing the field and all checks on it.
