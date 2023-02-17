# Fabric-Samples


``` shell
./bin/cryptogen generate --config=./crypto-config.yaml
./bin/configtxgen -profile SampleMultiNodeEtcdRaft -channelID sys-channel -outputBlock ./channel-artifacts/genesis.block
./bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID testchannel
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/ShandongBankMSPanchors.tx -channelID testchannel -asOrg ShandongBankMSP
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/BeijingBankMSPanchors.tx -channelID testchannel -asOrg BeijingBankMSP

peer channel create -o orderer0.shandongbank:7080  -c testchannel -f ./channel-artifacts/channel.tx --tls --cafile ./crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem

peer channel join -b 
```
