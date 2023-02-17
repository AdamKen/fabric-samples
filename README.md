# Fabric-Samples

## 一、组网
假设山东银行和北京银行希望组建一个跨银行结算业务联盟链。为了实现公平、自治，他们的组织和物理节点关系如下：
| 组织类型 | 组织名称            |  节点数量  |
| ------  | ------------------  | ----------|
| 排序    | OrdererBeijingBank  | 2         |     
| 排序    | OrdererShandongBank | 2         |
| 背书    | ShandongBank        | 2         |
| 背书    | BeijingBank         | 2         |

基于这个方案，双方的技术人员开始着手组网的具体工作。  
首先双方需要生成各自的证书，在组网的测试阶段，他们决定使用fabric团队提供的cryptogen生成证书。使用的配置文件如下：
```yaml
OrdererOrgs:
  - Name: OrdererShandongBank
    Domain: shandongbank
    CA:
      Country: CN
      Province: Shandong
      Locality: Jinan
      StreetAddress: Huaneng
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer0
      - Hostname: orderer1
  - Name: OrdererBeijingBank
    Domain: beijingbank
    CA:
      Country: CN
      Province: Peking
      Locality: Xicheng
      StreetAddress: Jinrong
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer0
      - Hostname: orderer1
PeerOrgs:
  - Name: ShandongBank
    Domain: shandongbank
    CA:
      Country: CN
      Province: Shandong
      Locality: Jinan
      StreetAddress: Huaneng
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: BeijingBank
    Domain: beijingbank
    CA:
      Country: CN
      Province: Peking
      Locality: Xicheng
      StreetAddress: Jinrong
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```
注意，这个配置文件会生成两家银行的所有证书，问题是这样会把各自的私钥也暴露给对方，那么组建起来的联盟链也就毫无隐私可言。所以正常情况下双方的配置文件只会包含自家组织的配置，然后交换所需的公钥证书。当然如果只是用来个人测试，使用上面的配置会比较方便。下面使用cryptogen生成证书：
```shell
 ./bin/cryptogen generate --config=./crypto-config.yaml
 #这条命令会将证书生成到crypto-config目录。  
```
基于设定方案和生成的证书，双方管理员下一步要做的就是协商制定组网的底层规则，比如参与组网的组织、背书节点、背书策略、通道权限等。这些规则会被写到configtx.yaml中，并被用来生成将来联盟链网络的创世区块和业务通道所需的配置交易。下面的代码包含了创世区块所需的总体配置。
```yaml
SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            EtcdRaft:
                Consenters:
                - Host: orderer0.beijingbank
                  Port: 8080
                  ClientTLSCert: crypto-config/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/tls/server.crt
                - Host: orderer1.beijingbank
                  Port: 8090
                  ClientTLSCert: crypto-config/ordererOrganizations/beijingbank/orderers/orderer1.beijingbank/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/beijingbank/orderers/orderer1.beijingbank/tls/server.crt
                - Host: orderer0.shandongbank
                  Port: 7080
                  ClientTLSCert: crypto-config/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/tls/server.crt
                - Host: orderer1.shandongbank
                  Port: 7090
                  ClientTLSCert: crypto-config/ordererOrganizations/shandongbank/orderers/orderer1.shandongbank/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/shandongbank/orderers/orderer1.shandongbank/tls/server.crt
            Addresses:
                - orderer0.beijingbank:8080
                - orderer1.beijingbank:8090
                - orderer0.shandongbank:7080
                - orderer1.shandongbank:7090
            Organizations:
            - *OrdererShandongBank
            - *OrdererBeijingBank
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
            - <<: *OrdererBeijingBank
            - <<: *OrdererShandongBank
        Consortiums:
            SampleConsortium:
                Organizations:
                - *BeijingBank
                - *ShandongBank
```
假设双方管理员对这个配置文件达成一致，下一步要做的就是生成联盟链的创世区块和业务通道的相关交易：
```shell
#生成创世区块
./bin/configtxgen -profile SampleMultiNodeEtcdRaft -channelID sys-channel -outputBlock ./channel-artifacts/genesis.block 
#生成业务通道配置交易
./bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID testchannel 
#生成ShandongBank的锚节点配置交易
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/ShandongBankMSPanchors.tx -channelID testchannel -asOrg ShandongBankMSP 
#生成BeijingBank的锚节点配置交易
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/BeijingBankMSPanchors.tx -channelID testchannel -asOrg BeijingBankMSP 
```

