# Matter-Chain network configuration

## Nodes
The network consists of 3 virtual machines:
- node1.matter-chain.com
- node2.matter-chain.com
- node3.matter-chain.com

### node1.matter-chain.com
This is the "main" node of the network and hosts the following fabric components:
- CA - **ca.prod.matter-chain.com**
- Orderer - **orderer1.matter-chain.com**
- Peer - **peer1.prod.matter-chain.com**

### node2.matter-chain.com
This is the first replication node and hosts the following fabric components:
- Orderer - **orderer2.matter-chain.com**
- Peer - **peer2.matter-chain.com**

### node3.matter-chain.com
This is the second replication node and hosts the following fabric components:
- Orderer **orderer3.matter-chain.com**
- Peer **peer3.matter-chain.com**

## Channels
The network uses just one channel **ProdChannel** so data is replicated in this channel.

## Organizations

### Orderer

All orderers belong to the matter-chain organisation.

### Peer

Peers running on nodes (node1, node2, node2) all belong to the same organization (ProdOrg). This organization is used to host the main chain data (Tenants, Devices, Rooms, Token definitions, ...).

Each tenant (IoT device) will have it's own organization that will be joined to the ProdChannel. There will be most commonly only one peer node per this organization. Each tenant will have their own organization, to they will be able to use PDC (Private Data Collections). The hash will still get stored on the chain and could be verified, but the actual data will be only accessible to the specific IoT organization.

Example of IoT peer:
- **peer1.tenant-X.matter-chain.com**


# Setup guide

## CA

Base path: /root/chain-ca
```
mkdir chain-ca
mkdir chain-ca/config
```

Create certificate
User openssl-san file:
```
containers->node1->ca->openssl-san.cnf
```
```
openssl ec -in config/ca-key.pem -noout -text
openssl req -new -x509 -key config/ca-key.pem   -out config/ca-cert.pem   -days 3650   -config openssl-san.cnf   -extensions v3_ca

```

Use docker compose file
```
containers->node1->ca->docker-compose.yaml
```

Start the container and check the logs
```
docker-compose up
docker logs ca-prod
```
## Node1

### Copy main CA file

```
cd /root/chain-ca
mkdir -p /root/matter-chain/organizations/ordererOrganizations/matter-chain.com
docker cp ca-prod:/etc/hyperledger/fabric-ca-server-config/ca-cert.pem /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/tls-cert.pem
```
### Enroll CA admin
```
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://ca-admin:ca-adminpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
### Create folder structure for chain
```
mkdir /root/matter-chain # main folder for containers, certificates and artifacts
mkdir /root/matter-chain/system-genesis-block # folder which stores generated genesis block
mkdir /root/channel-artifacts # folder which stores channel artifacts (create channel, ...)
```

In the main folder (/root/matter-chain) create the configtx.yaml file
```
nano configtx.yaml
(config->configtx.yaml)
``` 
### Register orderers
```
# orderer1
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client register \
    --id.name orderer1.matter-chain.com \
    --id.secret ordererpw \
    --id.type orderer \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```

```
# orderer2
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client register \
    --id.name orderer2.matter-chain.com \
    --id.secret ordererpw \
    --id.type orderer \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```

```
# orderer3
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client register \
    --id.name orderer3.matter-chain.com \
    --id.secret ordererpw \
    --id.type orderer \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
### Register & Enroll orderer admin
```
# create folder structure and copy root cert
mkdir $(pwd)/organizations/ordererOrganizations/matter-chain.com/users/Admin@matter-chain.com
cp /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/tls-cert.pem /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/users/Admin@matter-chain.com

# register admin
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client register \
    --id.name orderer-admin \
    --id.secret ordereradminpw \
    --id.type admin \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem

# enroll admin
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/users/Admin@matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer-admin:ordereradminpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
### Enroll orderers
```
# create folder structure and copy root cert
mkdir /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com
mkdir /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com
mkdir /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com
cp /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/tls-cert.pem /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com
cp /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/tls-cert.pem /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com
cp /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/tls-cert.pem /root/matter-chain/organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com
```

```
# enroll orderer1
# MSP
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer1.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --csr.cn orderer1.matter-chain.com \
    --csr.hosts orderer1.matter-chain.com \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer1.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --enrollment.profile tls \
    --csr.cn orderer1.matter-chain.com \
    --csr.hosts orderer1.matter-chain.com \
    -M /etc/hyperledger/fabric-ca-client/tls \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```

```
# enroll orderer2
# MSP
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer2.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --csr.cn orderer2.matter-chain.com \
    --csr.hosts orderer2.matter-chain.com \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer2.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --enrollment.profile tls \
    --csr.cn orderer2.matter-chain.com \
    --csr.hosts orderer2.matter-chain.com \
    -M /etc/hyperledger/fabric-ca-client/tls \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```

```
# enroll orderer3
# MSP
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer3.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --csr.cn orderer3.matter-chain.com \
    --csr.hosts orderer3.matter-chain.com \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://orderer3.matter-chain.com:ordererpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --enrollment.profile tls \
    --csr.cn orderer3.matter-chain.com \
    --csr.hosts orderer3.matter-chain.com \
    -M /etc/hyperledger/fabric-ca-client/tls \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
