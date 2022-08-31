# TVM linker

This repository stores the source code for tvm_linker utility. It takes TVM (https://test.ton.org/tvm.pdf) 
assembly source code of TON smart contract, compiles it and links its parts, adds standard selector 
and runtime and stores it into binary TVC file. Also, it can immediately execute a smart 
contract by emulating the computing phase of TON transaction.

TVM assembly can be generated by one of TONLabs compilers:
- TON solidity compiler (https://github.com/tonlabs/TON-Solidity-Compiler)
- C and C++ compiler for TVM (https://github.com/tonlabs/TON-Compiler)

## Prerequisites

- Latest version of Rust
- Cargo tool
[Get them here](https://doc.rust-lang.org/cargo/getting-started/installation.html)

## How to build

```bash
$ cargo update && cargo build --release -j8
```

## How to use

tvm_linker has several modes of work:

### 1) Generating a ready-to-deploy contract.

```bash
$ tvm_linker compile [--lib <lib_file>] [--abi-json <abi_file>] [-w <workchain_id>] [--debug] [--print_code] [--silent] [--debug-map <debug_info_path>] <source>
```

Here `source` is a name of tvm assembly source file, `library` is a runtime library file (can be more than one: `--lib` 
should be supplied for every file). If `--lib` option is not specified linker looks for environment variable
`TVM_LINKER_LIB_PATH`, if it is set that path is used to load a library.

If there is an ABI file, it is better to use `--abi-json` option to supply a contract ABI file. Function ID's are
generated according to function signatures in the ABI. If neither `-a` nor `--abi-json` option is specified, linker
checks whether file `source`(without extension) + `.abi.json` exists. If file exists, linker loads ABI from it.

Linker generates the `<address>.tvc` file, where `<address>` is a hash from initial data and code of the contract.

Linker prints initial contract address in different formats: raw and user-friendly (testnet and mainnet). Define the workchain
ID option `-w` to generate proper user-friendly address. -1 is used by default.

To add a key to the contract data and obtain real contract address user should use [`tonos-cli genaddr` command](https://github.com/tonlabs/tonos-cli/blob/master/README.md#41-generate-contract-address). 

While execution if option `--debug-map <debug_info_path>` is specified, this command can generate a debug info file, 
which contains mapping that can bind contract code with source files. This file can be used while debugging the
contract.

`--print_code` option allows user only generate code and print it without creating the TVC file.
`--silent` option mutes all extra notifications.

### 2) Decoding of .boc messages prepared externally.
To use this method, call

```bash
$ tvm_linker decode [--tvc] boc-file
```

If `--tvc` is omitted, `boc-file` is a file with a serialized message, otherwise it is a contract `tvc` file.

### 3) Preparing an external inbound messages in .boc format.

First, generate a contract as described in 1). Then use `message` subcommand to create external inbound message in boc
format:

```bash
$ tvm_linker message <contract-address> [--init] [--data] [-w]
$ tvm_linker message <contract-address> [--init] --abi-json <abi_file> --abi-method <method_name> --abi-params {json_with_params} [-w]
```

`contract-address` - the name of the compiled contract file without extension. The contract .tvc file should be placed in the current directory.

To create a `constructor message` with contract's code and data, use `--init` option.

Additionally, you can add a raw body to the message. Use the `--data` option:

```bash
$ tvm_linker message <contract-address> --data <data_string>
```

Instead of `<data_string>`, specify the necessary message body in hex format. 

Or make a message with ABI call using combination of options:
- `--abi-json <abi_file>` - path to a .json with contract interface described according to ABI specification;
- `--abi-method <method-name>` - name of the contract method to call;
- `--abi-params {<json-string-with-params>}` - arguments of the method declared in json like this: `{"arg_a": "0x1234", "arg_b": "x12345678"}`.

By default, -1 is used as a workchain id in contract address. To use another one, use `-w` option:

```bash
$ tvm_linker message -w 0
```

### 4) Emulating contract execution:

Linker can emulate compute phase of blockchain transaction. It is useful for contract debugging.

```bash
$ tvm_linker test <contract-address> --body XXXX... [--sign key-file] [--trace] [--decode-c6] [--internal <value>] [--src address] [--now unixtime] [-s source-file] [--balance <value>]
```

Loads contract from file by contract address `address` and emulates contract call sending external inbound message (by default) with body defined after `--body` parameter to the contract. `XXXX...` is a hex string. 

If `--sign` specified, the body will be signed with the private key from `key-file` file.

Use `--trace` flag to trace VM execution: stack, registers and gas will be printed after each executed VM command.

Use `--decode-c6` to see output actions in user-friendly format.

Use `--balance <value>` to define account balance in nanograms. It will be available  at the bottom of initial stake and in SmartContractInfo tuple from c7 register .

Use `--config <tvc_file>` to define the config parameters to run VM with. The TVC file is a state of the config smart-contract. 
The capabilities field of the config defines the VM mode of operation. If the config parameter is omitted, the capabilities default value of 0x42E is used. 
For the available capability codes consult [here](https://github.com/tonlabs/ton-labs-block/blob/master/src/config_params.rs#L336)

Note: configuration smart-contract resides at the address: -1:5555555555555555555555555555555555555555555555555555555555555555


Use `--internal` to send internal message to the contract with defined nanograms in `value`. By default, source address in internal message in zero address (`0000...0000`), to define another address use option `--src <address>`, where address should be in the format <wc>:<bytes32> (i.e. "0:1122...AABB"). 

Account and message balance can have extended format with extra currencies: `{ "main": int, "extra": {"i": int, ...} }`.

Example: `--internal 100000 --src "0:6011b66a47238cf992f1033fe6aff00ce0f850df387ee92468d9c26b5564ba53"`

Use `--now <unixtime>` option to define transaction creation time. By default, current time is used.

Use `--bounced` flag to emulate bounced internal message, use this flag only with `--internal` option.

An ABI body can be generated if `abi-params`, `abi-json` and `abi-method` will be used instead of `--body XXXX...`.

If `--body` is used, contract's public function ids can be encoded by their names using `$...$` syntax:`$name:[0len][type]$`, 
where `name` is a name of public function, `len` - length in chars of the id (if `len` is bigger than `name`'s length in chars than 
zeros will be added on the left side to fit required length), `type` can be `x` or `X` - hexadecimal integer  in lowercase or uppercase. You have to set `-s source` option when you use $...$ syntax.

Example:

```bash
$ tvm_linker address test --body 00$main$ -s source //main id will be inserted as decimal string. Dont use this case, just as example
$ tvm_linker address test --body 00$main:08X$ -s source
$ tvm_linker address test --body 00$main:X$ -s source
$ tvm_linker address test --body 00$main:x$ -s source
```

The `--body-from-boc` option is analogous to `--body` but extracts the message body from the specified message boc file.

### 5) Disassembler

There are a number of tools under the `disasm` umbrella:

`dump` outputs a pseudo-graphical representation of a tree of cells.
`text` disassembles a tvc produced by Solidity and FunC compilers.
`graphviz` produces an output in dot format for generation of graphical DAG representation of a tvc.

### More Help
Use `tvm_linker --help` for detailed description about all options, flags and subcommands.

## Input format

As a temporary measure, some LLVM-assembler like input is used. The source code should contain several function types:

- `.internal` - special functions, which are used only by contract's runtime. There are some wellknown internal functions:

	`main_external`, `main_internal`, `onTickTock`, `onBounce`.

## Support

Get more documents at docs.ton.dev and check our [YouTube Channel](https://www.youtube.com/channel/UC9kJ6DKaxSxk6T3lEGdq-Gg) for tutorials. Stay tuned.
