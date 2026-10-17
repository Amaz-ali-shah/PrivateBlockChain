# My First Private Ethereum Blockchain (Genesis Block)

A private Ethereum network built with **Geth 1.10.26** and **Puppeth**, using **Ethash (Proof of Work)** consensus. No account is pre-funded: every ether on this chain was created by mining.

| Item | Value |
|---|---|
| Network name | `myprivateeth` |
| Chain / Network ID | `2026` |
| Consensus | Ethash (Proof of Work) |
| Geth version | 1.10.26-stable |
| Pre-funded accounts | None |
| Mining reward | 2 ether per block |
| Data directory | `node1` |

---

## 1. Concepts

- **Genesis block**: the first block (block 0) of a blockchain. It is defined in a JSON file and fixes the chain's rules: chain ID, consensus engine, initial difficulty, gas limit and starting balances.
- **Geth**: the Go implementation of Ethereum. It runs the node, mines blocks and provides the console.
- **Puppeth**: a wizard bundled with Geth that generates the genesis file. It was removed in Geth 1.11+, so this project uses 1.10.26.
- **Ethash**: the Proof of Work algorithm. Miners search for a valid nonce. It needs a ~1 GB **DAG** file, generated on first mining (a few minutes) and reused afterwards.
- **Network ID vs Chain ID**: both are `2026` here. The network ID separates peers at the networking level; the chain ID is used when signing transactions.
- **Keystore**: the folder `node1/keystore/` holds each account's private key, encrypted with its password.
- **Mining reward**: the miner's account (etherbase) receives 2 ether for each block it seals. This is how ether is created.

---

## 2. Prerequisites

- Ubuntu (VirtualBox VM is fine), at least 4 GB RAM and 2 CPU cores recommended
- Geth and Puppeth 1.10.26, installed globally

### 2.1 Remove a newer Geth (if installed via apt)

```bash
sudo apt remove --purge geth abigen evm rlpdump -y
hash -r
which -a geth
```

- `apt remove --purge` uninstalls the packages and their config files.
- `hash -r` clears the shell's cache of command locations.
- `which -a geth` lists every `geth` on the PATH. It should print nothing now.

### 2.2 Install Geth 1.10.26 globally

Download `geth-alltools-linux-amd64-1.10.26-e5eb32ac.tar.gz` from the official Geth releases, extract it, then:

```bash
cd ~/geth-alltools-linux-amd64-1.10.26-e5eb32ac
sudo cp geth puppeth bootnode clef abigen evm rlpdump /usr/local/bin/
sudo chmod +x /usr/local/bin/geth /usr/local/bin/puppeth
```

- Copies the binaries to `/usr/local/bin`, which is on the PATH, so they run from any folder.
- `chmod +x` makes them executable.

Verify:

```bash
cd ~
hash -r
geth version
which puppeth
```

Expected: `Version: 1.10.26-stable` and `/usr/local/bin/puppeth`.

---

## 3. Create the project folder and accounts

```bash
cd ~
mkdir myPrivateEth && cd myPrivateEth
geth --datadir node1 account new
geth --datadir node1 account new
```

- `mkdir` and `cd` create and enter the project folder.
- `--datadir node1` sets where the node stores its data (chain database and keystore).
- `account new` generates a new key pair, asks for a password, and prints the public address. Run it twice to get two accounts.

**Remember the passwords and copy both addresses.** The password is needed later to unlock the account for sending ether.

---

## 4. Create the genesis file with Puppeth

```bash
puppeth
```

Answer the prompts as follows:

| # | Prompt | Answer | Explanation |
|---|---|---|---|
| 1 | Network name | `myprivateeth` | Lowercase only, no spaces or hyphens. Used for the output file name. |
| 2 | What would you like to do? | `2` | Configure new genesis. |
| 3 | Create or import | `1` | Create new genesis from scratch. |
| 4 | Consensus engine | `1` | Ethash (Proof of Work). |
| 5 | Pre-funded accounts | *(press Enter on an empty line)* | Nobody starts with ether. All ether comes from mining. |
| 6 | Pre-fund precompiles | `no` | Precompiles are built-in contracts at addresses 0x1 to 0xff. Funding them with 1 wei is optional compatibility. |
| 7 | Chain / network ID | `2026` | Unique ID for this network. |
| 8 | What would you like to do? | `2` | Manage existing genesis. |
| 9 | Option | `2` | Export genesis configurations. |
| 10 | Folder to save to | *(press Enter)* | Saves `myprivateeth.json` in the current folder. |

