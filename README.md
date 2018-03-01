# fabric-example

A succinct play-by-play of some major Fabric operations.

## Steps

### Bring up the Vagrant environment

```bash
$ git clone git@github.com:kchristidis/fabric.git
$ git checkout fabric-example
$ cd devenv
$ vagrant up
$ vagrant ssh
$ make docker cryptogen configtxgen
docker pull hyperledger/fabric-couchdb:x86_64-0.4.6
x86_64-0.4.6: Pulling from hyperledger/fabric-couchdb
8f7c85c2269a: Already exists
9e72e494a6dd: Already exists
3009ec50c887: Already exists
9d5ffccbec91: Already exists
e872a2642ce1: Already exists
9b84958a26b3: Already exists
68d4ced7ec19: Already exists
ff1d2b44d88d: Already exists
99e6a41c35bd: Already exists
87b2e4a0b9d2: Already exists
55f108d3ee4a: Already exists
9e76f6c2c001: Pull complete
368be4b23f81: Pull complete
[..]
build/bin/configtxlator
CGO_CFLAGS=" " GOBIN=/opt/gopath/src/github.com/hyperledger/fabric/build/bin go install -tags "experimental" -ldflags "-X github.com/hyperledger/fabric/common/tools/configtxlator/metadata.Version=1.1.0-beta-snapshot-8ee2bcb" github.com/hyperledger/fabric/common/tools/configtxlator
Binary available as build/bin/configtxlator
$ export PATH="$PATH:$GOPATH/src/github.com/hyperledger/fabric/build/bin"
$ cd $GOPATH/src/github.com/kchristidis/fabric-example/
```

All commands below assume `$GOPATH/src/github.com/kchristidis/fabric-example/` as the starting point.

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
$ FABRIC_CFG_PATH=./config configtxgen -profile SystemGenesis -channelID systemchain -outputBlock ./config/genesis.block
2018-02-20 18:20:39.097 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-02-20 18:20:39.151 UTC [common/tools/configtxgen] doOutputBlock -> INFO 002 Generating genesis block
2018-02-20 18:20:39.152 UTC [common/tools/configtxgen] doOutputBlock -> INFO 003 Writing genesis block
```

Bring up the Docker composition for the entire network:

```bash
$ cd docker
$ docker-compose up -d
Creating network "net_example" with the default driver
Creating joe.example.com ...
Creating joe.example.com ... done
Creating p0.clark.example.com ...
Creating p0.amelia.example.com ...
Creating p0.clark.example.com
Creating p0.clark.example.com ... done
Creating admin.clark.example.com ...
Creating p0.amelia.example.com ... done
Creating admin.amelia.example.com ...
Creating admin.amelia.example.com ... done
$ docker-compose ps
          Name                      Command            State                       Ports
