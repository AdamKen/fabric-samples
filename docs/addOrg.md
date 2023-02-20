## 添加组织
在之前的场景中，山东银行和北京银行组建了一条联盟链并投入使用。现在有一家新的银行——河西银行慕名而来，准备加入这个跨行结算联盟链。  
按照平等共治的原则，河西银行希望和之前的银行一样引入两个组织四个节点：
| 组织类型 | 组织名称            |  节点数量  |
| ------  | ------------------  | ----------|
| 排序    | OrdererHexiBank    | 2         |     
| 背书    | HexiBank           | 2         |

很遗憾的是，排序组织和背书组织的新增过程并不一样。原因在于排序节点的更改在系统通道层级，而背书节点的更改一般只涉及应用通道。另一个区别是，排序节点只允许逐个添加，而背书节点没有这个限制。
### 添加OrdererMSP
要添加排序组织，必须更新系统通道和现有的应用通道。因为新建业务通道时，会拉取系统通道的配置，其中就包括排序组织的信息。
下面以更新系统通道为例，说明如何向通道添加排序组织（和节点）。注意，因为ETCD-RAFT的限制，一次更改只能添加一个节点。
#### 生成证书
新组织的管理员写好配置文件使用cryptogen生成证书：
```yaml
OrdererOrgs:
  - Name: OrdererHexiBank
    Domain: hexibank
    CA:
      Country: CN
      Province: Hexi
      Locality: Jixi
      StreetAddress: Jiefang
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer0
      - Hostname: orderer1
PeerOrgs:
  - Name: HexiBank
    Domain: hexibank
    CA:
      Country: CN
      Province: Hexi
      Locality: Jixi
      StreetAddress: Jiefang
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```
```shell
cd newOrg
../bin/cryptogen generate --config=crypto-config.yaml
```
#### 编写组织配置文件
新组织管理员需要和联盟内组织管理员协商制定组织规则，保存到configtx.yaml。但是内容更简短：
```yaml
Organizations:
    - &OrdererHexiBank
        Name: OrdererHexiBankMSP
        ID: OrdererHexiBankMSP
        MSPDir: crypto-config/ordererOrganizations/hexibank/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('OrdererHexiBankMSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('OrdererHexiBankMSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('OrdererHexiBankMSP.admin')"
    - &HexiBank
        Name: HexiBankMSP
        ID: HexiBankMSP
        MSPDir: crypto-config/peerOrganizations/hexibank/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('HexiBankMSP.admin', 'HexiBankMSP.peer', 'HexiBankMSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('HexiBankMSP.admin', 'HexiBankMSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('HexiBankMSP.admin')"
        AnchorPeers:
            - Host: peer0.hexibank
              Port: 9060
```
#### 更新通道配置
这一步需要从链上获取最新配置区块，并将新组织、节点信息添加进去，最后将新配置通过交易的形式提交，达到更新通道的目的。  
首先，根据新组织的配置文件生成通用的json文件，方便后面使用：
```shell
../bin/configtxgen -printOrg OrdererHexiBankMSP > ../channel-artifacts/ordererhexibank.json
```
然后联盟内管理员拉取最新配置区块到本地：
```shell
docker exec -it cli-sdb-peer0 bash
export CORE_PEER_LOCALMSPID=OrdererShandongBankMSP
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/shandongbank/users/Admin@shandongbank/msp/
#此处没有指定区块序号，而是用config自动获取最新配置区块
#-c使用的通道名称是系统通道的名字sys-channel，这是联盟链创建时指定的。
peer channel fetch config ./channel-artifacts/config.block -o orderer0.shandongbank:7080 -c sys-channel --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem
exit 
```
现在需要将配置区块转为json形式以方便编辑：
```shell
../bin/configtxlator proto_decode --input config.block --type common.Block | jq .data.data[0].payload.data.config > config.json
```
现在可以将之前生成的ordererhexibank.json添加到config.json中了。  
- config.json > channel_group > groups> Orderer> groups和config.json > channel_group > groups > Application > groups 中可以看到联盟内已有的两个 orderer组织 OrdererBeijingBank和 OrdererShandongBank，管理员要做的就是添加一条键为 OrdererHexiBank，值为 ordererhexibank.json中内容的记录。这样就把Orderer组织加入到了配置。    
- config.json>channel_group>groups>Orderer>values>ConsensusType>value>metadata>consenters中定义了共识节点，也就是orderer，管理员可以参照已经存在的节点记录添加新节点的记录，其中的client_tls_cert和server_tls_cert一般是节点证书路径下的tls/server.crt的base64编码。  
- config.json>channel_group>values>OrdererAddresses>value>addresses添加相应的orderer节点记录。  
再次强调下，管理员可能想相同时添加两个orderer节点，但是这会导致配置更新失败，因为etcd-raft决定了一次只能添加一个节点。  
编辑完成后，将文件另存为m_config.json, 区别于原来的config.json。 然后将这两个json转回protobuf 格式：
```shell
../bin/configtxlator proto_encode --input config.json --type common.Config --output config.pb
../bin/configtxlator proto_encode --input m_config.json --type common.Config --output m_config.pb
```
现在将两者对比，得到变更部分：
```shell
../bin/configtxlator compute_update --channel_id sys-channel --original config.pb --updated m_config.pb --output ordererhexibank_update.pb
```
然后将变更部分转为json形式，以方便添加交易头信息：
```shell
../bin/configtxlator proto_decode --input ordererhexibank_update.pb --type common.ConfigUpdate > ordererhexibank_update.json
#添加交易头
echo '{"payload":{"header":{"channel_header":{"channel_id":"sys-channel", "type":2}},"data":{"config_update":'$(cat ordererhexibank_update.json)'}}}' | jq . > ordererhexibank_update_envelope.json
```
将交易json转为protobuf格式：
```shell
../bin/configtxlator proto_encode --input ordererhexibank_update_envelope.json --type common.Envelope > ordererhexibank_update_envelope.pb
```
最后由联盟内双方管理员签名并更新配置到联盟链。由于更新操作实际上包含了签名动作，所以实际上是一方执行签名操作，然后另一方执行更新操作。
```shell
docker exec -it cli-sdb-peer0 bash
#使用OrdererShandongbankMSP 的管理员身份执行签名操作
export CORE_PEER_LOCALMSPID=OrdererShandongBankMSP
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/shandongbank/users/Admin@shandongbank/msp/
peer channel signconfigtx -f channel-artifacts/ordererhexibank_update_envelope.pb
exit

docker exec -it cli-bjb-peer0 bash 
#使用OrdererBeijingBankMSP 的管理员身份执行更新操作
export CORE_PEER_LOCALMSPID=OrdererBeijingBankMSP
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/beijingbank/users/Admin@beijingbank/msp/
#ordererhexibank_update_envelope.pb应该是上个组织签名后的文件
peer channel update -f channel-artifacts/ordererhexibank_update_envelope.pb -c sys-channel -o orderer0.beijingbank:8080 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/beijingbank/orderers/orderer0.beijingbank/msp/tlscacerts/tlsca.beijingbank-cert.pem
exit
```
至此，已经成功向系统通道添加了一个排序组织OrdererHexiBankMSP，以及一个节点。启动这个节点，等待一段时间，即可看到这个节点被纳入到raft集群中。  
下一步管理员需要将另一个orderer节点添加到系统通道。完成后，即完成了系统通道的更新，但是要想将这个组织纳入到正在运行的业务通道，管理员需要按上面的步骤再更新一次业务通道，唯一的区别是把系统通道的名称（这里是sys-channel）更改为待更新的业务通道名称（如本例中的testchannel）。
