cd ~/fabric/fabric-samples/test-network

./network.sh up createChannel -ca -c mychannel

# composer tools steps (if installed)
yo hyperledger-composer

composer archive create --sourceType dir --sourceName .

composer network install --card PeerAdmin@hlfv1 --archiveFile supply-chain-network@0.0.1.bna

composer network start --networkName supply-chain-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file supply-admin.card

composer card import --file supply-admin.card

composer network ping --card admin@supply-chain-network

cd ~/fabric/fabric-samples/asset-transfer-basic/application-javascript

npm install

node app.js         # runs sample app to create and transfer assets
