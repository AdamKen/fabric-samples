## 添加组织
在之前的场景中，山东银行和北京银行组建了一条联盟链并投入使用。现在有一家新的银行——河西银行慕名而来，准备加入这个跨行结算联盟链。  
按照平等共治的原则，河西银行希望和之前的银行一样引入两个组织四个节点：
| 组织类型 | 组织名称            |  节点数量  |
| ------  | ------------------  | ----------|
| 排序    | OrdererHexiBank    | 2         |     
| 背书    | HexiBank           | 2         |

很遗憾的是，排序组织和背书组织的新增过程并不一样。原因在于排序节点的更改在系统通道层级，而背书节点的更改一般只涉及应用通道。另一个细节是，排序节点只允许逐个添加，而背书节点没有这个限制。
### 添加OrdererMSP
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
./bin/cryptogen generate --config=./newOrg/crypto-config.yaml
```
注意，虽然这里一次性生成了所有需要的节点证书，但是下面的操作要分开操作。
#### 编写组织配置文件
新组织管理员需要和联盟内组织管理员协商制定组织规则，保存到configtx.yaml。但是内容更简短：
```yaml

```
#### 更新通道配置
这一步需要从链上获取最新配置区块，并将新组织、节点信息添加进去，最后将新配置通过交易的形式提交，达到更新通道的目的。
```shell
peer channel fetch config config.block -o orderer0.shandongbank:7080 -c testchannel --tls --cafile $(pwd)/crypto/ordererOrganizations/shandongbank/orderers/orderer0.shandongbank/msp/tlscacerts/tlsca.shandongbank-cert.pem

```
#### 加入通道
