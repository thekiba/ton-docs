# Storage daemon

*A storage daemon is a program used to download and share files in the TON network. The storage daemon console program, `storage-daemon-cli`, is used to manage a running storage daemon.*

The current version of the storage daemon can be found in the [tonstorage](https://github.com/SpyCheese/ton/tree/master/storage) branch.


## Installation 

Before installing all the necessary components, it is worth reading [this tutorial on installing](https://ton.org/docs/develop/smart-contracts/environment/installation) the environment.

### Linux (Ubuntu / Debian):

* Download the latest version of the TON Blockchain source code available at the [GitHub repository](https://github.com/ton-blockchain/ton/)

```
git clone --recurse-submodules https://github.com/ton-blockchain/ton.git
```

* Install the latest versions of:
     * `make`
     * `cmake` version 3.0.2+
     * C++ Compiler: `g++` or `clang` 
     * OpenSSL

* Run this in the linux terminal:

```bash
sudo apt update
sudo apt install git make cmake g++ libssl-dev zlib1g-dev wget
cd ~ && git clone https://github.com/ton-blockchain/ton.git
cd ~/ton && git submodule update --init
mkdir ~/ton/build && cd ~/ton/build && cmake .. -DCMAKE_BUILD_TYPE=Release && make -j 4
```

### Run project


To run, run this and you're done:
```
cmake --build . --target storage-daemon storage-daemon-cli
```


## Key concepts
* *Bag of files* or *Bag* - a collection of files distributed through TON Storage
* TON Storage's network part is based on technology similar to torrents, so the terms *Torrent*, *Bag of files*, and *Bag* will be used interchangeably. It's important to note some differences, however: TON Storage transfers data over ADNL, each *Bag* is distributed through its own overlay, the merkle structure can exist in two versions - with large chunks for efficient downloading and small ones for efficient ownership proof, and TON's DHT network is used for finding peers.
* A *Bag of files* consists of *torrent info* and a data block.
* The data block starts with a *torrent header* - a structure that contains a list of files with their names and sizes. The files themselves follow in the data block.
* The data block is divided into chunks (128 KB by default), and a *merkle tree* (made of TVM cells) is built on the SHA256 hashes of these chunks. This allows building and verifying *merkle proofs* of individual chunks, as well as efficiently updating the *Bag* by exchanging only the proof of the modified chunk.
* *Torrent info* contains the *merkle root* of the 
    * Chunk size (data block)
    * the list of chunks' sizes
    * Hash *merkle tree*
    * Description - any text specified by the creator of the torrent
* *Torrent info* is serialized to a TVM cell. The hash of this cell is called *BagID*, and it uniquely identifies *Bag*.
* *Bag meta* is a file containing *torrent info* and *torrent header*.* This is an analog `.torrent` files.


## Starting the storage daemon and storage-daemon-cli

### An example command for starting the storage-daemon:


```storage-daemon -v 3 -C global.config.json -I <ip>:3333 -p 5555 -D storage-db```

* `-v` - verbosity level (INFO)
* `-C` - global network config
* `-I` - IP address and port for adnl
* `-p` - tcp port for console interface
* `-D` - directory for the storage daemon database

### storage-daemon-cli management
It's started like this:

```
storage-daemon-cli -I 127.0.0.1:5555 -k storage-db/cli-keys/client -p storage-db/cli-keys/server.pub
```

* `-I` - this is the IP address and port of the daemon (the port is the same one specified in the `-p` parameter above)
* `-k` and `-p` - these are the client's private key and the server's public key (similar to `validator-engine-console`). These keys are generated on the first run of the daemon and are placed in `<db>/cli-keys/`.

### List of commands

The list of `storage-daemon-cli` commands can be obtained with the `help` command.
Parameters with spaces should be enclosed in quotes.

`storage-daemon-cli` can be run in non-interactive mode by passing it commands to execute:

```
storage-daemon-cli ... -c "add-by-meta m" -c "list --hashes"
```

## Adding a Bag of Files
To download a *Bag of Files*, you need to know its `BagID` or have a meta-file. The following commands can be used to add a *Bag* for download:
```
add-by-hash <hash> -d directory
add-by-meta <meta-file> -d directory
```
The *Bag* will be downloaded to the specified directory. You can omit it, then it will be saved to the storage daemon directory.

:::info
The hash is specified in hexadecimal form (length - 64 characters).
:::

When adding a *Bag* by a meta-file, information about the *Bag* will be immediately available: size, description, list of files. When adding by hash, you will have to wait for this information to be loaded.


## Managing Added Bags
* The `list` command outputs a list of *Bags*. 
* `list --hashes` outputs a list with full hashes. In all subsequent commands, 
* `<BagID>` is either a hash (hexadecimal) or an ordinal number of the *Bag* within the session (a number that can be seen in the list by the `list` command). Ordinal numbers of *Bags* are not saved between restarts of storage-daemon-cli and are not available in non-interactive mode.

### Methods

* `get <BagID>` - outputs detailed information about the *Bag*: description, size, download speed, list of files.
* `get-peers <BagID>` - outputs a list of peers.
* `download-pause <BagID>`, `download-resume <BagID>` - pauses or resumes downloading.
* `remove <BagID>` - removes the *Bag*. `remove --remove-files` also deletes all files of the *Bag*. Note that if the *Bag* is saved in the internal storage daemon directory, the files will be deleted in any case.


## Partial Download, Priorities
:::info
When adding a *Bag* you can specify which files you want to download from it:
:::
```
add-by-hash <hash> -d dir --partial file1 file2 file3
add-by-meta <meta-file> -d dir --partial file1 file2 file3
```

### Priorities

Each file in the *Bag of Files* has a priority, a number from 0 to 255. Priority 0 means the file won't be downloaded. The `--partial` flag sets the specified files to priority 1, the others to 0.

You can change the priorities of an already added *Bag* with the following commands:
* `priority-all <BagID> <priority>` - for all files.
* `priority-idx <BagID> <idx> <priority>` - for one file by number (see with `get` command).
* `priority-name <BagID> <name> <priority>` - for one file by name.
Priorities can be set even before the list of files is downloaded.


## Creating a Bag of Files
To create a *Bag* and start distributing it, use the `create` command:
```
create <path>
```
`<path>` can point to either a single file or a directory. When creating a *Bag*, you can specify a description:
```
create <path> -d "Bag of Files description"
```
After the *Bag* is created, the console will display detailed information about it (including the hash, which is the `BagID` by which the *Bag* will be identified), and the daemon will start distributing the torrent.

To download the *Bag*, other users just need to know its hash. You can also save the torrent meta file:
```
get-meta <BagID> <meta-file>
```