-------------------------------------------------------------------------------------------------------------
admin.amelia.example.com   /bin/bash -c sleep 100000   Up
admin.clark.example.com    /bin/bash -c sleep 100000   Up
joe.example.com            orderer                     Up      0.0.0.0:7050->7050/tcp
p0.amelia.example.com      peer node start             Up      0.0.0.0:9051->7051/tcp, 0.0.0.0:9053->7053/tcp
p0.clark.example.com       peer node start             Up      0.0.0.0:8051->7051/tcp, 0.0.0.0:8053->7053/tcp
```

### Create a channel

Generate the channel creation transaction:

```bash
$ FABRIC_CFG_PATH=./config configtxgen -profile ClarksChannel -channelID clarkschannel -outputCreateChannelTx config/clarkschannel.tx
2018-02-20 18:20:42.168 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-02-20 18:20:42.183 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-02-20 18:20:42.221 UTC [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 003 Writing new channel tx
```

Create the channel via Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel create --orderer joe.example.com:7050 --channelID clarkschannel --file $SHARED_PATH/clarkschannel.tx --tls --cafile $ORDERER_CA
2018-02-20 18:42:18.372 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 18:42:18.372 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 18:42:18.381 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 18:42:18.383 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-02-20 18:42:18.384 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-02-20 18:42:18.384 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-02-20 18:42:18.384 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-02-20 18:42:18.384 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AA7060A08436C61726B4D5350129A06...120E0A0C4D79436F6E736F727469756D
2018-02-20 18:42:18.385 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: 16B9A2DD80AD6A7BD739AB0C35BE8B6EB903475715927F014A8DFF944E8B058D
2018-02-20 18:42:18.385 UTC [msp] GetLocalMSP -> DEBU 00a Returning existing local MSP
2018-02-20 18:42:18.385 UTC [msp] GetDefaultSigningIdentity -> DEBU 00b Obtaining default signing identity
2018-02-20 18:42:18.386 UTC [msp] GetLocalMSP -> DEBU 00c Returning existing local MSP
2018-02-20 18:42:18.386 UTC [msp] GetDefaultSigningIdentity -> DEBU 00d Obtaining default signing identity
2018-02-20 18:42:18.386 UTC [msp/identity] Sign -> DEBU 00e Sign: plaintext: 0AE2060A1908021A06088AD8B1D40522...DB9EFACD4D2A7528960082AB256B0FFD
2018-02-20 18:42:18.386 UTC [msp/identity] Sign -> DEBU 00f Sign: digest: A7684F37318FF14B599C7F6F393FD7C1230BC99DFE614C0BF84F8A741A69AA65
2018-02-20 18:42:18.430 UTC [msp] GetLocalMSP -> DEBU 010 Returning existing local MSP
2018-02-20 18:42:18.430 UTC [msp] GetDefaultSigningIdentity -> DEBU 011 Obtaining default signing identity
2018-02-20 18:42:18.431 UTC [msp] GetLocalMSP -> DEBU 012 Returning existing local MSP
2018-02-20 18:42:18.431 UTC [msp] GetDefaultSigningIdentity -> DEBU 013 Obtaining default signing identity
2018-02-20 18:42:18.431 UTC [msp/identity] Sign -> DEBU 014 Sign: plaintext: 0AE2060A1908021A06088AD8B1D40522...B876331F916712080A021A0012021A00
2018-02-20 18:42:18.431 UTC [msp/identity] Sign -> DEBU 015 Sign: digest: B0907BC5A4C265D813309CDB8FD2F88BFE4268F636CC79495A418D3DFB9BF925
2018-02-20 18:42:18.431 UTC [channelCmd] readBlock -> DEBU 016 Got status: &{NOT_FOUND}
2018-02-20 18:42:18.431 UTC [msp] GetLocalMSP -> DEBU 017 Returning existing local MSP
2018-02-20 18:42:18.431 UTC [msp] GetDefaultSigningIdentity -> DEBU 018 Obtaining default signing identity
2018-02-20 18:42:18.456 UTC [channelCmd] InitCmdFactory -> INFO 019 Endorser and orderer connections initialized
2018-02-20 18:42:18.657 UTC [msp] GetLocalMSP -> DEBU 01a Returning existing local MSP
2018-02-20 18:42:18.657 UTC [msp] GetDefaultSigningIdentity -> DEBU 01b Obtaining default signing identity
2018-02-20 18:42:18.658 UTC [msp] GetLocalMSP -> DEBU 01c Returning existing local MSP
2018-02-20 18:42:18.658 UTC [msp] GetDefaultSigningIdentity -> DEBU 01d Obtaining default signing identity
2018-02-20 18:42:18.658 UTC [msp/identity] Sign -> DEBU 01e Sign: plaintext: 0AE2060A1908021A06088AD8B1D40522...EC6555CAE4A212080A021A0012021A00
2018-02-20 18:42:18.658 UTC [msp/identity] Sign -> DEBU 01f Sign: digest: 38645EEF16D5CDB06D00170E1C9F77C09C59BD98DC23331F52A3EADD3E5AD310
2018-02-20 18:42:18.662 UTC [channelCmd] readBlock -> DEBU 020 Received block: 0
2018-02-20 18:42:18.664 UTC [main] main -> INFO 021 Exiting.....
```

Use the returned block and have the peer join the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel join --blockpath clarkschannel.block
2018-02-20 18:42:44.213 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 18:42:44.213 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 18:42:44.220 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 18:42:44.221 UTC [msp/identity] Sign -> DEBU 004 Sign: plaintext: 0AA4070A5B08011A0B08A4D8B1D40510...16319C47774F1A080A000A000A000A00
2018-02-20 18:42:44.221 UTC [msp/identity] Sign -> DEBU 005 Sign: digest: 5D71B90E10E426F9F9A2AE9AD0B435F11F3AC0EE4568A55D7D91DD42B443DF0A
2018-02-20 18:42:44.260 UTC [channelCmd] executeJoin -> INFO 006 Successfully submitted proposal to join channel
2018-02-20 18:42:44.260 UTC [main] main -> INFO 007 Exiting.....
```

### Install, instantiate, invoke chaincode

Install a chaincode on Clark's peer:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.0 --path github.com/kchristidis/fabric-example/chaincode
2018-02-20 19:09:38.026 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:09:38.026 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:09:38.026 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-20 19:09:38.026 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-20 19:09:38.026 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-20 19:09:38.065 UTC [golang-platform] getCodeFromFS -> DEBU 006 getCodeFromFS github.com/kchristidis/fabric-example/chaincode
2018-02-20 19:09:38.363 UTC [golang-platform] func1 -> DEBU 007 Discarding GOROOT package fmt
2018-02-20 19:09:38.363 UTC [golang-platform] func1 -> DEBU 008 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-02-20 19:09:38.363 UTC [golang-platform] func1 -> DEBU 009 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-02-20 19:09:38.363 UTC [golang-platform] func1 -> DEBU 00a Discarding GOROOT package strconv
2018-02-20 19:09:38.364 UTC [golang-platform] GetDeploymentPayload -> DEBU 00b done
2018-02-20 19:09:38.364 UTC [container] WriteFileToPackage -> DEBU 00c Writing file to tarball: src/github.com/kchristidis/fabric-example/chaincode/simple.go
2018-02-20 19:09:38.373 UTC [msp/identity] Sign -> DEBU 00d Sign: plaintext: 0AA5070A5C08031A0C08F2E4B1D40510...9F61FD3B0000FFFFD4B49D15001A0000
2018-02-20 19:09:38.373 UTC [msp/identity] Sign -> DEBU 00e Sign: digest: ACBA8E859B198CF82E3A3C6953AE0DBDFC664A3CC51147AC50FCB41FA3EB2032
2018-02-20 19:09:38.381 UTC [chaincodeCmd] install -> DEBU 00f Installed remotely response:<status:200 payload:"OK" >
2018-02-20 19:09:38.382 UTC [main] main -> INFO 010 Exiting.....
```

Instantiate it on the channel:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode instantiate --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA --channelID clarkschannel --name clarkschaincode --version 1.0 -c '{"Args":["init","a", "100", "b","200"]}' --policy "OR ('ClarkMSP.member')"
2018-02-20 19:34:19.291 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:34:19.291 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:34:19.297 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-20 19:34:19.297 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-20 19:34:19.298 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-20 19:34:19.298 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0AB4070A6B08031A0C08BBF0B1D40510...6B4D53500A04657363630A0476736363
2018-02-20 19:34:19.299 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: C7AF2ACD27BE073BA41003FEEE791F78734BE4DADB9482F82BF6BD2913CDB75C
2018-02-20 19:34:19.852 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AB4070A6B08031A0C08BBF0B1D40510...45A8C4013FA90A7DDF29610890736D4F
2018-02-20 19:34:19.852 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: 3A1ADF48202F1739157F3D0B840B69EA664217D211F73D97A689ED0C18F52F13
2018-02-20 19:34:19.857 UTC [main] main -> INFO 00a Exiting.....
```

Query the chaincode to confirm the instantiation was successful:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
2018-02-20 19:35:41.285 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:35:41.285 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:35:41.285 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-20 19:35:41.285 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-20 19:35:41.286 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-20 19:35:41.287 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0ABF070A7608031A0C088DF1B1D40510...636F64651A0A0A0571756572790A0161
2018-02-20 19:35:41.287 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 4CADD49522E3E581710033AEED454F7122FDD69BC993F2400E4E5975523E925A
Query Result: 100
2018-02-20 19:35:41.302 UTC [main] main -> INFO 008 Exiting.....
```

Invoke the chaincode:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode invoke --orderer joe.example.com:7050  --tls --cafile $ORDERER_CA  --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["invoke","a","b","10"]}'
2018-02-20 19:35:49.890 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:35:49.890 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:35:49.895 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-20 19:35:49.896 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-20 19:35:49.896 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-20 19:35:49.897 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0ABF070A7608031A0C0895F1B1D40510...696E766F6B650A01610A01620A023130
2018-02-20 19:35:49.897 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: A0C84D897E0913026690785005F61A12CD139BC1CCD140F3E25C494902D5CEDC
2018-02-20 19:35:49.932 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0ABF070A7608031A0C0895F1B1D40510...E0E94A4F16E4852E3A4AD28CAFE4CA49
2018-02-20 19:35:49.932 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: E6E0569709D0270648CFAA5AFC8B502A34D72518FAC964FEAA51B2FB5B08C35E
2018-02-20 19:35:49.934 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> DEBU 00a ESCC invoke result: version:1 response:<status:200 message:"OK" > payload:"\n \327\375\200\247\362]\235\331\273\331\033\302\231\017\230eF\367\306\016\360C\327f\330|)\207r'\235?\022z\n[\0228\n\017clarkschaincode\022%\n\007\n\001a\022\002\010\001\n\007\n\001b\022\002\010\001\032\007\n\001a\032\00290\032\010\n\001b\032\003210\022\037\n\004lscc\022\027\n\025\n\017clarkschaincode\022\002\010\001\032\003\010\310\001\"\026\022\017clarkschaincode\032\0031.0" endorsement:<endorser:"\n\010ClarkMSP\022\226\006-----BEGIN CERTIFICATE-----\nMIICGjCCAcCgAwIBAgIRAJuPv9SsR7z2tvnkIMs0YVswCgYIKoZIzj0EAwIwdTEL\nMAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG\ncmFuY2lzY28xGjAYBgNVBAoTEWNsYXJrLmV4YW1wbGUuY29tMR0wGwYDVQQDExRj\nYS5jbGFyay5leGFtcGxlLmNvbTAeFw0xODAyMjAxODAyNDZaFw0yODAyMTgxODAy\nNDZaMFkxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH\nEw1TYW4gRnJhbmNpc2NvMR0wGwYDVQQDExRwMC5jbGFyay5leGFtcGxlLmNvbTBZ\nMBMGByqGSM49AgEGCCqGSM49AwEHA0IABERvhKV3jE/FhhBopWq8WWon9dlDMJyr\nkcuGBlfigY7uogkjQTbbKVP6V0/fRvMRY78II1oaR3n+cdW7UL9SgpajTTBLMA4G\nA1UdDwEB/wQEAwIHgDAMBgNVHRMBAf8EAjAAMCsGA1UdIwQkMCKAIDWltx302YTW\njr/UY0TrqMJZVXLZ6Yn0U/We4Qcmv7ZhMAoGCCqGSM49BAMCA0gAMEUCIQC1N01L\n63nc4ZW3KxiOnDthTCq3JB/cV83i1AIK+0KbogIgVRfqrkkqt43fq71hOrqn1piQ\npHBK04VhaR3M8CfJEC8=\n-----END CERTIFICATE-----\n" signature:"0D\002 \025\335d\003mi\320,\205E~\263\227\t\2553?\227\304rsf\001\356\331Db\226\341\263\220c\002 f\027\220j\361W\312\274\277\033N\271\034\315\377\357\340\351JO\026\344\205.:J\322\214\257\344\312I" >
2018-02-20 19:35:49.934 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 00b Chaincode invoke successful. result: status:200
2018-02-20 19:35:49.935 UTC [main] main -> INFO 00c Exiting.....
```

Query the chaincode again to confirm the invocation was successful:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode query --channelID clarkschannel --name clarkschaincode --ctor '{"Args":["query", "a"]}'
2018-02-20 19:35:53.101 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:35:53.101 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:35:53.102 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-20 19:35:53.102 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-20 19:35:53.102 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-20 19:35:53.102 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0ABE070A7508031A0B0899F1B1D40510...636F64651A0A0A0571756572790A0161
2018-02-20 19:35:53.102 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 6F5C43A5579539BD03843CE7D1BAD97DE9F36BED8BEE15E6BC608AE72DFF4242
Query Result: 90
2018-02-20 19:35:53.118 UTC [main] main -> INFO 008 Exiting.....
```

