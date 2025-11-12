cd ~/fabric/fabric-samples/test-network
./network.sh up createChannel -ca -c mychannel
./network.sh deployCC -ccn carauction -ccp ../asset-transfer-basic/chaincode-javascript/ -ccl javascript

mkdir -p ~/fabric/car-auction-webapp
cd ~/fabric/car-auction-webapp
npm init -y
npm install express ejs fabric-network
# add app.js to interact with chaincode (createCar, bid, transfer)
node app.js

peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ... -C mychannel -n carauction -c '{"function":"CreateCar","Args":["CAR10","Toyota","Camry","Red","Lucifer"]}'
