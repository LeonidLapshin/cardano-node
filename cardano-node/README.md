
# Shelley Testnet

The `./scripts/shelley-testnet.sh` script starts up three nodes that are
connected via TCP sockets to each other and produce blocks according to the
algorithm selected (e.g. "BFT").  The blocks are shared among the nodes and
after verification integrated into a nodes ledger.  The user can submit
transactions to a node which includes them in its local mempool, and eventually
in the next block it will create.


```

 +---------+         +---------+
 |         | <-----> |         |
 | node 0  |         | node 1  |
 |         | <-+ +-> |         |
 +---------+   | |   +---------+
               v v

            +---------+
            |         |
            | node 2  |
            |         |
            +---------+


```

## Startup testnet

Add the next two lines to your $HOME/.tmux.conf file:
```
set-window-option -g mouse on
set -g default-terminal "tmux-256color"
```
In a first terminal start the central logging process:

    `./scripts/trace-acceptor.sh`

In a second terminal:

1.) create a `tmux` session

    `tmux new-session -s Demo`

2.) run the demo script in this new session

    `./scripts/shelley-testnet.sh`

The window of the terminal will be split into four panes showing the three
nodes running and a shell to enter commands for transaction submission, e.g.

```
./scripts/submit-tx.sh -n 2 --address a --amount 99 --txin ababa --txix 0

```
The above command will prepare a transaction of amount 99 to address _a_ and
sends the transaction for validation and integration into a block to node _2_.
Increment the last argument '--txix' to send in a new transaction.