至此，联盟链的初始数据已经准备完毕，下面要启动各个节点，将区块链网络运行起来。  
fabric提供了各类节点的镜像供使用者部署，基于此，使用者只需要注意三个部分（挂载、网络，和环境变量）就可以轻松运行起一个联盟链网络。下面是其中一个节点的docker-compose配置,它列出了可能需要变更的配置项，其他配置项被放在了perr-base.yaml中。另外，管理员可能想启动一个客户端（cli）容器以便行使组织管理员权限。
```yaml
  peer0.shandongbank:
    container_name: peer0.shandongbank
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.shandongbank
      - CORE_PEER_ADDRESS=peer0.shandongbank:7060
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7060
      - CORE_PEER_CHAINCODEADDRESS=peer0.shandongbank:7061
      # - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      # - CORE_CHAINCODE_LOGGING_SHIM=DEBUG
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7061
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.shandongbank:7070
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.shandongbank:7060
      - CORE_PEER_LOCALMSPID=ShandongBankMSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/shandongbank/peers/peer0.shandongbank/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/shandongbank/peers/peer0.shandongbank/tls:/etc/hyperledger/fabric/tls
    ports:
      - 7060:7060
      - 7061:7061
#  peer0.shandongbank 的客户端，使用Admin权限运行
cli-sdb-peer0:
    container_name: cli-sdb-peer0
    image: hyperledger/fabric-tools:1.4.6
    tty: true
    stdin_open: true
    environment:
      - SYS_CHANNEL=sys-channel
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.shandongbank:7060
      - CORE_PEER_LOCALMSPID=ShandongBankMSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/shandongbank/peers/peer0.shandongbank/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/shandongbank/peers/peer0.shandongbank/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/shandongbank/peers/peer0.shandongbank/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/shandongbank/users/Admin@shandongbank/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ../chaincode/:/opt/gopath/src/github.com/chaincode
        - ../crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ../scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
        - ../channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
```
这里使用docker-compose来部署区块链节点，当然也可以用docker、k8s等任何支持容器化应用的方式部署,不同之处在于所需的配置格式。对于目前的两个银行，可以是一方host docker，另一方k8s，也可以是双方一致，这取决于各自的需求。  
总之当管理员写好配置文件后（比如docker-composr.yaml），就可以启动节点了：
```shell
docker compose -f docker-compose.yaml up -d
```
节点启动后，首先开始通信的是排序（orderer）节点。观察他们的日志，排序节点间使用的是ectd-raft协议，因此会不断打印选举日志，直到选出一个领导节点。
peer节点目前没有处于活动状态，是因为它们还没有加入业务通道，不知道其他节点的存在。所以下一步要创建一个业务通道，并把peer加进去。涉及通道变更的操作需要管理员权限，一般会通过客户端搭配管理员证书来实现：
```shell
docker exec -it cli-sdb-peer0 bash
#创建通道
peer channel create -o orderer0.shandongbank:7080  -c testchannel -f $(pwd)/channel-artifacts/channel.tx --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
#加入通道
peer channel join -b testchannel.block
#更新锚节点
peer channel update -o orderer0.shandongbank:7080 -c testchannel -f ./channel-artifacts/ShandongBankMSPanchors.tx --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
exit
```
上面的peer channel create命令会在本地生成一个testchannel.block区块，双方的任何一个peer节点都可以通过这个区块加入这个通道，如果双方不方便共享这个文件，其他的peer也可以通过获取0号区块达到同样的效果：
```shell
peer channel fetch 0 testchannel.block -o orderer0.shandongbank:7080 -c testchannel --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
```
peer加入通道后，会通过gossip协议查找其他peer,在日志中打印的Membership view记录了这个过程。  
至此完成了组网的全部过程。  
尽管链码的开发一般属于业务方链码开发人员的工作，但其部署工作可能还是由底层链提供方代劳，所以下面管理员还要将链码安装、实例化到peer上。
从定义上来说，peer节点只有安装了链码才能被称为背书节点；从容灾的角度，既然双方都只有两个peer节点，那么最好在两个peer上都安装链码：
```shell
docker exec -it cli-sdb-peer0 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
docker exec -it cli-sdb-peer1 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc

docker exec -it cli-bjb-peer0 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
docker exec -it cli-bjb-peer1 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
```
按照Fabric的逻辑，参与背书的各个组织都至少有一个peer安装链码，最后由其中一个peer执行实例化操作：
```shell
#instantiate chaincode
docker exec -it cli-sdb-peer0 bash
peer chaincode instantiate -o orderer0.shandongbank:7080 --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem -C testchannel -n tcc1 -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('ShandongBankMSP.peer','BeijingBankMSP.peer')"
exit
```
现在业务方可以自由调用这个链码了。管理员也许还要进行一些运维的工作，但整体上的工作已经完成了。