### Add another org to the channel

Let's get the latest configuration from the channel.

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel fetch config config_1.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-02-20 19:53:45.038 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 19:53:45.039 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 19:53:45.045 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 19:53:45.045 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-02-20 19:53:45.046 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-02-20 19:53:45.046 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-02-20 19:53:45.047 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-02-20 19:53:45.047 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AE2060A1908021A0608C9F9B1D40522...C051C43C483C12080A020A0012020A00
2018-02-20 19:53:45.047 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: F4CA2A518CA61B229846F8482F385AD7D4436C495DCCF8A745D852BA3B8244A5
2018-02-20 19:53:45.057 UTC [channelCmd] readBlock -> DEBU 00a Received block: 2
2018-02-20 19:53:45.057 UTC [msp] GetLocalMSP -> DEBU 00b Returning existing local MSP
2018-02-20 19:53:45.057 UTC [msp] GetDefaultSigningIdentity -> DEBU 00c Obtaining default signing identity
2018-02-20 19:53:45.058 UTC [msp] GetLocalMSP -> DEBU 00d Returning existing local MSP
2018-02-20 19:53:45.058 UTC [msp] GetDefaultSigningIdentity -> DEBU 00e Obtaining default signing identity
2018-02-20 19:53:45.058 UTC [msp/identity] Sign -> DEBU 00f Sign: plaintext: 0AE2060A1908021A0608C9F9B1D40522...AA2B352726E712080A021A0012021A00
2018-02-20 19:53:45.058 UTC [msp/identity] Sign -> DEBU 010 Sign: digest: 7EB4D4784FB4C9996C2AA0B1C8480717E4D5D9145E133E0F77E6FD6B0B81306F
2018-02-20 19:53:45.064 UTC [channelCmd] readBlock -> DEBU 011 Received block: 0
2018-02-20 19:53:45.066 UTC [main] main -> INFO 012 Exiting.....
```

Convert into JSON:

```bash
$ docker exec -ti admin.clark.example.com bash
$ sudo apt-get update && sudo apt-get install jq -y
$ configtxlator proto_decode --input config_1.pb --type common.Block | jq .data.data[0].payload.data.config > config_1.json
```

Let's print the definition for the new org into a JSON file. Do this on your localhost, on this project's root folder (because we need access to the `crypto` folder):

```bash
$ FABRIC_CFG_PATH=./config configtxgen --printOrg Amelia > config/amelia.json
2018-02-20 20:16:18.814 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
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
2018-02-20 20:25:20.568 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 20:25:20.568 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 20:25:20.568 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 20:25:20.569 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-02-20 20:25:20.569 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-02-20 20:25:20.570 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-02-20 20:25:20.570 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-02-20 20:25:20.570 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AA7060A08436C61726B4D5350129A06...65616465727312002A0641646D696E73
2018-02-20 20:25:20.570 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: 593CFAEA40FA4380B915DF9672AB4DDCB49808D9893420BEA52FE9BB9C7EB281
2018-02-20 20:25:20.570 UTC [msp] GetLocalMSP -> DEBU 00a Returning existing local MSP
2018-02-20 20:25:20.570 UTC [msp] GetDefaultSigningIdentity -> DEBU 00b Obtaining default signing identity
2018-02-20 20:25:20.570 UTC [msp] GetLocalMSP -> DEBU 00c Returning existing local MSP
2018-02-20 20:25:20.571 UTC [msp] GetDefaultSigningIdentity -> DEBU 00d Obtaining default signing identity
2018-02-20 20:25:20.571 UTC [msp/identity] Sign -> DEBU 00e Sign: plaintext: 0AE2060A1908021A0608B088B2D40522...FB857E0FDDD81143B46485069276521F
2018-02-20 20:25:20.571 UTC [msp/identity] Sign -> DEBU 00f Sign: digest: C91B851244B10EFDA86DFB7781E4279D6CC79184174BA6CF9490479D6C4B4C6B
2018-02-20 20:25:20.572 UTC [main] main -> INFO 010 Exiting.....
```

Now send the signed envelope the ordering service:

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer channel update --file config_upd.env.pb --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-02-20 20:26:01.166 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 20:26:01.166 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 20:26:01.170 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 20:26:01.173 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-02-20 20:26:01.173 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-02-20 20:26:01.173 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-02-20 20:26:01.173 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-02-20 20:26:01.173 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AA7060A08436C61726B4D5350129A06...65616465727312002A0641646D696E73
2018-02-20 20:26:01.174 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: 42E7ED8256F8628147AB5C4E884AF6DFF12C43673E40C9DE12E5CF1C18F75066
2018-02-20 20:26:01.174 UTC [msp] GetLocalMSP -> DEBU 00a Returning existing local MSP
2018-02-20 20:26:01.174 UTC [msp] GetDefaultSigningIdentity -> DEBU 00b Obtaining default signing identity
2018-02-20 20:26:01.175 UTC [msp] GetLocalMSP -> DEBU 00c Returning existing local MSP
2018-02-20 20:26:01.175 UTC [msp] GetDefaultSigningIdentity -> DEBU 00d Obtaining default signing identity
2018-02-20 20:26:01.175 UTC [msp/identity] Sign -> DEBU 00e Sign: plaintext: 0AE2060A1908021A0608D988B2D40522...499F37BED807BB98749017D77F82FDDE
2018-02-20 20:26:01.175 UTC [msp/identity] Sign -> DEBU 00f Sign: digest: 4AD3E29BE568D5B940DAE4827FBE24A377F17579C1698E8D3B225E9139D998EB
2018-02-20 20:26:01.211 UTC [main] main -> INFO 010 Exiting.....
```

