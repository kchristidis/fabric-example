# fabric-example

A succinct play-by-play of some major Fabric operations.

## Steps

### Bring up the Vagrant environment

```bash
$ git clone git@github.com:kchristidis/fabric.git
$ git checkout fabric-example-1.2
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
amelia.example.com
```

Create the genesis block for the ordering service and consortium:

```bash
$ configtxgen -configPath=./config -profile SystemGenesis -channelID systemchain -outputBlock ./config/genesis.block
2018-07-28 13:39:35.906 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-28 13:39:35.992 UTC [msp] getMspConfig -> INFO 002 Loading NodeOUs
2018-07-28 13:39:36.003 UTC [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-07-28 13:39:36.004 UTC [common/tools/configtxgen] doOutputBlock -> INFO 004 Generating genesis block
2018-07-28 13:39:36.007 UTC [common/tools/configtxgen] doOutputBlock -> INFO 005 Writing genesis block
```

Bring up the Docker composition for the entire network:

```bash
$ cd docker && docker-compose up -d
Creating network "fabric_default" with the default driver
Creating joe.example.com ...
Creating joe.example.com ... done
Creating p0.clark.example.com ...
Creating p0.amelia.example.com ...
Creating p0.clark.example.com
Creating p0.clark.example.com ... done
Creating admin.clark.example.com ...
Creating p0.amelia.example.com ... done
Creating admin.amelia.example.com ...
Creating admin.clark.example.com ... done
```

```bash
$ cd docker && docker-compose ps
          Name                 Command       State                       Ports
---------------------------------------------------------------------------------------------------
admin.amelia.example.com   /bin/bash         Up
admin.clark.example.com    /bin/bash         Up
joe.example.com            orderer           Up      0.0.0.0:7050->7050/tcp
p0.amelia.example.com      peer node start   Up      0.0.0.0:9051->7051/tcp, 0.0.0.0:9053->7053/tcp
p0.clark.example.com       peer node start   Up      0.0.0.0:8051->7051/tcp, 0.0.0.0:8053->7053/tcp
```

### Create a channel

Generate the channel creation transaction:

```bash
$ configtxgen -configPath=./config -profile ClarksChannel -channelID clarkschannel -outputCreateChannelTx config/clarkschannel.tx
2018-07-28 13:41:05.479 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-28 13:41:05.516 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-07-28 13:41:05.530 UTC [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-07-28 13:41:05.535 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 004 Writing new channel tx
```

Create the channel via Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel create --orderer joe.example.com:7050 --channelID clarkschannel --file $SHARED_PATH/clarkschannel.tx --tls --cafile $ORDERER_CA
2018-07-28 13:41:27.417 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 13:41:27.435 UTC [cli/common] readBlock -> INFO 002 Got status: &{NOT_FOUND}
2018-07-28 13:41:27.442 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-07-28 13:41:27.645 UTC [cli/common] readBlock -> INFO 004 Received block: 0
```

Use the returned block and have the peer join the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel join --blockpath clarkschannel.block
2018-07-28 13:42:02.755 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 13:42:02.804 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

### Install, instantiate, invoke chaincode

Install a chaincode on Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.0 --path github.com/kchristidis/fabric-example/chaincode
2018-07-28 13:43:10.203 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-28 13:43:10.203 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-07-28 13:43:10.401 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

Instantiate it on the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode instantiate --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA --channelID clarkschannel --name clarkschaincode --version 1.0 -c '{"Args":["init","a", "100", "b","200"]}' --policy "OR ('ClarkMSP.member')"
2018-07-28 13:43:44.929 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-28 13:43:44.929 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
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
2018-07-28 13:45:02.241 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 001 Chaincode invoke successful. result: status:200
```

