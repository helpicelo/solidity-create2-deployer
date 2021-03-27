![GitHub Workflow Status](https://img.shields.io/github/workflow/status/thegostep/solidity-create2-deployer/CI)
![npm](https://img.shields.io/npm/v/solidity-create2-deployer)
![node-current](https://img.shields.io/node/v/solidity-create2-deployer)
![GitHub last commit](https://img.shields.io/github/last-commit/thegostep/solidity-create2-deployer)
![npm](https://img.shields.io/npm/dw/solidity-create2-deployer)
![NPM](https://img.shields.io/npm/l/solidity-create2-deployer)

# Solidity `CREATE2` Deployer

This library is a minimal utility for deploying ethereum contracts at detereministic addresss using `CREATE2`. It allows for contracts to be deployed at the same address on all networks.

[`CREATE2`](https://github.com/ethereum/EIPs/pull/1014) opcode was released in the [Constantinople](https://github.com/paritytech/parity-ethereum/issues/8427) update for Ethereum.

## Example Usage

```js
// import
const {
  ethers,
  deployContract,
  deployFactory,
  getCreate2Address,
  isDeployed
} = require("solidity-create2-deployer");

// declare deployment parameters
const salt = "hello";
const bytecode = "0x...";
const privateKey = "0x...";
const constructorTypes = ["address", "uint256", "..."];
const constructorArgs = ["0x...", "123...", "..."];
const provider = ethers.getDefaultProvider();
const signer = new ethers.Wallet(privateKey, provider);

// Calculate contract address
const computedAddress = getCreate2Address({
  salt: salt,
  contractBytecode: bytecode,
  constructorTypes: constructorTypes,
  constructorArgs: constructorArgs
});

// Deploy contract
const { txHash, address, receipt } = await deployContract({
  salt: salt,
  contractBytecode: bytecode,
  constructorTypes: constructorTypes,
  constructorArgs: constructorArgs,
  signer: signer
});

// Query if contract deployed at address
const success = await isDeployed(address, provider);

// Deploy create2 factory (for local chains only)
const factoryAddress = await deployFactory(provider);
```

## Caveats

Contracts deployed using this library need to follow these guidelines:

- `msg.sender` cannot be used in the constructor as it will refer to the factory contract.
- `tx.origin` should not bee used in the constructor as the deploy transaction can be front-run.
- In order to produce a deterministic address on all networks, the salt and constructor parameters must be the same.

## API Documentation

```js
/**
 * Deploy contract using create2.
 *
 * Deploy an arbitrary contract using a create2 factory. Can be used with an ethers provider on any network.
 *
 * @param {Object} args
 * @param {String} args.salt                Salt used to calculate deterministic create2 address.
 * @param {String} args.contractBytecode    Compiled bytecode of the contract.
 * @param {Object} args.signer              Ethers.js signer of the account from which to deploy the contract.
 * @param {Array}  [args.constructorTypes]  Array of solidity types of the contract constructor.
 * @param {Array}  [args.constructorArgs]   Array of arguments of the contract constructor.
 *
 * @return {Object} Returns object with `txHash`, `address` and `receipt` from the deployed contract.
 */

/**
 * Calculate create2 address of a contract.
 *
 * Calculates deterministic create2 address locally.
 *
 * @param {Object} args
 * @param {String} args.salt                Salt used to calculate deterministic create2 address.
 * @param {String} args.contractBytecode    Compiled bytecode of the contract.
 * @param {Array}  [args.constructorTypes]  Array of solidity types of the contract constructor.
 * @param {Array}  [args.constructorArgs]   Array of arguments of the contract constructor.
 *
 * @return {String} Returns the address of the create2 contract.
 */

/**
 * Determine if a given contract is deployed.
 *
 * Determines if a given contract is deployed at the address provided.
 *
 * @param {String} address  Address to query.
 * @param {Object} provider Ethers.js provider.
 *
 * @return {Boolean} Returns true if address is a deployed contract.
 */

/**
 * Deploy create2 factory for local development.
 *
 * Deploys the create2 factory locally for development purposes. Requires funding address `0x2287Fa6efdEc6d8c3E0f4612ce551dEcf89A357A` with eth to perform deployment.
 *
 * @param {Object} provider Ethers.js provider.
 *
 * @return {String} Returns the address of the create2 factory.
 */
```

## Tutorial

These tutorial will show you how to predetermine a smart contract address off-chain and then deploy using `create2` from a smart contract.

`Factory.sol` - a contract that deploys other contracts using the `create2` opcode:

```solidity
pragma solidity >0.4.99 <0.6.0;

contract Factory {
  event Deployed(address addr, uint256 salt);

  function deploy(bytes memory code, uint256 salt) public {
    address addr;
    assembly {
      addr := create2(0, add(code, 0x20), mload(code), salt)
      if iszero(extcodesize(addr)) {
        revert(0, 0)
      }
    }

    emit Deployed(addr, salt);
  }
}
```

`Account.sol` - the contract to counterfactual instantiate:

```solidity
pragma solidity >0.4.99 <0.6.0;

contract Account {
  address public owner;

  constructor(address payable _owner) public {
    owner = _owner;
  }

  function setOwner(address _owner) public {
    require(msg.sender == owner);
    owner = _owner;
  }

  function destroy(address payable recipient) public {
    require(msg.sender == owner);
    selfdestruct(recipient);
  }

  function() payable external {}
}
```

Create helper functions:

```js
// deterministically computes the smart contract address given
// the account the will deploy the contract (factory contract)
// the salt as uint256 and the contract bytecode
function buildCreate2Address(creatorAddress, saltHex, byteCode) {
  return `0x${ethers.utils
    .keccak256(
      `0x${["ff", creatorAddress, saltHex, ethers.utils.keccak256(byteCode)]
        .map(x => x.replace(/0x/, ""))
        .join("")}`
    )
    .slice(-40)}`.toLowerCase();
}

// converts an int to uint256
function numberToUint256(value) {
  const hex = value.toString(16);
  return `0x${"0".repeat(64 - hex.length)}${hex}`;
}

// encodes parameter to pass as contract argument
function encodeParam(dataType, data) {
  const abiCoder = ethers.utils.defaultAbiCoder;
  return abiCoder.encode([dataType], [data]);
}

// returns true if contract is deployed on-chain
async function isContract(address) {
  const code = await ethers.provider.getCode(address);
  return code.slice(2).length > 0;
}
```

Now you can compute off-chain deterministically the address of the account contract:

```js
// constructor arguments are appended to contract bytecode
const bytecode = `${accountBytecode}${encodeParam(
  "address",
  "0x262d41499c802decd532fd65d991e477a068e132"
).slice(2)}`;
const salt = 1;

const computedAddr = buildCreate2Address(
  factoryAddress,
  numberToUint256(salt),
  bytecode
);

console.log(computedAddr); // "0x45d673256f870c135b2858e593653fb22d39795f"
console.log(await isContract(computedAddr)); // false (not deployed on-chain)
```

You can send eth to the precomputed contract address `0x45d673256f870c135b2858e593653fb22d39795f` even though it's not deployed. Once there's eth in the contract you can deploy the contract and have the funds sent to a different address if you wish. CREATE2 is useful because you don't need to deploy a new contract on-chain for new users; you or anyone can deploy the contract only once there's already funds in it (which the contract can have refund logic for gas).

Let's deploy the account contract using the factory:

```js
const signer = (await ethers.getSigners())[0];
const factory = new ethers.Contract(factoryAddress, factoryAbi, signer);
const salt = 1;
const bytecode = `${accountBytecode}${encodeParam(
  "address",
  "0x262d41499c802decd532fd65d991e477a068e132"
).slice(2)}`;
const result = await (await factory.deploy(bytecode, salt)).wait();

const addr = result.events[0].args.addr.toLowerCase();
console.log(computedAddr == addr); // true (deployed contract address is the same as precomputed address)
console.log(result.transactionHash); // "0x4b0f212af772aab80094b5fe6b5f3f3c544c099d43ce3ca7343c63bbb0776de4"
console.log(addr); // "0x45d673256f870c135b2858e593653fb22d39795f"
console.log(await isContract(computedAddr)); // true (deployed on-chain)
```

Example code found [here](./test/).

## Development

Install dependencies:

```bash
yarn
```

Test contracts:

```bash
yarn test
```

## Resources

- [EIP 1014: Skinny CREATE2](https://eips.ethereum.org/EIPS/eip-1014)

## Credits

- [@stanislaw-glogowski](https://github.com/stanislaw-glogowski/CREATE2) for initial implementation example
- [@miguelmota](https://github.com/miguelmota/solidity-create2-example) for factory implementation example with web3

## License

[MIT](LICENSE)