Now have the new org get the channel's genesis block:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer channel fetch 0 clarkschannel.block --channelID clarkschannel --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA
2018-02-20 20:30:05.639 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 20:30:05.639 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 20:30:05.645 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 20:30:05.646 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-02-20 20:30:05.646 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-02-20 20:30:05.647 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-02-20 20:30:05.648 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-02-20 20:30:05.648 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0AE7060A1908021A0608CD8AB2D40522...2C1DAEA9001B12080A021A0012021A00
2018-02-20 20:30:05.648 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: E7C8CDBE5408A1205E0699F343E26165356B5485209D848AFC0D826FAAC53826
2018-02-20 20:30:05.655 UTC [channelCmd] readBlock -> DEBU 00a Received block: 0
2018-02-20 20:30:05.656 UTC [main] main -> INFO 00b Exiting.....
```

And use it to join the channel:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer channel join --blockpath clarkschannel.block
2018-02-20 20:31:13.500 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-20 20:31:13.500 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-20 20:31:13.505 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-02-20 20:31:13.506 UTC [msp/identity] Sign -> DEBU 004 Sign: plaintext: 0AAA070A5C08011A0C08918BB2D40510...61E7554224BB1A080A000A000A000A00
2018-02-20 20:31:13.506 UTC [msp/identity] Sign -> DEBU 005 Sign: digest: 1EBE926FEB147E864175C7F73A539722AAFE381BA8AB98D1C0B137C6E9380B90
2018-02-20 20:31:13.607 UTC [channelCmd] executeJoin -> INFO 006 Successfully submitted proposal to join channel
2018-02-20 20:31:13.608 UTC [main] main -> INFO 007 Exiting.....
```

