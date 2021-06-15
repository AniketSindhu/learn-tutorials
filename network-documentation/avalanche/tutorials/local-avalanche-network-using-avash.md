---
description: Learn how to setup and run a local Avalanche network using Avash
---

# Setting up a local Avalanche network using Avash

## Introduction

[Avash](https://github.com/ava-labs/avash) is a temporary stateful shell client which can be used for various purposes like deploying local Avalanche networks, managing their processes, and for running network tests. However, once we exit from Avash, this local copy of Avalanche stops running and all its history and transactions are lost. This provides us with the opportunity to experiment with the properties and behaviour of Avalanche networks on our local system without needing to connect to the main Avalanche network.   
  
Avash also provides us with the ability to write scripts using Lua, to automate the creation of such networks and their configuration. [Lua](http://www.lua.org/) is highly portable scripting language which can be embedded into other applications, this is especially useful because of it's fast language engine with a small footprint. Lua is being used as the scripting language within Avash. In this tutorial, we're going to install a copy of Avash on our machine and create a Lua script that can be used to fire up a 5 node staking network \(for testing purposes\).

## Requirements

For the smooth completion of this tutorial, we need the following software to be already present on your system:

* [Golang](https://golang.org/) \(1.15.5+\)

## AvalancheGo Installation

[AvalancheGo](https://github.com/ava-labs/avalanchego) is the official node implementation of the Avalanche network. Avash depends on an executable of this node implementation being present. So, we must install AvalancheGo before we attempt to run Avash.

To begin, look at the actual version of Golang which has been installed with the terminal command :

```bash
go version
```

### For go versions &lt; 1.16:

If the version number reported by `go version` is _less_ than v1.16 :

{% code title="Enter this command in a terminal" %}
```bash
go get -v -d github.com/ava-labs/avalanchego/...
```
{% endcode %}

Since go 1.16, the module-aware mode is enabled by default, and this along with many other things, means that when we execute `go get ...`, the project gets downloaded to `$GOPATH/pkg/mod` and the permissions on this directory are set such that we won't be able to execute `scripts/build.sh` for building AvalancheGo and so we must turn this mode off for our installation of AvalancheGo.  
Hopefully this incompatibility between versions will be resolved in the future, but for now, we've got to take care of this ourselves.

### For go versions &gt;= 1.16:

If the version number reported by `go version` is _greater than_ **or** _equal to_ v1.16 :

{% code title="Enter this command in a terminal" %}
```bash
GO111MODULE=off go get -v -d github.com/ava-labs/avalanchego/...
```
{% endcode %}

{% hint style="info" %}
Make sure that the environment variable GOPATH is already set. Usually, it is located at ~/go.
{% endhint %}

Now we change to the directory in which the project was downloaded and build it:

{% code title="Enter these commands in a terminal" %}
```bash
cd $GOPATH/src/github.com/ava-labs/avalanchego
./scripts/build.sh
```
{% endcode %}

{% hint style="info" %}
If the build process fails, please make sure that the version of Golang installed on your machine is &gt; 1.15.5.
{% endhint %}

After the build process is complete, you can find the AvalancheGo binary, named avalanchego, inside the build directory.

## Avash Installation

Now we go onto install Avash. Unlike AvalancheGo, Avash needs the module-aware mode enabled for it to be successfully installed.

```bash
go get github.com/ava-labs/avash
```

If this command fails to execute with similar errors like below, that means we've to turn on the module-aware mode explicitly.

```text
cannot find package "github.com/decred/dcrd/dcrec/secp256k1/v3" in any of:
/snap/go/7416/src/github.com/decred/dcrd/dcrec/secp256k1/v3 (from $GOROOT)
~/go/src/github.com/decred/dcrd/dcrec/secp256k1/v3 (from $GOPATH)
cannot find package "github.com/decred/dcrd/dcrec/secp256k1/v3/ecdsa" in any of:
/snap/go/7416/src/github.com/decred/dcrd/dcrec/secp256k1/v3/ecdsa (from $GOROOT)
~/go/src/github.com/decred/dcrd/dcrec/secp256k1/v3/ecdsa (from $GOPATH)
cannot find package "github.com/hashicorp/hcl/hcl/printer" in any of:
/snap/go/7416/src/github.com/hashicorp/hcl/hcl/printer (from $GOROOT)
~/go/src/github.com/hashicorp/hcl/hcl/printer (from $GOPATH)
# cd .; git clone -- https://github.com/chzyer/readline /home/kevin/go/src/github.com/chzyer/readline
Cloning into '~/go/src/github.com/chzyer/readline'...
fatal: unable to access 'https://github.com/chzyer/readline/': gnutls_handshake() failed: Error in the pull function.
package github.com/chzyer/readline: exit status 128
```

To continue, we use the same command as before but we pass in an extra environment variable to explicitly turn on module-aware mode:

```bash
GO111MODULE=on go get github.com/ava-labs/avash
```

Now we have the code for Avash downloaded onto our machines at this point. Again we have some differences in behavior based on golang versions being used.

### For go versions &gt;= 1.16:

In this case, the Avash code gets automatically built after being downloaded and you can find the binary available at $GOPATH/bin directory. To verify this, try:

```bash
cd $GOPATH/bin
./avash
```

You must be greeted with the Avash console. Feel free to skip to the end of this section where the Avash console is shown.

### For go versions &lt; 1.16:

However, for those who're using go versions &lt; 1.16, you have to manually build the Avash source:

```bash
cd $GOPATH/src/github.com/ava-labs/avash
go build
```

After the the project is built successfully, the Avash binary should be present in the same directory. To verify this, try:

```bash
cd $GOPATH/bin
./avash
```

You must be greeted with the Avash console:

```text
avash>
```

And you've successfully installed and run Avash on your machine! To exit from Avash, type:

```bash
exit
```

## Adding Lua scripts

Now that we have a successful installation of Avash on our machine, we go ahead and add a Lua script that we'll use to fire up the Avalanche network.

### For go versions &lt; 1.16:

We need to add a configuration file and a Lua script to the `scripts` directory inside the Avash installation.

```text
cd $GOPATH/src/github.com/ava-labs/avash
```

The configuration below will be used inside the Lua script to configure the nodes that we start up from within the script. The main difference in configuration between this node and the official five\_node\_staking.lua script is that for the nodes we fire up, we enable personal\_namespace in coreth which is currently disabled by default. This will be useful later on in other tutorials regarding smart-contracts using truffle, hardhat, waffle, etc.

_**scripts/config/staking\_node\_config.json**_

```text
{
  "db-enabled": false,
  "staking-enabled": true,
  "log-level": "debug",
  "coreth-config": {
    "snowman-api-enabled": false,
    "coreth-admin-api-enabled": false,
    "net-api-enabled": true,
    "rpc-gas-cap": 2500000000,
    "rpc-tx-fee-cap": 100,
    "eth-api-enabled": true,
    "tx-pool-api-enabled": true,
    "debug-api-enabled": true,
    "web3-api-enabled": true,
    "personal-api-enabled": true
  }
}
```

Next comes our lua script itself:

_**scripts/five\_node\_staking\_with\_config.lua**_

```javascript
cmds = {
"startnode node1 --config-file=scripts/config/staking_node_config.json --http-port=9650 --staking-port=9651 --bootstrap-ips= --staking-tls-cert-file=certs/keys1/staker.crt --staking-tls-key-file=certs/keys1/staker.key",
"startnode node2 --config-file=scripts/config/staking_node_config.json --http-port=9652 --staking-port=9653 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=certs/keys2/staker.crt --staking-tls-key-file=certs/keys2/staker.key",
"startnode node3 --config-file=scripts/config/staking_node_config.json --http-port=9654 --staking-port=9655 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=certs/keys3/staker.crt --staking-tls-key-file=certs/keys3/staker.key",
"startnode node4 --config-file=scripts/config/staking_node_config.json --http-port=9656 --staking-port=9657 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=certs/keys4/staker.crt --staking-tls-key-file=certs/keys4/staker.key",
"startnode node5 --config-file=scripts/config/staking_node_config.json --http-port=9658 --staking-port=9659 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=certs/keys5/staker.crt --staking-tls-key-file=certs/keys5/staker.key",
}

for key, cmd in ipairs(cmds) do
avash_call(cmd)
end
```

### For go versions &gt;= 1.16:

We need to add a configuration file and a Lua script to the `avash_scripts` directory inside of the home directory of the current user.

```text
mkdir ~/avash_scripts
cd ~/avash_scripts
```

The configuration below will be used inside the Lua script to configure the nodes that we start up from within the script. The main difference in configuration between this node and the official five\_node\_staking.lua script is that for the nodes we fire up, we enable personal\_namespace in coreth which is currently disabled by default. This will be useful later on in other tutorials regarding smart-contracts using truffle, hardhat, waffle, etc.

_**config/staking\_node\_config.json**_

```text
{
  "db-enabled": false,
  "staking-enabled": true,
  "log-level": "debug",
  "coreth-config": {
    "snowman-api-enabled": false,
    "coreth-admin-api-enabled": false,
    "net-api-enabled": true,
    "rpc-gas-cap": 2500000000,
    "rpc-tx-fee-cap": 100,
    "eth-api-enabled": true,
    "tx-pool-api-enabled": true,
    "debug-api-enabled": true,
    "web3-api-enabled": true,
    "personal-api-enabled": true
  }
}
```

Next comes our lua script itself:

_**five\_node\_staking\_with\_config.lua**_

{% hint style="info" %}
You have to replace all occurrences of **avash@v1.1.4** in the configuration file below with the actual version of Avash that is installed on your machine - **avash@v{your\_version}**. Search and Replace functionality \(Ctrl+H in most programs\) is a good choice for this task.
{% endhint %}

```javascript
cmds = {
"startnode node1 --config-file=../../avash_scripts/config/staking_node_config.json --http-port=9650 --staking-port=9651 --bootstrap-ips= --staking-tls-cert-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys1/staker.crt --staking-tls-key-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys1/staker.key",
"startnode node2 --config-file=../../avash_scripts/config/staking_node_config.json --http-port=9652 --staking-port=9653 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys2/staker.crt --staking-tls-key-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys2/staker.key",
"startnode node3 --config-file=../../avash_scripts/config/staking_node_config.json --http-port=9654 --staking-port=9655 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys3/staker.crt --staking-tls-key-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys3/staker.key",
"startnode node4 --config-file=../../avash_scripts/config/staking_node_config.json --http-port=9656 --staking-port=9657 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys4/staker.crt --staking-tls-key-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys4/staker.key",
"startnode node5 --config-file=../../avash_scripts/config/staking_node_config.json --http-port=9658 --staking-port=9659 --bootstrap-ips=127.0.0.1:9651 --bootstrap-ids=NodeID-7Xhw2mDxuDS44j42TCB6U5579esbSt3Lg --staking-tls-cert-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys5/staker.crt --staking-tls-key-file=../pkg/mod/github.com/ava-labs/avash@v1.1.4/certs/keys5/staker.key",
}

for key, cmd in ipairs(cmds) do
avash_call(cmd)
end
```

## Setup a local Avalanche network using Avash

In the last section, we've added the Lua script in the appropriate location, which we could now use to fire up from within Avash.

### For go versions &lt; 1.16:

To start a local five node Avalanche network, follow these steps:

```text
cd $GOPATH/src/github.com/ava-labs/avash
$ ./avash
```

You must be looking at the Avash prompt as shown below:

```text
avash>
```

Now you need to run the script we created in the last section which will start a local five-node staking network on your machine.

```text
runscript scripts/five_node_staking_with_config.lua
```

The nodes must have now started successfully, and the console should look similar to what you see below:

```text
avash> runscript scripts/five_node_staking_with_config.lua
RunScript: Running scripts/five_node_staking_with_config.lua
RunScript: Successfully ran scripts/five_node_staking_with_config.lua
```

### For go versions &gt;= 1.16:

To start a local five-node Avalanche network, follow these steps:

```text
cd $GOPATH/bin
./avash
```

You must be looking at the Avash prompt as shown below:

```text
avash>
```

Now you need to run the script we created in the last section which will start a local five-node staking network on your machine.

```text
runscript ../../avash_scripts/five_node_staking_with_config.lua
```

The nodes must have now started successfully, and the console should look similar to what you see below:

```text
avash> runscript ../../avash_scripts/five_node_staking_with_config.lua
RunScript: Running ../../avash_scripts/five_node_staking_with_config.lua
RunScript: Successfully ran ../../avash_scripts/five_node_staking_with_config.lua
```

## Interacting with the local Avalanche network

To interact with the running Avalanche network, open up a new terminal and type in the following command:

```bash
curl --location --request POST 'http://localhost:9650/ext/info' \
--header 'Content-Type: application/json' \
--data-raw '{"jsonrpc":"2.0","id":1, "method" :"info.getBlockchainID", "params": {"alias": "X"}}'
```

This should return a response similar to what you can see below:

{% hint style="info" %}
Remember, do not close the Avash terminal window until you are done working with the local nodes. When the terminal is closed, the local Avalanche network gets destroyed along with it.
{% endhint %}

```text
{"jsonrpc":"2.0","result":{"blockchainID":"2eNy1mUFdmaxXNj1eQHUe7Np4gju9sJsEtWQ4MX3ToiNKuADed"},"id":1}
```

When you're all done experimenting with the local Avalanche network we just fired up, to exit from Avash, type the following command in the Avash prompt:

```bash
exit
```

This closes the Avash terminal and with it, all the nodes started during its lifetime, essentially destroying the temporary local Avalanche network we fired up using the Lua script.

## Conclusion

In this tutorial, we've successfully managed to install Avash, create a Lua script that fires up a five-node staking network on your machine, and fire it up and interact with the network from the terminal.

Congratulations on making it to the end of this tutorial!

> “No great thing is created suddenly, any more than a bunch of grapes or a fig. If you tell me that you desire a fig, I answer that there must be time. Let it first blossom, then bear fruit, then ripen.”
>
> -- Epictetus

So, keep learning and keep building and I'm sure you're on your way to building something great! Good luck!

If you had any difficulties following this tutorial or simply want to discuss Avalanche tech with us you can join [**our community**](https://discord.gg/fszyM7K) today!

## References

* [Avalanchego Readme](https://github.com/ava-labs/avalanchego/blob/master/README.md)
* [Avash Documentation](https://docs.avax.network/build/tools/avash)