Press `Ctrl+C` to exit Puppeth, then check:

```bash
ls
```

You should see `myprivateeth.json` and the `node1` folder. `myprivateeth.json` is the **genesis block definition**.

---

## 5. Initialize the node with the genesis block

```bash
geth --datadir node1 init myprivateeth.json
```

- `init` reads the genesis file and writes block 0 into the node's database.
- Success looks like: `Successfully wrote genesis state`.
- Run `init` **only once**. Running it again on a different genesis requires deleting the chain data first.

---

## 6. Start the node (Terminal 1)

```bash
geth --datadir node1 --networkid 2026 --nodiscover --http --http.api eth,net,web3,personal,miner,admin --allow-insecure-unlock
```

| Flag | Meaning |
|---|---|
| `--datadir node1` | Use the data folder created earlier. |
| `--networkid 2026` | Must match the chain ID in the genesis file. |
| `--nodiscover` | Do not search for peers. Keeps the network private. |
| `--http` | Enable the HTTP JSON-RPC server on `127.0.0.1:8545`. |
| `--http.api ...` | Which API modules are exposed over HTTP. |
| `--allow-insecure-unlock` | Allows unlocking accounts over HTTP. Use only on test networks. |

Leave this terminal running. It prints the logs (blocks sealed, DAG progress, etc.).

---

## 7. Attach to the node (Terminal 2)

Open a new terminal:

```bash
cd ~/myPrivateEth
geth attach node1/geth.ipc
```

- `attach` opens the interactive JavaScript console connected to the running node.
- `node1/geth.ipc` is the IPC socket file the node created. IPC is local and does not need `--http`.
- You get a `>` prompt. All commands below are typed there.

---

## 8. Console commands

### 8.1 Inspect accounts and balances

```js
eth.accounts
web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
web3.fromWei(eth.getBalance(eth.accounts[1]), "ether")
```

| Command | Description |
|---|---|
| `eth.accounts` | Lists all accounts in the keystore. |
| `eth.getBalance(addr)` | Returns the balance in **wei** (1 ether = 10^18 wei). |
| `web3.fromWei(x, "ether")` | Converts wei to ether for readability. |

Both balances should be `0` because nothing is pre-funded.

### 8.2 Start mining (creates ether)

```js
miner.setEtherbase(eth.accounts[0])
miner.start(1)
```

| Command | Description |
|---|---|
| `miner.setEtherbase(addr)` | Sets the account that receives mining rewards. |
| `miner.start(1)` | Starts mining with 1 CPU thread. Use more threads if you have more cores. |

The first start builds the DAG (about 4 minutes on a VM). After that, blocks are sealed quickly. In Terminal 1 you will see `Successfully sealed new block` and `🔨 mined potential block`.

Check progress:

```js
eth.blockNumber
web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
eth.mining
```

- `eth.blockNumber` returns the latest block number.
- `eth.mining` returns `true` while mining.
- The balance of account 0 grows by 2 ether per mined block.

### 8.3 Stop mining

```js
miner.stop()
eth.mining
```

`eth.mining` should now print `false`.

### 8.4 Unlock an account

```js
personal.unlockAccount(eth.accounts[0], "YOUR_PASSWORD", 300)
```

- Sending a transaction requires a digital signature from the sender's private key.
- The key is stored encrypted in the keystore. Unlocking decrypts it into memory so the node can sign.
- `YOUR_PASSWORD` is the password chosen at `account new`.
- `300` is the unlock duration in seconds (`0` means until the node stops).
- Returns `true` on success.

### 8.5 Transfer ether

```js
eth.sendTransaction({from: eth.accounts[0], to: eth.accounts[1], value: web3.toWei(5, "ether")})
```

- `from`: sender (must be unlocked)
- `to`: receiver (no password needed to receive)
- `value`: amount in wei. `web3.toWei(5, "ether")` converts 5 ether to wei.
- Returns a **transaction hash**.