3.) or you can run

    `./scripts/shelley-testnet-dns.sh`

    instead of `shelley-testnet.sh`.  It requires that the addresses
    `local.iohk.io` and `local6.iohk.io` resolve to `127.0.0.1` and `::1`
    respectively.  You can use [unbound](https://github.com/NLnetLabs/unbound)
    dns server.  You can use the following `/etc/unbound/unbound.conf` file:
    ```
    server:
      verbosity: 1
      local-data: "local.iohk.io A 127.0.0.1"
      local-data: "local6.iohk.io AAAA ::1"
    ```

# `cardano-cli`

A CLI utility to support a variety of key material operations (genesis, migration, pretty-printing..) for different system generations.

The general synopsis is as follows:
 ```
   cardano-cli SYSTEMVER COMMAND
```

NOTE: the exact invocation command depends on the environment.  If you have only
built `cardano-cli`, without installing it, then you have to prepend `cabal
new-run -- ` before `cardano-cli`.  We henceforth assume that the necessary
environment-specific adjustment has been made, so we only mention `cardano-cli`.

`SYSTEMVER` is one of the supported system generations: `byron-legacy`, `real-pbft` etc.
The full list can be seen in the output of `cardano-cli --help`, along with the list of
supported commands.

All commands have help available:

```
$ cabal new-exec -- cardano-cli real-pbft migrate-delegate-key-from --help
Usage: cardano-cli real-pbft migrate-delegate-key-from (--byron-legacy | --bft | --praos |
                                                        --mock-pbft | --real-pbft)
                                                        --to FILEPATH --from FILEPATH
  Migrate a delegate key from an older version.

Available options:
  --byron-legacy           Use the Byron/Ouroboros Classic suite of algorithms
  --bft                    Use the BFT consensus algorithm
  --praos                  Use the Praos consensus algorithm
  --mock-pbft              Use the Permissive BFT consensus algorithm using a
                           mock ledger
  --real-pbft              Use the Permissive BFT consensus algorithm using the
                           real ledger
  --to FILEPATH            Non-existent file to write the signing key to.
  --from FILEPATH          Signing key file to migrate.
```

## Genesis operations

The canned `scripts/genesis.sh` example provides a nice set of defaults and
illustrates available options.  Running it will produce a `./genesis.${systemStart}` directory,
where `${systemStart}` will be chosen 15 minutes in the future.

The directory will have the following content:

                            ## Affected by

 - `avvm-seed.*.seed`         - --avvm-entry-count and --avvm-entry-balance
 - `delegate-keys.*.key`      - --n-delegate-addresses
 - `delegation-cert.*.json`   - --n-delegate-addresses
 - `genesis-keys.*.key`       - --n-delegate-addresses, --total-balance
 - `poor-keys.*.key`          - --n-poor-addresses, --total-balance

Dummy genesis, that corresponds to `Test.Cardano.Chain.Genesis.Dummy.dummyConfig.configGenesisData`
can be dumped by the `dump-hardcoded-genesis` subcommand.

The `print-genesis-hash` subcommand will compute the genesis hash of a given genesis JSON file.
That value will be suitable for the `--genesis-hash` option of `cardano-node`.

## Key operations

Note that no key operation currently supports accepting password-protected keys.
The `keygen` subcommand, though, can generate such keys.

### Signing key generation & verification key extraction

Signing keys can be generated using the `keygen` subcommand, password protection being
controlled by the `--no-password` flag.

Extracting a verification key out of the signing key is performed by the `to-verification` subcommand.

### Delegate key migration

In order to continue using a delegate key from the Byron Legacy era in the new implementation,
it needs to be migrated over, which is done by the `migrate-delegate-key-from` subcommand:


```
$ cabal new-run -- cardano-cli real-pbft migrate-delegate-key-from byron-legacy \
                                         --from key0.sk --to key0.pbft
```

### Signing key queries

One can gather information about a signing key's properties through the `signing-key-public`
and `signing-key-address` subcommands (the latter requires the network magic):

```
$ cabal new-run -- cardano-cli real-pbft signing-key-public \
                                         --secret key0.pbft
public key hash: a2b1af0df8ca764876a45608fae36cf04400ed9f413de2e37d92ce04
     public key: sc4pa1pAriXO7IzMpByKo4cG90HCFD465Iad284uDYz06dHCqBwMHRukReQ90+TA/vQpj4L1YNaLHI7DS0Z2Vg==

$ cabal new-run -- cardano-cli real-pbft signing-key-address \
                                         --secret key0.pbft  \
                                         --testnet-magic 459045235
2cWKMJemoBakxhXgZSsMteLP9TUvz7owHyEYbUDwKRLsw2UGDrG93gPqmpv1D9ohWNddx
VerKey address with root e5a3807d99a1807c3f161a1558bcbc45de8392e049682df01809c488, attributes: AddrAttributes { derivation path: {} }
```

## Delegation

The `issue-delegation-certificate` subcommand enables generation of Byron genesis
delegation certificates, given the following inputs:

   - protocol magic
   - starting epoch of delegation
   - genesis delegator signing key
   - delegate verification key

To check the generated delegation certificate, you can use the `check-delegation` subcommand,
which would verify:

   - certificate signature validity
   - correspondence of the expected issuer/delegate with those on the certificate.

The expected issuer and delegate are supplied through the `--issuer-key` and `--delegate-key`
options.

## Transactions

### Submission

The `submit-tx` subcommand provides the option of submitting a pre-signed
transaction, in its raw wire format (see GenTx for Byron transactions).

The canned `scripts/submit-tx.sh` script will submit the supplied transaction to a testnet
launched by `scripts/shelley-testnet*.sh` family of scripts.

### Issuing genesis UTxO expenditure Tx

To make a transaction spending genesis UTxO, you can either use the `issue-genesis-utxo-expenditure`
subcommand directly, or, again use a canned script that will make a transaction tailored
for the aforementioned testnet cluster -- `scripts/issue-genesis-utxo-expenditure.sh`.

The script requires the target file name to write the transaction to, and optionally
allows specifying the target address and lovelace value to send.

The target address defaults to the 1-st richman key (`configuration/delegate-keys.001.key`)
of the testnet, and lovelace amount is almost the entirety of its funds.

# Running the wallet client

First you will need to start the core node with which the wallet client will
communicate.  You can do that with `./script/start-node.sh` (or
`./script/shelley-testnet2.sh`).  Then run

```
./scripts/start-wallet.sh --bft -n 0 -m 3
```

# Development

run *ghcid* with: `ghcid -c "cabal new-repl exe:cardano-node --reorder-goals"`