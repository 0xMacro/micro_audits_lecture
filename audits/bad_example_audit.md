# Issues

## **[H-1]** Counterparty can cheat

You don't check that they deposited the full amount. That means they can cheat the system by depositing a really small amount.

## **[H-2]** Reentrancy vulnerability in `refund`

An attacker can use reentrancy to steal funds when they call refund.

## **[Technical Mistake]** Creator and counterparty can be same address

The function is supposed to fail when the two address are equal.

## **[Q-1]** Change order of function signature

I would change the order of the arguments in the function signature for createEscrow. I think it's always better to have variables in alphabetical order.