### Modify the chaincode

Let's modify the chaincode's endorsement policy so as to include the new org.

Let's install the chaincode on Clark's peer. Same exact bytes, just tag it with a different version

```bash
$ docker exec -ti admin.clark.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.1 --path github.com/kchristidis/fabric-example/chaincode
2018-02-21 00:44:27.553 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-21 00:44:27.553 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-21 00:44:27.553 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-21 00:44:27.553 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-21 00:44:27.554 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-21 00:44:27.606 UTC [golang-platform] getCodeFromFS -> DEBU 006 getCodeFromFS github.com/kchristidis/fabric-example/chaincode
2018-02-21 00:44:27.761 UTC [golang-platform] func1 -> DEBU 007 Discarding GOROOT package fmt
2018-02-21 00:44:27.761 UTC [golang-platform] func1 -> DEBU 008 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-02-21 00:44:27.761 UTC [golang-platform] func1 -> DEBU 009 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-02-21 00:44:27.761 UTC [golang-platform] func1 -> DEBU 00a Discarding GOROOT package strconv
2018-02-21 00:44:27.763 UTC [golang-platform] GetDeploymentPayload -> DEBU 00b done
2018-02-21 00:44:27.763 UTC [container] WriteFileToPackage -> DEBU 00c Writing file to tarball: src/github.com/kchristidis/fabric-example/chaincode/simple.go
2018-02-21 00:44:27.768 UTC [msp/identity] Sign -> DEBU 00d Sign: plaintext: 0AA5070A5C08031A0C08EB81B3D40510...9F61FD3B0000FFFFD4B49D15001A0000
2018-02-21 00:44:27.768 UTC [msp/identity] Sign -> DEBU 00e Sign: digest: DD761F4DAB0E7849070AA6B48DD16C49B059E23DCFE812B560E7A7036B3DD7AB
2018-02-21 00:44:27.778 UTC [chaincodeCmd] install -> DEBU 00f Installed remotely response:<status:200 payload:"OK" >
2018-02-21 00:44:27.778 UTC [main] main -> INFO 010 Exiting.....
```