A transaction stays **pending** until a miner includes it in a block, so mining must be running:

```js
miner.start(1)
```

Wait for `eth.blockNumber` to increase, then check:

```js
eth.pendingTransactions
web3.fromWei(eth.getBalance(eth.accounts[1]), "ether")
```

- `eth.pendingTransactions` returns `[]` when nothing is waiting.
- Account 1 should now show `5`.

### 8.6 Inspect a transaction

```js
eth.getTransaction("0xYOUR_TX_HASH")
eth.getTransactionReceipt("0xYOUR_TX_HASH")
```

- `getTransaction` shows sender, receiver, value and `blockNumber` (`null` if pending).
- `getTransactionReceipt` shows the block, gas used and `status: "0x1"` for success.

### 8.7 Create another account

```js
personal.newAccount("YOUR_PASSWORD")
```

Creates a new account inside the running node and returns its address.

### 8.8 Inspect the genesis block

```js
eth.getBlock(0)
```

Shows the genesis block: hash, difficulty, gas limit and timestamp.

---

## 9. Shut down and restart

Shut down:

1. Type `exit` in the console (Terminal 2).
2. Press `Ctrl+C` in the node terminal (Terminal 1).

Restart later. **Do not run `init` again:**

```bash
cd ~/myPrivateEth
geth --datadir node1 --networkid 2026 --nodiscover --http --http.api eth,net,web3,personal,miner,admin --allow-insecure-unlock
```

Then attach again with `geth attach node1/geth.ipc`.

---

## 10. Reset everything (start from scratch)

```bash
cd ~
rm -rf myPrivateEth
rm -rf ~/.puppeth/myprivateeth
```

- Deletes the project folder (chain data and accounts) and Puppeth's saved config.
- The DAG in `~/.ethash` is kept on purpose so mining starts quickly next time.

---

## 11. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `geth version` shows the wrong version | Another Geth is earlier on the PATH. Run `which -a geth` and remove it. |
| `puppeth: command not found` | Not copied to `/usr/local/bin`. Repeat section 2.2. |
| Mining is very slow at first | The DAG is being generated. Wait until it reaches 100%. |
| Balance of receiver still 0 after sending | The transaction is pending. Keep mining and wait for a new block. |
| `authentication needed: password or unlock` | Unlock the sender with `personal.unlockAccount`. |
| `could not decrypt key with given passphrase` | Wrong password for that account. |
| Balances are not zero at start | Accounts were pre-funded in the genesis. Reset (section 10) and leave the pre-funded prompt empty. |

---

## 12. Results

- Genesis block created with Puppeth (Ethash, chain ID 2026, no pre-funding)
- Node initialized, started and attached via the console
- Ether created by mining (2 ether per block)
- Ether transferred from account 0 to account 1 and confirmed in a mined block

---

## 13. Files in this repository

| File | Purpose |
|---|---|
| `myprivateeth.json` | The genesis block definition |
| `README.md` | This documentation |
| `.gitignore` | Keeps private data out of the repository |

> **Security warning:** never commit `node1/` (it contains your encrypted private keys and chain data) or any real passwords. Only commit `myprivateeth.json` and this README.

---

## 14. Push to GitHub

Create an empty repository on GitHub first (for example `my-first-private-blockchain`), then:

```bash
mkdir ~/blockchain-repo && cd ~/blockchain-repo
cp ~/myPrivateEth/myprivateeth.json .
cp /path/to/README.md .
printf "node1/\n*.ipc\nkeystore/\n.ethash/\n" > .gitignore

git init
git add .
git commit -m "My first private blockchain network - genesis block"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/my-first-private-blockchain.git
git push -u origin main
```

| Command | Description |
|---|---|
| `git init` | Creates a new local repository. |
| `git add .` | Stages all files (except those in `.gitignore`). |
| `git commit -m` | Saves a snapshot with a message. |
| `git branch -M main` | Names the branch `main`. |
| `git remote add origin URL` | Links the local repo to your GitHub repo. |
| `git push -u origin main` | Uploads the commit to GitHub. |

GitHub no longer accepts account passwords for `git push`. Use a **Personal Access Token** as the password, or log in with the GitHub CLI (`gh auth login`).