Create NodeOUs file
```
nano organizations/ordererOrganizations/matter-chain.com/msp/config.yaml

(config->config.yaml)
```
Copy file to orderer MSP folders and admin MSP folder
```
cp organizations/ordererOrganizations/matter-chain.com/msp/config.yaml organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com/msp
cp organizations/ordererOrganizations/matter-chain.com/msp/config.yaml organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com/msp
cp organizations/ordererOrganizations/matter-chain.com/msp/config.yaml organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com/msp

cp organizations/ordererOrganizations/matter-chain.com/msp/config.yaml organizations/ordererOrganizations/matter-chain.com/users/Admin@matter-chain.com/msp/
```

Copy TLS certificates
```
export ORDERER_TLS=organizations/ordererOrganizations/matter-chain.com/orderers/orderer1.matter-chain.com/tls
cp $ORDERER_TLS/tlscacerts/* $ORDERER_TLS/ca.crt
cp $ORDERER_TLS/signcerts/* $ORDERER_TLS/server.crt
cp $ORDERER_TLS/keystore/*  $ORDERER_TLS/server.key

export ORDERER_TLS=organizations/ordererOrganizations/matter-chain.com/orderers/orderer2.matter-chain.com/tls
cp $ORDERER_TLS/tlscacerts/* $ORDERER_TLS/ca.crt
cp $ORDERER_TLS/signcerts/* $ORDERER_TLS/server.crt
cp $ORDERER_TLS/keystore/*  $ORDERER_TLS/server.key

export ORDERER_TLS=organizations/ordererOrganizations/matter-chain.com/orderers/orderer3.matter-chain.com/tls
cp $ORDERER_TLS/tlscacerts/* $ORDERER_TLS/ca.crt
cp $ORDERER_TLS/signcerts/* $ORDERER_TLS/server.crt
cp $ORDERER_TLS/keystore/*  $ORDERER_TLS/server.key
```
### Enroll peers
Create folder structure for peers
```
mkdir -p organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com
mkdir -p organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com
mkdir -p organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com
```
Copy the root certificate to peer folder:
```
docker cp ca-prod:/etc/hyperledger/fabric-ca-server-config/ca-cert.pem /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/tls-cert.pem
```

Enroll CA for peer ORG
```
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://ca-admin:ca-adminpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
Copy main CA to admin folder:
```
mkdir -p $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/users/Admin@prod.matter-chain.com
cp /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/tls-cert.pem /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/users/Admin@prod.matter-chain.com
```

Register and enroll peer admin
```
# register
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client register \
    --id.name peer-admin \
    --id.secret peeradminpw \
    --id.type admin \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem

# enroll
docker run --rm \
  --add-host ca.prod.matter-chain.com:172.17.0.1 \
  -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
  -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/users/Admin@prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
  hyperledger/fabric-ca:1.5 \
  fabric-ca-client enroll \
    -u https://peer-admin:peeradminpw@ca.prod.matter-chain.com:7054 \
    --caname ca-prod \
    --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```

Register peers

```
# peer1
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client register \
      --id.name peer1 \
      --id.secret peerpw \
      --id.type peer \
      --caname ca-prod \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
```
# peer2
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client register \
      --id.name peer2 \
      --id.secret peerpw \
      --id.type peer \
      --caname ca-prod \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
```
# peer3
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client register \
      --id.name peer3 \
      --id.secret peerpw \
      --id.type peer \
      --caname ca-prod \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
Copy root cert to peer folders
```
cp /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/tls-cert.pem /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com
cp /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/tls-cert.pem /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com
cp /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/tls-cert.pem /root/matter-chain/organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com
```
Enroll peers
```
# peer1
# MSP
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer1:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --csr.hosts peer1.prod.matter-chain.com \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer1:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --enrollment.profile tls \
      --csr.hosts peer1.prod.matter-chain.com \
      -M /etc/hyperledger/fabric-ca-client/tls \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
```
# peer2
# MSP
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer2:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --csr.hosts peer2.prod.matter-chain.com \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer2:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --enrollment.profile tls \
      --csr.hosts peer2.prod.matter-chain.com \
      -M /etc/hyperledger/fabric-ca-client/tls \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
