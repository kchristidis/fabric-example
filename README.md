# fabric-example (Kafka reset)

A succinct play-by-play of some major Fabric operations.

## Steps

### Bring up the Vagrant environment

```bash
$ git clone git@github.com:kchristidis/fabric.git
$ git checkout fabric-example-1.2-kafka-reset
$ cd devenv
$ vagrant up
$ vagrant ssh
```

ATTN: **All commands below** assume `$GOPATH/src/github.com/kchristidis/fabric-example/` as the starting point.

### Bootstrap

Edit `config/cryptogen.yaml` to reflect your network's structure.

Generate the crypto material for the network:

```bash
$ cryptogen generate --config=./config/cryptogen.yaml --output=crypto
clark.example.com
```

Create the genesis block for the ordering service and consortium:

```bash
$ configtxgen -configPath=./config -profile SystemChannel -channelID systemchannel -outputBlock ./config/genesis.block
2018-11-22 19:54:16.395 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-11-22 19:54:16.511 UTC [msp] getMspConfig -> INFO 002 Loading NodeOUs
2018-11-22 19:54:16.514 UTC [common/tools/configtxgen] doOutputBlock -> INFO 003 Generating genesis block
2018-11-22 19:54:16.518 UTC [common/tools/configtxgen] doOutputBlock -> INFO 004 Writing genesis block
```

Bring up the Docker composition for the entire network:

```bash
$ cd docker && docker-compose up -d
Creating network "fabric_default" with the default driver
Pulling joe.example.com (kchristidis/fabric-orderer-kafka-reset:1.2)...
1.2: Pulling from kchristidis/fabric-orderer-kafka-reset
b234f539f7a1: Already exists
55172d420b43: Already exists
5ba5bbeb6b91: Already exists
43ae2841ad7a: Already exists
f6c9c6de4190: Already exists
c6af77e36488: Already exists
964f7f4f22f3: Already exists
cb6f37b526e0: Pull complete
4ebc185c113a: Pull complete
b0a5c7958182: Pull complete
Digest: sha256:4896026315e285c1045ad9cf9ce94f50fed722c143d36fe3c4ab126642828cff
Status: Downloaded newer image for kchristidis/fabric-orderer-kafka-reset:1.2
Pulling zookeeper-bar.joe.example.com (hyperledger/fabric-zookeeper:latest)...
latest: Pulling from hyperledger/fabric-zookeeper
3b37166ec614: Pull complete
504facff238f: Pull complete
ebbcacd28e10: Pull complete
c7fb3351ecad: Pull complete
2e3debadcbf7: Pull complete
fc435e46e32e: Pull complete
a4922bafdce8: Pull complete
14675a1189ca: Pull complete
33f930d7053e: Pull complete
7aa21e006739: Pull complete
806ba27e29bb: Pull complete
04fff5ccfdcb: Pull complete
dbe82feb14ae: Pull complete
ba7a23bea382: Pull complete
5f6e0d1bbcbd: Pull complete
Digest: sha256:ac342ed87997175bfd557c53f7ffc6e0f8aa32bcaebb54a9bd55fb4c7f954802
Status: Downloaded newer image for hyperledger/fabric-zookeeper:latest
Pulling kafka-bar.joe.example.com (hyperledger/fabric-kafka:latest)...
latest: Pulling from hyperledger/fabric-kafka
3b37166ec614: Already exists
504facff238f: Already exists
ebbcacd28e10: Already exists
c7fb3351ecad: Already exists
2e3debadcbf7: Already exists
fc435e46e32e: Already exists
a4922bafdce8: Already exists
14675a1189ca: Already exists
33f930d7053e: Already exists
7aa21e006739: Already exists
806ba27e29bb: Already exists
5b93606e43dd: Pull complete
c28e9396b3a9: Pull complete
d4fac70bff73: Pull complete
Digest: sha256:82ac81938320d05a538b9fd6de0fcd54b5a999188cf9b08822cf25f9ad7970a9
Status: Downloaded newer image for hyperledger/fabric-kafka:latest
Creating zookeeper-bar.joe.example.com ...
Creating joe.example.com ...
Creating zookeeper-foo.joe.example.com ...
Creating zookeeper-bar.joe.example.com
Creating zookeeper-foo.joe.example.com
Creating zookeeper-bar.joe.example.com ... done
Creating kafka-bar.joe.example.com ...
Creating joe.example.com ... done
Creating zookeeper-foo.joe.example.com ... done
Creating kafka-foo.joe.example.com ...
Creating p0.clark.example.com
Creating p0.clark.example.com ... done
Creating admin.clark.example.com ...
Creating admin.clark.example.com ... done
```