```shell
docker compose -f docker-compose.yaml down --volumes --remove-orphans


./bin/cryptogen generate --config=./crypto-config.yaml
./bin/configtxgen -profile SampleMultiNodeEtcdRaft -channelID sys-channel -outputBlock ./channel-artifacts/genesis.block
./bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID testchannel
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/ShandongBankMSPanchors.tx -channelID testchannel -asOrg ShandongBankMSP
./bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/BeijingBankMSPanchors.tx -channelID testchannel -asOrg BeijingBankMSP



docker exec -it cli-sdb-peer0 bash
peer channel create -o orderer0.shandongbank:7080  -c testchannel -f $(pwd)/channel-artifacts/channel.tx --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
peer channel join -b testchannel.block
peer channel update -o orderer0.shandongbank:7080 -c testchannel -f ./channel-artifacts/ShandongBankMSPanchors.tx --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
exit

docker exec -it cli-sdb-peer1 bash
peer channel fetch 0 testchannel.block -o orderer0.shandongbank:7080 -c testchannel --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
peer channel join -b testchannel.block
exit

docker exec -it cli-bjb-peer0 bash
peer channel fetch 0 testchannel.block -o orderer0.beijingbank:8080 -c testchannel --tls --cafile $(pwd)/crypto/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/msp/tlscacerts/tlsca.beijingbank-cert.pem
peer channel join -b testchannel.block
peer channel update -o orderer0.beijingbank:8080 -c testchannel -f ./channel-artifacts/BeijingBankMSPanchors.tx --tls --cafile $(pwd)/crypto/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/msp/tlscacerts/tlsca.beijingbank-cert.pem
exit

docker exec -it cli-bjb-peer1 bash
peer channel fetch 0 testchannel.block -o orderer0.beijingbank:8080 -c testchannel --tls --cafile $(pwd)/crypto/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/msp/tlscacerts/tlsca.beijingbank-cert.pem
peer channel join -b testchannel.block
exit


#installl chaincode 
docker exec -it cli-sdb-peer0 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
docker exec -it cli-sdb-peer1 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
docker exec -it cli-bjb-peer0 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
docker exec -it cli-bjb-peer1 peer chaincode install -n tcc1 -v 1.0 -l golang -p github.com/chaincode/tcc
#instantiate chaincode
docker exec -it cli-sdb-peer0 -- pwd
peer chaincode instantiate -o orderer0.shandongbank:7080 --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem -C testchannel -n tcc1 -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('ShandongBankMSP.peer','BeijingBankMSP.peer')"
exit


```