Do the same on Amelia's peer:

```bash
$ docker exec -ti admin.amelia.example.com bash
$ peer chaincode install --name clarkschaincode --version 1.1 --path github.com/kchristidis/fabric-example/chaincode
2018-02-21 00:47:56.782 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-21 00:47:56.782 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-21 00:47:56.782 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-21 00:47:56.782 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-21 00:47:56.783 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-21 00:47:56.821 UTC [golang-platform] getCodeFromFS -> DEBU 006 getCodeFromFS github.com/kchristidis/fabric-example/chaincode
2018-02-21 00:47:56.969 UTC [golang-platform] func1 -> DEBU 007 Discarding GOROOT package fmt
2018-02-21 00:47:56.969 UTC [golang-platform] func1 -> DEBU 008 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-02-21 00:47:56.969 UTC [golang-platform] func1 -> DEBU 009 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-02-21 00:47:56.969 UTC [golang-platform] func1 -> DEBU 00a Discarding GOROOT package strconv
2018-02-21 00:47:56.971 UTC [golang-platform] GetDeploymentPayload -> DEBU 00b done
2018-02-21 00:47:56.971 UTC [container] WriteFileToPackage -> DEBU 00c Writing file to tarball: src/github.com/kchristidis/fabric-example/chaincode/simple.go
2018-02-21 00:47:56.977 UTC [msp/identity] Sign -> DEBU 00d Sign: plaintext: 0AAA070A5C08031A0C08BC83B3D40510...9F61FD3B0000FFFFD4B49D15001A0000
2018-02-21 00:47:56.977 UTC [msp/identity] Sign -> DEBU 00e Sign: digest: 54E50B9254B927E94B1D19CA10CFB960849870535656341108D3297C9CB69397
2018-02-21 00:47:56.982 UTC [chaincodeCmd] install -> DEBU 00f Installed remotely response:<status:200 payload:"OK" >
2018-02-21 00:47:56.982 UTC [main] main -> INFO 010 Exiting.....
```

