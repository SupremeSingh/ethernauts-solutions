# Ethernauts Solutions!

This is a series of a (mostly) self-created solutions for the famous Ethernauts series by OpenZeppelin. Ethernauts is a great "capture-the-flag" style educational series designed to train Solidity developers in identifying and (hopefully avoiding) subtle vulnerabilities in contracts. 

PRs with more efficient solutions are welcome !!

## P1 - 

A **Fallback** function is a function of last resort for any smart contract. It doesn't have a unique identifier and is invoked when no other function in the contract meets the criteria for an external call. This could happen if an external party mistypes the name of a function, gives the wrong arguments, or simply calls something that doesn't exist. 

Notably, fallback functions are also called when someone sends only ether to a contract (without other information). By default, smart contracts cannot accept such Ether, but they are able to if an appropriate fallback function is implemented in them. 

**Properties** - 

 - Has no name or arguments. Can be marked payable. 
 - Can not return anything. 
 - Can be called by anyone (EOA or Contract)
 - Can be defined once per contract. 
 - It is mandatory to mark it external.
 - Limited to 2300 gas when called by another function. 

**Vulnerability** - Fallback function enables the transfer of ownership of this contract. 

**Exploit** - 

Step 1 - Call the "contribute" function with the requisite amount of seed capital 
Step 2 - Send a transaction without the data field to send a menial amount of Ether.

Useful code - *contract.sendTransaction({from: player,value: toWei(...)})*

**Learning** - 

 - Follow check-emt-interactions in fallbacks too 
 - Emit an event for any state change in a fallback 
 - Do not implement ownership change or other important state changes in a fallback
