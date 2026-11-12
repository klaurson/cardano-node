# cardano-node

A Cardano relay. One Docker Compose stack, two profiles. The default is the preprod testnet, sized for a Hetzner CX23. The same checkout runs mainnet on a Hetzner CX53 after one change in `.env`.

This is a relay. It does not hold stake-pool keys, a pledge, or a pool registration.

The image is the official stable release:

`ghcr.io/intersectmbo/cardano-node:11.0.1`

digest `sha256:007da778998d0bdb043fb834c62b9da1c95e33610f31f6f9cccab29a945e2b99`

That tag is [cardano-node 11.0.1](https://github.com/IntersectMBO/cardano-node/releases/tag/11.0.1) (5 May 2026, git `97036a66`). The 11.1.0 pre-release and the 11.2 pre-release line are not used.

## Run the testnet profile

Install [Docker Engine](https://docs.docker.com/engine/install/) and the Compose plugin. On the CX23 (2 vCPU, 4 GB RAM, 40 GB disk):

```sh
git clone https://github.com/klaurson/cardano-node.git
cd cardano-node
./apply
```

`.env` already contains `PROFILE=testnet`. That profile selects the preprod network baked into the 11.0.1 image (`LedgerDB.Backend` is `V2InMemory`), a 3 GB container limit, a 2400 MB Haskell heap cap, and 1.5 CPUs. Preprod's official minimum is 2 cores, 4 GB RAM, and 20 GB disk, which fits a CX23 with room for the operating system.

The node listens on TCP 3001. Open that port inbound and outbound on the host firewall. The first start syncs from the preprod bootstrap peer and the peer snapshot shipped in the image. The chain database is the Docker volume `cardano-node-preprod-db`.

```sh
./apply logs -f
./apply ps
```

A one-shot handshake against the local socket. On 11.0.1, a unix socket answers a version query:

```sh
./apply exec cardano-node cardano-cli ping -u /ipc/node.socket -m 1 -Q
```

Preprod's network magic is 1, so the negotiated line ends in `1`. Mainnet's magic is `764824073`; use that value for `-m` after switching profiles.

Stop the container and keep the chain:

```sh
./apply down
```

## Switch to mainnet on a CX53

On the CX53 (16 vCPU, 32 GB RAM, 320 GB disk), use the same files. Change the one setting in `.env`:

```sh
PROFILE=mainnet
```

Then apply again:

```sh
./apply
```

That single line is the whole switch. `./apply` maps it to:

| | testnet (default) | mainnet |
| --- | --- | --- |
| Network | preprod (magic 1) | mainnet (magic 764824073) |
| Machine | CX23: 2 vCPU, 4 GB RAM, 40 GB disk | CX53: 16 vCPU, 32 GB RAM, 320 GB disk |
| Container memory | 3 GB, no extra swap | 30 GB, no extra swap |
| Haskell heap (`GHCRTS`) | `-M2400M` | `-M26G` |
| CPU quota | 1.5 | 14 |
| Chain volume | `cardano-node-preprod-db` | `cardano-node-mainnet-db` |

Docker rejects a CPU quota higher than the host's CPU count. The mainnet quota of 14 fits the CX53's 16 vCPUs.

The image launcher uses two Haskell capabilities (`-N2`) on both networks. That matches the CX23. On the CX53 the extra cores stay available for the operating system, disk, and the node's non-Haskell threads. The ledger backend stays the image default, `V2InMemory`. Release 11.0.1 has known bugs in the LSM backend, so this stack does not switch to it.

Each network has its own Docker volume. A preprod chain is not reused for mainnet. `./apply down` leaves the volumes in place. Delete one only when you mean to resync:

```sh
docker volume rm cardano-node-preprod-db
```

### Official mainnet requirements

From the [11.0.1 release](https://github.com/IntersectMBO/cardano-node/releases/tag/11.0.1) minimum system requirements, which the later release notes repeat:

- 24 GB of RAM when running with the InMemory backend (8 GB with the OnDisk backend is listed as pending confirmation)
- 300 GB of free storage, with 350 GB recommended for growth
- Two or more cores at 1.6 GHz or faster, or 2 GHz or faster for a relay

The [Cardano Developer Portal](https://developers.cardano.org/docs/get-started/infrastructure/node/installing-cardano-node/) (written for 11.0.1) lists the same 24 GB RAM figure and 300 GB as the mainnet minimum, and notes that the chain grows past that. Testnet is listed there as 4 GB RAM and 20 GB disk.

A CX53 has 32 GB RAM, so the 24 GB InMemory working set fits. The profile gives the container 30 GB and caps the Haskell heap at 26 GB, leaving about 2 GB for the operating system. The 320 GB disk covers the 300 GB minimum after a small system install. It does not cover the 350 GB growth recommendation. Grow the disk before the chain fills it. `./apply` prints a warning when free space is under 300 GB.

Mainnet sync from the public backbone peers is a long download of the full chain. Allow TCP 3001 inbound and outbound.

Check the profile before starting:

```sh
./apply config
```

The resolved service shows `NETWORK=mainnet` and `GHCRTS=-M26G`.

## What stays out of git

This repository is the install. The chain, the socket, and any key material are not part of it.

Do not commit:

- Cold, KES, VRF, or payment keys (`*.skey`, `*.vkey`)
- Operational certificates (`*.cert`, `node.cert`)
- Wallet seeds, recovery phrases, or stake addresses
- A `secrets/` or `keys/` directory
- The chain database or `node.socket`

`.env` is tracked and holds one line, `PROFILE`. Keep it that way. Do not put hostnames with credentials, API tokens, or key paths into it.

`.gitignore` already excludes key files, `secrets/`, `keys/`, `db/`, and `ipc/`. The node never mounts a key directory. Leave `CARDANO_BLOCK_PRODUCER` unset. The 11.0.1 image runs a relay when `NETWORK` is set, and it does not read pool keys in that mode.

The chain volumes can contain nothing you would publish, and they are large. They live in Docker (`docker volume ls`), not in the working tree.
