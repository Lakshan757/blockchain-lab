cd ~/fabric/fabric-samples
# Example for first-network (older) or test-network for later versions

cd test-network
./network.sh up createChannel -ca -c mychannel

# if the lab uses first-network BYFN:
cd ~/fabric/fabric-samples/first-network
./byfn.sh generate
./byfn.sh up -l java

# OR for test-network (fabric v2.x)
cd ~/fabric/fabric-samples
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-java/ -ccl java

# sample app path may be fabric-samples/asset-transfer-basic/application-java
cd fabric-samples/asset-transfer-basic/application-java
mvn clean package
java -jar target/application.jar   # sample command; actual jar name may vary

# peer CLI example (requires env variables set by fabric-samples)
peer chaincode invoke -o localhost:7050 ...   # use lab-provided commands
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
