# 2. Fallout
Контракт реализует систему выделения денежных средств. Позволяет получать на контракт эфир от распределителей, возвращать обратно. Владелец может отправить все средства в контракте себе.

Код контракта:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

В Solidity до версии 0.5.0 для объявления конструктора контракта использовался метод с тем же именем что и у контракта.
Из-за этой особенности шанс совершить ошибку в названии конструктора был больше, из-за чего эту функцию можно было вызвать не единожды, что и представлено в данном контракте.
В новых версиях используется специальная функция `constructor()`.

Чтобы завладеть контрактом достаточно вызвать функцию `Fal1out()`.

Исправленный код контракта:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract Fallout {
    mapping(address => uint256) allocations;
    address public immutable OWNER;

    error NotOwner();
    error TransferFailed();

    constructor() payable {
        OWNER = msg.sender;
        allocations[OWNER] = msg.value;
    }

    function _checkOwner() private view {
        if (msg.sender != OWNER) {
            revert NotOwner();
        }
    }

    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] += msg.value;
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        (bool success, ) = allocator.call{value: allocations[allocator]}("");
        if (!success) {
            revert TransferFailed();
        }
    }

    function collectAllocations() public onlyOwner {
        (bool success, ) = OWNER.call{value: address(this).balance}("");
        if (!success) {
            revert TransferFailed();
        }
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

Note: зачем только тут вообще функцию `sendAllocation(address payable allocator)`? Она может вызываться любым и я так понимаю, возвращает средства выделителю.