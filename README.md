> _“It's not improbable that a man may receive more solid satisfaction from pudding while he is alive than from praise after he is dead”_

-- Chinese Proverb

### Ether Pudding

Ether Pudding (or just “Pudding”) is a packager and build artifact manager for Ethereum and Javascript. It turns ABIs, binaries, deployed addresses, etc. into Javascript files you can include in your project with simply a `require`. e.g., it lets you do this:


```javascript
var MyContract = require("./MyContract.sol.js");
MyContract.setProvider(myWeb3Provider);
MyContract.deployed().then(function(instance) {
	return instance.doStuff();
}).then(function(result) {
  // We just talked to the ethereum network!
	// And we get the transaction hash, logs (events) and receipt.
	console.log(result.tx, result.logs, result.receipt);
});
```

👏

### Features

* Manages contract ABIs, binaries and deployed addresses, so you don't have to.
* Packages up build artifacts into `.sol.js` files, which can then be included in your project with a simple `require`.
* Includes multiple versions of the same contract in a single package, automatically detecting which artifacts to use based on the network version (more on this below).
* Manages library addresses for linked libraries.
* Manages events, making them available on a per-transaction basis (no more `event.watch()`!)

See the following discussion for more features.

### The State of Web3: A Discussion

We can't go on without noting that Pudding was originally created to be an easier-to-use contract abstraction for Web3. However, Web3 [plans to include a lot of Pudding's features](https://github.com/ethereum/EIPs/issues/68) (like promises) and so Pudding has pivoted to focus on package management. Pudding's original abstraction API still exists, but may undergo changes once the related EIP has been merged and published.

Some of the features Pudding provides over the current abstraction:

* Synchronized transactions, so your app/tests won’t receive their callbacks until the transactions have been processed.
* Promises. No more callback hell.
* Default values for transactions, like `from` address or `gas`.
* Returning logs, transaction receipt and transaction hash of every synchronized transaction

### Install

```
$ npm install ether-pudding
```

### Usage

Pudding consists of two parts: The code generator that generates your `.sol.js` files, and then the `.sol.js` files themselves.

#### Code Generator (Pudding)

Generate `.sol.js` files given a contract name and contract data, structured as an object. This will save a `.sol.js` file into the destination directory for each contract specified.

**Note:** Pudding isn't intended to be used in the browser or in a package manager like browserify or webpack. The resultant `.sol.js` files **_are_** well suited for the browser, however.

```javascript
var Pudding = require("ether-pudding");

var contract_data = {
	abi: ...,              // Array; required.
	unlinked_binary: "..." // String; optional.
	address: "..."         // String; optional.
};

Pudding.save(contract_data, "./MyContract.sol.js").then(function() {
  // The file ./MyContract.sol.js now exists, which you can
  // import into your project like any other Javascript file.
});
```

Note that in the example above, there are three important pieces of data:

* `abi`: The ABI of your contract, provided by the solidity compiler.
* `unlinked_binary`: The compiled binary code of your contract. Ensure that if your contract relies on libraries to be linked, you pass in the *unlinked* version here.
* `address`: An address this contract is deployed to, if it's a singleton.

#### Contract Abstraction (`.sol.js` files)

Once a `.sol.js` file has been created, using it is easy. First require the generated `.sol.js` file and then set its Web3 provider:

```javascript
var provider = new Web3.providers.HttpProvider("http://localhost:8545");

var MyContract = require("./MyContract.sol.js");
MyContract.setProvider(provider);
```

Once you've imported your `.sol.js` file, you have access to the following functions:

* `at()`: Create an abstraction for your contract that exists at a specific address.
* `deployed()`: Create an abstraction for your contract that's been saved within the `.sol.js` file itself, perhaps by [Truffle](http://github.com/Consensys/truffle) or another development environment.
* `new()`: Deploy a new version of this contract, getting an abstraction that represents the newly deployed instance.

The above functions create an "instance" of the abstraction which is tied to a specific address on the Ethereum network, and each instance has a 1-to-1 mapping from Javascript functions to contract functions. For instance, if your contract had a function defined  `someFunction(uint value) {}` (solidity), then you could execute that function on the network like so:

  ```javascript
  var deployed;
  MyContract.deployed().then(function(instance) {
    var deployed = instance;
    return instance.someFunction(5);
  }).then(function(result) {
    // Do something with the result or continue with more transactions.
  });
  ```  

The abstraction itself -- i.e., `MyContract` in the example above -- has many other useful functions as well. See the Contract Abstraction API below for more information.

#### Full Example

Let's use Pudding with an example contract from [Dapps For Beginners](https://dappsforbeginners.wordpress.com/tutorials/your-first-dapp/):

```javascript
// Require the package
var MetaCoin = require("./path/to/MetaCoin.sol.js");

// Remember to set the Web3 provider (see above).
MetaCoin.setProvider(provider);

// In this scenario, two users will send MetaCoin back and forth, showing
// how Pudding allows for easy control flow.
var account_one = "5b42bd01ff...";
var account_two = "e1fd0d4a52...";

// Note our MetaCoin contract exists at a specific address.
var contract_address = "8e2e2cf785...";
var coin;

MetaCoin.at(contract_address).then(function(instance) {
  coin = instance;

  // Make a transaction that calls the function `sendCoin`, sending 3 MetaCoin
  // to the account listed as account_two.
  return coin.sendCoin(account_two, 3, {from: account_one});
}).then(function(result) {
  // This code block will not be executed until Pudding has verified
  // the transaction has been processed and it is included in a mined block.
  // Pudding will error if the transaction hasn't been processed in 120 seconds.

  // Since we're using promises, we can return a promise for a call that will
  // check account two's balance.
  return coin.balances.call(account_two);
}).then(function(balance_of_account_two) {
  alert("Balance of account two is " + balance_of_account_two + "!"); // => 3

  // But maybe too much was sent. Let's send some back.
  // Like before, will create a transaction that returns a promise, where
  // the callback won't be executed until the transaction has been processed.
  return coin.sendCoin(account_one, 1.5, {from: account_two});
}).then(function(result) {
  // Again, get the balance of account two
  return coin.balances.call(account_two)
}).then(function(balance_of_account_two) {
  alert("Balance of account two is " + balance_of_account_two + "!") // => 1.5
}).catch(function(err) {
  // Easily catch all errors along the whole execution.
  alert("ERROR! " + err.message);
});
```

Pudding manages your contract's binary code as well, so you can easily deploy new contracts of the same type using the abstraction:

```javascript
MetaCoin.new().then(function(coin) {
  // From here, the example becomes just like the above.
  // Note that coin.address is the address of the newly created contract.
  return coin.sendCoin(...);
)}.catch(function(err) {
  console.log("Error creating contract!");
  console.log(err.stack);
)};
```

### Contract Abstraction API

Each contract abstraction -- `MyContract` in the examples above -- have the following useful functions:

#### `MyContract.new([arg1, arg2, ...], [tx params])`

This function take whatever contructor parameters your contract requires and deploys a new instance of the contract to the network. There's an optional last argument which you can use to pass transaction parameters including the transaction from address, gas limit and gas price. This function returns a Promise that resolves into a new instance of the contract abstraction at the newly deployed address.

#### `MyContract.at(address)`

This function creates a new instance of the contract abstraction representing the contract at the passed in address. Returns a "thenable" object (not yet an actual Promise for backward compatibility). Resolves to a contract abstraction instance after ensuring code exists at the specified address.

#### `MyContract.deployed()`

Creates an instance of the contract abstraction representing the contract at its deployed address. The deployed address is a special value given to Pudding (see below) that, when set, saves the address internally so that the deployed address can be inferred from the given Ethereum network being used. This allows you to write code referring to a specific deployed contract without having to manage those addresses yourself. Like `at()`, `deployed()` is thenable, and will resolve to a contract abstraction instance representing the deployed contract after ensuring that code exists at that location and that that address exists on the network being used.

#### `MyContract.link(instance)`

Link a library represented by a contract abstraction instance to MyContract. The library must first be deployed and have its deployed address set. The name and deployed address will be inferred from the contract abstraction instance. When this form of `MyContract.link()` is used, MyContract will consume all of the linked library's events and will be able to report that those events occurred during the result of a transaction.

Libraries can be linked multiple times and will overwrite their previous linkage.

Note: This method has two other forms, but this form is recommended.

#### `MyContract.link(name, address)`

Link a library with a specific name and address to MyContract. The library's events will not be consumed using this form.

#### `MyContract.link(object)`

Link multiple libraries denoted by an Object to MyContract. The keys must be strings representing the library names and the values must be strings representing the addresses. Like above, libraries' events will not be consumed using this form.

#### `MyContract.networks()`

View a list of network ids this contract abstraction has been set up to represent.

#### `MyContract.setProvider(provider)`

Sets the web3 provider this contract abstraction will use to make transactions.

#### `MyContract.setNetwork(network_id)`

Sets the network that MyContract is currently representing.

#### `MyContract.hasNetwork(network_id)`

Returns a boolean denoting whether or not this contract abstraction is set up to represent a specific network.

#### `MyContract.defaults([new_defaults])`

Get's and optionally sets transaction defaults for all instances created from this abstraction. If called without any parameters it will simply return an Object representing current defaults. If an Object is passed, this will set new defaults. Example default transaction values that can be set are:

```
MyContract.defaults({
  from: ...,
  gas: ...,
  gasPrice: ...,
  value: ...
})
```

Setting a default `from` address is useful when you have a contract abstraction you intend to represent one user (i.e., one address).

### Contract Abstraction Instance API

Each contract abstraction is different, though each abstraction's behavior has many similarites. Let's explore those similarities using the following contract:

```javascript
contract MyContract {
  uint public value;
  event ValueSet(uint val);
  function setValue(uint val) {
    value = val;
    ValueSet(value);
  }
  function getValue() constant returns (uint) {
    return value;
  }
}
```

#### Instance Functions & Behavior

* From Javascript's point of view, this contract has three functions: `setValue`, `getValue` and `value`. This is because `value` is public and automatically creates a getter function for it.
* When we call `setValue()`, this creates a transaction. From Javascript:

  ```javascript
  instance.setValue(5).then(function(result) {
    // result object contains import information about the transaction
  });
  ```

* We can call `setValue()` without creating a transaction by explicitly using `.call`:

  ```javascript
  instance.setValue.call(5).then(...);
  ```

  This isn't very useful in this case, since `setValue()` sets things, and the value we pass won't be saved since we're not creating a transaction.

* However, we can *get* the value using `getValue()`, using `.call()`. Calls are always free and don't cost any Ether, so they're good for calling functions that read data off the blockchain:

  ```javascript
  instance.getValue.call().then(function(val) {
    // val reprsents the `value` storage object in the solidity contract
    // since the contract returns that value.
  });
  ```

* Even more helpful, however is we *don't even need* to use `.call` when a function is marked as `constant`, because Pudding will automatically know that that function can only be interacted with via a call:

  ```javascript
  instance.getValue().then(function(val) {
    // val reprsents the `value` storage object in the solidity contract
    // since the contract returns that value.
  });
  ```

* When you make a transaction, you're given a `result` object that gives you a wealth of information about the transaction. You're given the transaction has (`result.tx`), the decoded events (also known as logs; `result.logs`), and a transaction receipt (`result.receipt`). In the below example, you'll recieve the `ValueSet()` event because you triggered the event using the `setValue()` function:

  ```javascript
  instance.setValue(5).then(function(result) {
    // result.tx => transaction hash, string
    // result.logs => array of trigger events (1 item in this case)
    // result.receipt => receipt object
  });
  ```



### Pudding API

#### `Pudding.save(options, filename[, extra_options])`

Save contract data as a `.sol.js` file. Returns a Promise.

* `options`: Object. Data that represents this contract:

    ```javascript
    {
      contract_name: "MyContract",  // String; optional. Defaults to "Contract"
      abi: ...,                     // Array; required.  Application binary interface.
      unlinked_binary: "...",       // String; optional. Binary without resolve library links.
      address: "...",               // String; optional. Deployed address of contract.
      network_id: "...",            // String; optional. ID of network being saved within abstraction.
      default_network: "..."        // String; optional. ID of default network this abstraction should use.
    }
    ```

    Note: `save()` will also accept an already `require`'d contract object. i.e.,

    ```javascript
    var MyContract = require("./path/to/MyContract.sol.js");

    Pudding.save(MyContract, ...).then(...);
    ```

		In this case, you can use the `extra_options` parameter to specify options that aren't managed by the contract abstraction itself.

* `filename`: Path to save contract file.
* `extra_options`: Object. Used if you need to specify other options within a separate object, for instance, when a contract abstraction is passed instead of an `options` object.

#### `Pudding.saveAll(contracts, directory, options)`

Save many contracts to the filesystem at once. Returns a Promise.

* `contracts`: Object. Keys are the contract names and the values are `contract_data` objects, as in the `save()` function above:

    ```javascript
    {
      "MyContract": {
        "abi": ...,
        "unlinked_binary": ...
      }
      "AnotherContract": {
        // ...
      }
    }
    ```

* `directory`: String. Destination directory. Files will be saved via `<contract_name>.sol.js` within that directory.
* `options`: Object. Same options listed in `save()` above.

#### `Pudding.generate(options, networks)`

Generate the source code that populates the `.sol.js` file. Returns a String.

* `options`: Object. Subset of options listed in the `save()` function above. Expects:

    ```javascript
		{
			abi: ...,
			unlinked_binary: ...
		}
		```


* `networks`: Object. Contains the information about this contract for each network, keyed by the network id.

    ```javascript
    {
      "1": {        // live network
        "address": ...
      },
      "2": {        // morden network
        "address": ...
      },
      "1337": {     // private network
        "address": ...
      }
    }
    ```

#### `Pudding.whisk(options, networks)`

Like `generate()`, this function will create the source code that populates the `.sol.js` file, but instead of returning it as a string it will import it and return an object ready for use. Parameters are the same as `generate()`.

### Running Tests

```
$ npm test
```

### License

MIT
