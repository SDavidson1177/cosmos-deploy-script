# Cosmos Deploy Script

## Purpose

This script can be used to configure and launch cosmos blockchains. You can choose how many validators the blockchain has and connected blockchains over IBC. Currently, you can launch "gaia" chains and "evmos" chains. Gaia is the same type of blockchain as Cosmos Hub. Evmos is a cosmos chain that also has the evm.

## Disclaimer

Currently, you **must** evmos v12.0.x if you would like to deploy evmos chains. The most recent version of gaia is compatible. To use other cosmos chains, like ones created using Ignite CLI, you will need to modify the script.

## Background

All cosmos blockchains are build in a similar way. There is a Makefile in the root directory of cosmos chains that allow you to build an executable for that chain. This script assumes that the appropriate executable has already been built. This means that you can modify the blockchain source code as much as you like. As long as you recompile the blockchain to an executable, this script can be used to deploy that blockchain.

## Deploying a Blockchain

### 1. Download the blockchain source code

The first step is to download the source code for the blockchain you would like to deploy. For instance, this is done for gaia using the following command:

```bash
git clone https://github.com/cosmos/gaia.git
```

From now on, all operations will be preformed from the root directory of this git repository.

### 2. Compile the executable

Next, you must compile the executable for that blockchain. How you do this depends on the blockchain. Typically, you may do so by running the following commands

```bash
make clean && make install
```

from the root directory. Once the executable has been build, you must move it into the chain's build directory. You can typically do so by running the following command from the root directory.

```bash
cp $(which <executable name>) ./build/
```

For gaia, this would be...

```bash
cp $(which gaiad) ./build/
```

### 3. Import the script

Copy *deploy.sh*, *DockerfileChain* and *DockerfileRelayer* into the root directory. Edit *DockerfileChain* to copy the intended executable. The example *DockerfileChain* copies over the executable for evmos.

### 4. Set Docker Environment Variable

Export the docker username environment variable

```bash
export DOCKER_USERNAME=<docker username>
```

### 5. Create Overlay Network

The overlay network is used for interblockchain communication. The standard network is used for internode communication for the same blockchain. Enter the command:

```bash
./deploy.sh create-overlay
```

### 6. Enter the start command

Run the following command:

```bash
./deploy.sh start <chain type> <chain name> <num validators> <id> --build --network
```

Some important notes:

- The chain type can be either "gaia" or "evmos".
- Chain name can be any string.
- Number of validators must be between 1 and 255 inclusive.
- The chain id is an integer of at most 255. There may be further restrictions depending on the blockchain. For example, you should use number in [5, 255] for evmos, as numbers 1-4 are reserved.
- The build flag should only be used once per new blockchain build.
- The network flag sets up a network for the docker containers. This needs to be done only once.

Example:

```bash
./deploy.sh start evmos earth 4 5
```

The above commands starts an evmos blockchain with 4 validators and with chain id 5 (translated internally as evmos_9000-5).

### 7. Inspect Containers

You can list the running containers using "docker container list". You can check a container's logs using the docker logs command. Furthermore, you can enter a container to perform additional commands. Each container has the executable for the blockchain stored in the working directory. The home directory for the blockchain is in the home directory, and is called either "evmos" or "gaia".

## Connecting Blockchains over IBC

To connect blockchains over IBC, you must have multiple blockchains running. To create a relayer for both blockchains, enter the following command:

```bash
./deploy.sh relayer <chain type> <chain A id> <chain B id> <chain A node> <chain B node> --swarm
```
Some important notes:

- The chain type can be either "gaia" or "evmos".
- Chain ids are the integers assigned to the blockchains upon creation (see section on deploying blockchains).
- The node is a value from [0, number of validators - 1]. By default, only nodes "0" are connected to the overlay network. Should you wish to use different nodes, you must first connect those nodes to the overlay network. This is done using the command **docker network connect baton-overlay "container name"**.
- The swarm command connects the relayer to the overlay network.

The relayer container is configured with the Hermes relayer. Once the container is running, you must enter into that container's shell. From there, you can use Hermes commands.

Use the command

```bash
hermes --config config.toml create channel --a-chain <chain a> --b-chain <chain b> --a-port transfer --b-port transfer --new-client-connection
```

to create a connection between two chains. An example for evmos is as follows:

```bash
hermes --config config.toml create channel --a-chain evmos_9000-5 --b-chain evmos_9000-6 --a-port transfer --b-port transfer --new-client-connection
```

Once the connection has been created, start the relayer using the following command:

```bash
hermes --config config.toml start
```

## Stopping The Containers

To stop a blockchain, run the command:

```bash
./deploy.sh stop <chain name>
```

To stop a relayer, run the command:

```bash
./deploy.sh relayer-stop <chain A/B id> <chain B/A id>
```

## Blockchains on Different Servers

The overlay network allows you to connect blockchains running on different servers. It is important that the blockchain repository is shared on the servers (so that they can access the same build/ directory). Furthermore, you must have the servers all join the same overlay network. For more information on the overlay network driver, please see https://docs.docker.com/network/drivers/overlay/
