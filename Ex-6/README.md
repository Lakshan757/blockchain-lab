node -v     # Should be ≥ 14.x

npm -v      # Should be ≥ 6.x

docker --version          # Should be ≥ 20.x

docker-compose --version  # Should be ≥ 1.29.x

git --version

curl --version

2. Start the Fabric Test Network

cd fabric-samples/test-network

./network.sh up createChannel -ca -c mychannel


This starts the Hyperledger Fabric test network with peers, orderers, and a default channel named mychannel.

3. Package and Deploy Chaincode

Deploy the Car Auction Chaincode:

./network.sh deployCC -ccn carauction -ccp ../asset-transfer-basic/chaincode-javascript -ccl javascript


This installs and instantiates the chaincode named carauction on the blockchain.

4. Test the Chaincode via Peer CLI

Invoke the chaincode to register a car on the blockchain:

peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
--tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlsca/tlsca.example.com-cert.pem \
-C mychannel -n carauction \
--peerAddresses localhost:7051 \
--tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
--peerAddresses localhost:9051 \
--tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt \
-c '{"function":"CreateCar","Args":["CAR10","Toyota","Camry","Red","Lucifer"]}'


This transaction creates a car asset CAR10 with owner Lucifer.

5. Set Up the Node.js Application

Create a web app directory and install dependencies:

cd ~/fabric-samples
mkdir car-auction-webapp
cd car-auction-webapp
npm init -y
npm install express ejs fabric-network


This will set up a lightweight web application to interact with the blockchain.

6. Connect to Blockchain Network

Inside your app directory:

mkdir ~/fabric-samples/car-auction-webapp
cd ~/fabric-samples/car-auction-webapp


Create app.js to connect and query car details using Fabric Node SDK.

7. Run the Application

Set environment variables and start the app:

export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
node app.js


Access the application at:
👉 http://localhost:3000

This web app allows you to view and interact with car auction data.

8. Shutdown the Network

After testing:

cd ~/fabric-samples/test-network
./network.sh down


This stops all Fabric containers and cleans up the environment.