```
# peer3
# MSP
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer3:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --csr.hosts peer3.prod.matter-chain.com \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
# TLS
docker run --rm \
    --add-host ca.prod.matter-chain.com:172.17.0.1 \
    -e FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric-ca-client \
    -v $(pwd)/organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com:/etc/hyperledger/fabric-ca-client \
    hyperledger/fabric-ca:1.5 \
    fabric-ca-client enroll \
      -u https://peer3:peerpw@ca.prod.matter-chain.com:7054 \
      --caname ca-prod \
      --enrollment.profile tls \
      --csr.hosts peer3.prod.matter-chain.com \
      -M /etc/hyperledger/fabric-ca-client/tls \
      --tls.certfiles /etc/hyperledger/fabric-ca-client/tls-cert.pem
```
Create config file for NodeOUs
```
nano organizations/peerOrganizations/prod.matter-chain.com/msp/config.yaml
(config->config.yaml)
```
Copy the file to peer and admin MSP folders
```
cp organizations/peerOrganizations/prod.matter-chain.com/msp/config.yaml organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com/msp/
cp organizations/peerOrganizations/prod.matter-chain.com/msp/config.yaml organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com/msp/
cp organizations/peerOrganizations/prod.matter-chain.com/msp/config.yaml organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com/msp/
cp organizations/peerOrganizations/prod.matter-chain.com/msp/config.yaml organizations/peerOrganizations/prod.matter-chain.com/users/Admin@prod.matter-chain.com/msp/
```
Copy generated certificates
```
export PEER_TLS=organizations/peerOrganizations/prod.matter-chain.com/peers/peer1.prod.matter-chain.com/tls
cp $PEER_TLS/tlscacerts/* $PEER_TLS/ca.crt
cp $PEER_TLS/signcerts/*  $PEER_TLS/server.crt
cp $PEER_TLS/keystore/*   $PEER_TLS/server.key

export PEER_TLS=organizations/peerOrganizations/prod.matter-chain.com/peers/peer2.prod.matter-chain.com/tls
cp $PEER_TLS/tlscacerts/* $PEER_TLS/ca.crt
cp $PEER_TLS/signcerts/*  $PEER_TLS/server.crt
cp $PEER_TLS/keystore/*   $PEER_TLS/server.key

export PEER_TLS=organizations/peerOrganizations/prod.matter-chain.com/peers/peer3.prod.matter-chain.com/tls
cp $PEER_TLS/tlscacerts/* $PEER_TLS/ca.crt
cp $PEER_TLS/signcerts/*  $PEER_TLS/server.crt
cp $PEER_TLS/keystore/*   $PEER_TLS/server.key
```

### Create genesis block
Genesis block is hosted in the system-genesis-block folder

```
docker run --rm   -v $(pwd):/etc/hyperledger/fabric   -e FABRIC_CFG_PATH=/etc/hyperledger/fabric   hyperledger/fabric-tools:2.5   configtxgen     -profile OrdererGenesis     -channelID system-channel     -outputBlock /etc/hyperledger/fabric/system-genesis-block/genesis.block
```
### Copy certificates and genesis to other nodes

```
# node2
scp -P 1023 -r ~/matter-chain/organizations ~/matter-chain/system-genesis-block root@orderer2.matter-chain.com:/root/matter-chain/
```

```
# node3
scp -P 1023 -r ~/matter-chain/organizations ~/matter-chain/system-genesis-block fabric@orderer3.matter-chain.com:/home/fabric/matter-chain/
```

### Start containers and check logs
```
docker-compose up -d
docker logs -f orderer1.matter-chain.com
```

## Node2

## Node3

## Creating the channel
In order to create the channel, we need to create a transaction artifact:

```
export FABRIC_CFG_PATH=$PWD
configtxgen -profile ProdChannel   -outputCreateChannelTx ./channel-artifacts/prod-channel.tx   -channelID prod-channel
```

Start a new shell inside the peer container
```
docker exec -it peer1.prod.matter-chain.com bash
```
Create new channel
```
peer channel create \
  -o orderer1.matter-chain.com:7050 \
  --ordererTLSHostnameOverride orderer1.matter-chain.com \
  -c prod-channel \
  -f /var/hyperledger/channel-artifacts/prod-channel.tx \
  --outputBlock /var/hyperledger/channel-artifacts/prod-channel.block \
  --tls \
  --cafile /var/hyperledger/orderer/tls/ca.crt
```