```bash
$ cd docker && docker-compose ps
            Name                           Command               State              Ports
-----------------------------------------------------------------------------------------------------
admin.clark.example.com         /bin/bash                        Up
joe.example.com                 orderer                          Up      7050/tcp
kafka-bar.joe.example.com       /docker-entrypoint.sh /opt ...   Up      9092/tcp, 9093/tcp
kafka-foo.joe.example.com       /docker-entrypoint.sh /opt ...   Up      9092/tcp, 9093/tcp
p0.clark.example.com            peer node start                  Up
zookeeper-bar.joe.example.com   /docker-entrypoint.sh zkSe ...   Up      2181/tcp, 2888/tcp, 3888/tcp
zookeeper-foo.joe.example.com   /docker-entrypoint.sh zkSe ...   Up      2181/tcp, 2888/tcp, 3888/tcp
```

### Create a channel

Generate the channel creation transaction:

```bash
$ configtxgen -configPath=./config -profile ClarksChannel -channelID clarkschannel -outputCreateChannelTx config/clarkschannel.tx
2018-11-22 19:58:58.699 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-11-22 19:58:58.765 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-11-22 19:58:58.785 UTC [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-11-22 19:58:58.793 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 004 Writing new channel tx
```

Create the channel via Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel create --orderer joe.example.com:7050 --channelID clarkschannel --file $SHARED_PATH/clarkschannel.tx --tls --cafile $ORDERER_CA
2018-11-22 19:59:25.571 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 19:59:25.597 UTC [cli/common] readBlock -> INFO 002 Got status: &{NOT_FOUND}
2018-11-22 19:59:25.601 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-11-22 19:59:25.803 UTC [cli/common] readBlock -> INFO 004 Got status: &{SERVICE_UNAVAILABLE}
2018-11-22 19:59:25.807 UTC [channelCmd] InitCmdFactory -> INFO 005 Endorser and orderer connections initialized
2018-11-22 19:59:26.011 UTC [cli/common] readBlock -> INFO 006 Received block: 0
```

Use the returned block and have the peer join the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel join --blockpath clarkschannel.block
2018-11-22 20:00:01.322 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 20:00:01.371 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

### Install, instantiate, invoke chaincode

Install a chaincode on Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.0 --path github.com/kchristidis/fabric-example/chaincode
2018-11-22 20:00:18.292 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-11-22 20:00:18.293 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-11-22 20:00:18.475 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

Instantiate it on the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode instantiate --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA --channelID clarkschannel --name clarkschaincode --version 1.0 -c '{"Args":["init","a", "100", "b","200"]}' --policy "OR ('ClarkMSP.member')"
2018-11-22 20:00:42.114 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-11-22 20:00:42.114 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
```

Query the chaincode to confirm the instantiation was successful:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
100
```

Invoke the chaincode:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode invoke --orderer joe.example.com:7050  --tls --cafile $ORDERER_CA  --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["invoke","a","b","10"]}'
2018-11-22 20:01:36.964 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 001 Chaincode invoke successful. result: status:200
```

Query the chaincode again to confirm the invocation was successful:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
90
```

### Switch backing Kafka cluster

Let's get the latest configuration from both channels: `systemchannel`, and `clarkschannel`

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel fetch config clark_config_1.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-11-22 20:06:38.753 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 20:06:38.755 UTC [cli/common] readBlock -> INFO 002 Received block: 2
2018-11-22 20:06:38.756 UTC [cli/common] readBlock -> INFO 003 Received block: 0
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel fetch config system_config_1.pb --channelID systemchannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-11-22 20:07:50.424 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 20:07:50.426 UTC [cli/common] readBlock -> INFO 002 Received block: 1
2018-11-22 20:07:50.428 UTC [cli/common] readBlock -> INFO 003 Received block: 0
```

Convert into JSON:

```bash
$ docker exec -ti admin.clark.example.com bash
$ configtxlator proto_decode --input clark_config_1.pb --type common.Block | jq .data.data[0].payload.data.config > clark_config_1.json
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ configtxlator proto_decode --input system_config_1.pb --type common.Block | jq .data.data[0].payload.data.config > system_config_1.json
```

Copy these files to `$SHARED_PATH`.

```bash
$ docker exec -ti admin.clark.example.com bash
$ cp *.json $SHARED_PATH
```

In your localhost, `cd` into the `SHARED_PATH` directory which is `config`:

```bash
$ cd config
```

Replace all instances of `kafka-foo` with `kafka-bar` and save the resulting files as, `clark_config_mod.json` and `system_config_mod.json` respectively.

At this point, your `config` directory should read like this:

```bash
$ $ ls -1 config
clark_config_1.json
clark_config_mod.json
clarkschannel.tx
configtx.yaml
cryptogen.yaml
genesis.block
system_config_1.json
system_config_mod.json
```

