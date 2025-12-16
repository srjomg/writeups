# Intro
"Each puzzle consists on sending a successful transaction to a contract. The bytecode of the contract is provided, and you need to fill the transaction data that won't revert the execution."
Challenge by [Franco Victorio](https://github.com/fvictorio).
[Original repo](https://github.com/fvictorio/evm-puzzles)

These are my solutions.

# Level 1
Bytecode:
```
3456FDFDFDFDFDFD5B00
```
Mnemonic:
```
[00]	CALLVALUE	
[01]	JUMP	
[02]	REVERT	
[03]	REVERT	
[04]	REVERT	
[05]	REVERT	
[06]	REVERT	
[07]	REVERT	
[08]	JUMPDEST	
[09]	STOP
```

`CALLVALUE` кладет на стек значение в `wei`, перееданное в контракт при вызове.
`REVERT` прерывает выполнение программы.
`JUMP` unconditionally jumps to a target address specified on the stack, but only if the byte at that address is a `JUMPDEST`.

Therefore, we need to call this contract with a valid jump destinasion - `8`.

# Level 2
Bytecode:
```
34380356FDFD5B00FDFD
```
Mnemonic:
```
[00]	CALLVALUE	
[01]	CODESIZE	
[02]	SUB	
[03]	JUMP	
[04]	REVERT	
[05]	REVERT	
[06]	JUMPDEST	
[07]	STOP	
[08]	REVERT	
[09]	REVERT
```

`CODESIZE` кладет на стек длину всего кода контракта (в нашем случае это `10`).
`SUB` берет два значения из стека и вычитает из первого второе.

Нам нужно вызвать контракт и передать столько `wei`, чтобы `code_size - call_value = 6`, то есть `10 - call_value = 6`, значит `call_value = 4`.

# Level 3
Bytecode:
```
3656FDFD5B00
```
Mnemonic:
```
[00]	CALLDATASIZE	
[01]	JUMP	
[02]	REVERT	
[03]	REVERT	
[04]	JUMPDEST	
[05]	STOP
```

`CALLDATASIZE` кладет на стек длину переданный данных вызова.
То есть, нужно передать любые данные длиной `4`, например `0x11223344`.

# Level 4
Bytecode:
```
34381856FDFDFDFDFDFD5B00
```
Mnemonic:
```
[00]	CALLVALUE	
[01]	CODESIZE	
[02]	XOR	
[03]	JUMP	
[04]	REVERT	
[05]	REVERT	
[06]	REVERT	
[07]	REVERT	
[08]	REVERT	
[09]	REVERT	
[0a]	JUMPDEST	
[0b]	STOP
```

`XOR` берет со стека два значения и выполняет между ними операцию XOR, а результат кладет на стек.

Фактически нужно, чтобы выполнилось `call_value ^ code_size = 0x0a`, где `code_size = 0x0c`, то есть `call_value ^ 0x0c = 0x0a`, далее, как известно: `call_value = 0x0a ^ 0x0c = 6`. То есть нам нужно вызвать контракт и передать туда `6 wei`.

# Level 5
Bytecode:
```
34800261010014600C57FDFD5B00FDFD
```
Mnemonic:
```
[00]	CALLVALUE	
[01]	DUP1	
[02]	MUL	
[03]	PUSH2	0100
[06]	EQ	
[07]	PUSH1	0C
[09]	JUMPI	
[0a]	REVERT	
[0b]	REVERT	
[0c]	JUMPDEST	
[0d]	STOP	
[0e]	REVERT	
[0f]	REVERT	
```

`DUP1`
`PUSH2`
`EQ`
`PUSH1`
`JUMPI`