Have the admin on Clark's peer upgrade the chaincode. Note that the call needs to be made by an entity satisfying the instantiation policy. Since that was not explicitly set, it defaults to any admin of any org which was present in the channel at the time of invoking the upgrade.

```bash
$ peer chaincode upgrade --orderer joe.example.com:7050 --tls --cafile $ORDERER_CA --channelID clarkschannel --name clarkschaincode --version 1.1 -c '{"Args":["init","a","100","b","200"]}' --policy "OR ('ClarkMSP.member','AmeliaMSP.member')"
2018-02-21 00:46:40.465 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-02-21 00:46:40.465 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-02-21 00:46:40.470 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-02-21 00:46:40.470 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-02-21 00:46:40.470 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode enabled
2018-02-21 00:46:40.471 UTC [chaincodeCmd] upgrade -> DEBU 006 Get upgrade proposal for chaincode <name:"clarkschaincode" version:"1.1" >
2018-02-21 00:46:40.471 UTC [msp/identity] Sign -> DEBU 007 Sign: plaintext: 0AB4070A6B08031A0C08F082B3D40510...614D53500A04657363630A0476736363
2018-02-21 00:46:40.471 UTC [msp/identity] Sign -> DEBU 008 Sign: digest: B5DD50ADEA49BEB32CC719C51DA55F0FAC760C2EEF0D5CD2CD9086AD7C7BA985
2018-02-21 00:46:52.810 UTC [chaincodeCmd] upgrade -> DEBU 009 endorse upgrade proposal, get response <status:200 message:"OK" payload:"\n\017clarkschaincode\022\0031.1\032\004escc\"\004vscc*+\022\014\022\n\010\001\022\002\010\000\022\002\010\001\032\014\022\n\n\010ClarkMSP\032\r\022\013\n\tAmeliaMSP2D\n \322\302\306\002\007\337\000\025f\364\347mT\374\342\273\321A\211\316\003\346\324\342\005\005\3204\257Z\034\252\022 \275\352\345\033s\255\367\302_PO\202\340o3\026?]\224\230\303\2055T;\372\001w\234}\374\271: !\321\t\315\353z\341\356$\311f\216\333\r\013P<\333\001\014\355\321\004SI\240u}\032+\232IB/\022\014\022\n\010\001\022\002\010\000\022\002\010\001\032\017\022\r\n\tAmeliaMSP\020\001\032\016\022\014\n\010ClarkMSP\020\001" >
2018-02-21 00:46:52.817 UTC [msp/identity] Sign -> DEBU 00a Sign: plaintext: 0AB4070A6B08031A0C08F082B3D40510...BC0E5E5B3DAA7B978A9649BFC4B944F9
2018-02-21 00:46:52.819 UTC [msp/identity] Sign -> DEBU 00b Sign: digest: 16A2B53A03C46142FAD872F3FE67DED478940A90AD6BAF7532F6AE02ABEC5960
2018-02-21 00:46:52.820 UTC [chaincodeCmd] upgrade -> DEBU 00c Get Signed envelope
2018-02-21 00:46:52.820 UTC [chaincodeCmd] chaincodeUpgrade -> DEBU 00d Send signed envelope to orderer
2018-02-21 00:46:52.824 UTC [main] main -> INFO 00e Exiting.....
```