For each channel, create the protobuf structure that captures the delta between the two configurations (the existing one, and the proposed):

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ configtxlator proto_encode --input clark_config_1.json --type common.Config --output clark_config_1.cc.pb
$ configtxlator proto_encode --input clark_config_mod.json --type common.Config --output clark_config_mod.cc.pb
$ configtxlator compute_update --channel_id clarkschannel --original clark_config_1.cc.pb --updated clark_config_mod.cc.pb --output clark_config_upd.pb
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ configtxlator proto_encode --input system_config_1.json --type common.Config --output system_config_1.cc.pb
$ configtxlator proto_encode --input system_config_mod.json --type common.Config --output system_config_mod.cc.pb
$ configtxlator compute_update --channel_id systemchannel --original system_config_1.cc.pb --updated system_config_mod.cc.pb --output system_config_upd.pb
```

For each channel, convert the protobuf structure from the previous step into the envelope structure that will be signed and submitted to the ordering service:

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ configtxlator proto_decode --input clark_config_upd.pb --type common.ConfigUpdate | jq . > clark_config_upd.json
$ echo '{"payload":{"header":{"channel_header":{"channel_id":"clarkschannel", "type":2}},"data":{"config_update":'$(cat clark_config_upd.json)'}}}' | jq . > clark_config_upd_env.json
$ configtxlator proto_encode --input clark_config_upd_env.json --type common.Envelope --output clark_config_upd.env.pb
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ configtxlator proto_decode --input system_config_upd.pb --type common.ConfigUpdate | jq . > system_config_upd.json
$ echo '{"payload":{"header":{"channel_header":{"channel_id":"systemchannel", "type":2}},"data":{"config_update":'$(cat system_config_upd.json)'}}}' | jq . > system_config_upd_env.json
$ configtxlator proto_encode --input system_config_upd_env.json --type common.Envelope --output system_config_upd.env.pb
```

For each channel, have the admin of Clark's peer sign the envelope generated in the previous step:

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ peer channel signconfigtx --file clark_config_upd.env.pb
2018-11-22 20:59:27.565 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ peer channel signconfigtx --file system_config_upd.env.pb
2018-11-22 20:59:35.883 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
```

For each channel, send the corresponding signed envelope to the ordering service:

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ peer channel update --file clark_config_upd.env.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-11-22 21:00:37.824 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 21:00:37.948 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```

```bash
$ docker exec -ti admin.clark.example.com bash
$ cd $SHARED_PATH
$ peer channel update --file system_config_upd.env.pb --channelID systemchannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-11-22 21:00:55.857 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-11-22 21:00:55.978 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```

Up until this point, the ordering service has been using Kafka broker `kafka-foo.joe.example.com`. With the steps above, we have successfully pushed a configuration update that switches both channels on the system (`systemchannel` and `clarkschannel`) to a different Kafka broker: `kafka-bar.joe.example.com`. This change won't take effect until we restart the ordering service.

### Restart the ordering service

If we restart the ordering service the usual way, the ordering service will attempt to resume the Kafka session by incrementing  the latest offset it has recorded for every channel (i.e. Kafka topic/partition) and seeking to it. If we're using a fresh Kafka cluster, this operation will fail.

We therefore restart the ordering service with the `RESET` environment variable set to `true`. (N.B. This is already set in the provided Docker Compose file in this repo.) This environment variable instructs the ordering service to just seek to the latest available offset for each channel and call it a day.

```bash
$ docker restart joe.example.com
```

We expect the ordering service to pick up from where it left, and start using `kafka-bar.joe.example.com` without issues. Let's invoke the chaincode we had instantiated before restarting the ordering service.

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode invoke --orderer joe.example.com:7050  --tls --cafile $ORDERER_CA  --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["invoke","a","b","10"]}'
2018-11-22 22:40:45.742 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 001 Chaincode invoke successful. result: status:200
```

The request was ordered without issues. Let's query the chaincode.

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
80
```

This the result that we expect — we had already invoked the chaincode once pre-restart, so its value before this last invocation was 90.

### Caveat emptor

If you intend to use this feature, please note down the following:

1. You should not write anything to the new Kafka cluster until the ordering service has been restarted.
2. If you wish to onboard a new OSN, you will need to copy the ledger of an existing OSN to it, and the donor OSN should be fully synced up with the _previous_ Kafka cluster (`kafka-foo` in our case) _and_ also have messages appended to its ledgers that come from the _new_ Kafka cluster (`kafka-bar` in our case). This is because the new OSN (which should be onboarded with the `RESET` flag set to `false`) will use the encoded offsets in its ledgers when interacting with the backing Kafka cluster. We need to make sure that the offsets it's using are actually persisted in the current Kafka cluster.