Query the chaincode again to confirm the invocation was successful:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
90
```

### Add another org to the channel

Let's get the latest configuration from the channel.

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel fetch config config_1.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-07-28 13:52:53.724 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 13:52:53.726 UTC [cli/common] readBlock -> INFO 002 Received block: 2
2018-07-28 13:52:53.728 UTC [cli/common] readBlock -> INFO 003 Received block: 0
```

Convert into JSON:

```bash
$ docker exec -ti admin.clark.example.com bash
$ configtxlator proto_decode --input config_1.pb --type common.Block | jq .data.data[0].payload.data.config > config_1.json
```

Let's print the definition for the new org into a JSON file. Do this on your localhost, on this project's root folder (because we need access to the `crypto` folder):

```bash
$ configtxgen -configPath=./config -channelID clarkschannel -printOrg Amelia > config/amelia.json
2018-07-28 14:02:37.306 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-28 14:02:37.339 UTC [msp] getMspConfig -> INFO 002 Loading NodeOUs
```

Add the new org to `Channel/Application`:

```bash
$ docker exec -ti admin.clark.example.com bash
$ jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Amelia":.[1]}}}}}' config_1.json $SHARED_PATH/amelia.json > config_mod.json
```

Create the protobuf structure that captures the delta between the two configurations (the existing one, and the proposed):

```bash
$ docker exec -ti admin.clark.example.com bash
$ configtxlator proto_encode --input config_1.json --type common.Config --output config_1.cc.pb
$ configtxlator proto_encode --input config_mod.json --type common.Config --output config_mod.cc.pb
$ configtxlator compute_update --channel_id clarkschannel --original config_1.cc.pb --updated config_mod.cc.pb --output config_upd.pb
```

Convert this into the envelope structure that will be signed and submitted to the ordering service:

```bash
$ docker exec -ti admin.clark.example.com bash
$ configtxlator proto_decode --input config_upd.pb --type common.ConfigUpdate | jq . > config_upd.json
$ echo '{"payload":{"header":{"channel_header":{"channel_id":"clarkschannel", "type":2}},"data":{"config_update":'$(cat config_upd.json)'}}}' | jq . > config_upd_env.json
$ configtxlator proto_encode --input config_upd_env.json --type common.Envelope --output config_upd.env.pb
```

Have the admin of Clark's peer sign the envelope:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel signconfigtx --file config_upd.env.pb
2018-07-28 14:06:22.880 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
```

Now send the signed envelope the ordering service:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel update --file config_upd.env.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-07-28 14:06:48.764 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 14:06:48.779 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```

Now have the new org get the channel's genesis block:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer channel fetch 0 clarkschannel.block --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-07-28 14:07:14.893 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 14:07:14.896 UTC [cli/common] readBlock -> INFO 002 Received block: 0
```

And use it to join the channel:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer channel join --blockpath clarkschannel.block
2018-07-28 14:07:26.107 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-28 14:07:26.146 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

### Modify the chaincode

Let's modify the chaincode's endorsement policy so as to include the new org.

Let's install the chaincode on Clark's peer. Same exact bytes, just tag it with a different version.

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.1 --path github.com/kchristidis/fabric-example/chaincode
2018-07-28 14:07:50.613 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-28 14:07:50.613 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-07-28 14:07:50.824 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

Do the same on Amelia's peer:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.1 --path github.com/kchristidis/fabric-example/chaincode
2018-07-28 14:08:20.803 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-28 14:08:20.804 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-07-28 14:08:21.000 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

Have the admin on Clark's peer upgrade the chaincode. Note that the call needs to be made by an entity satisfying the instantiation policy. Since that was not explicitly set, it defaults to any admin of any org which was present in the channel at the time of invoking the upgrade.

```bash
$ peer chaincode upgrade --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA --channelID clarkschannel --name clarkschaincode --version 1.1 -c '{"Args":["init","a","100","b","200"]}' --policy "OR ('ClarkMSP.member','AmeliaMSP.member')"
2018-07-28 14:09:23.538 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-28 14:09:23.538 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
```