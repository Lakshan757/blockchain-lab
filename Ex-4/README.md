export PATH=${PWD}/../../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../../config/

cd ~/fabric/fabric-samples/asset-transfer-basic/application-javascript
npm install
node app.js

# or run a specific script that performs create/query/transfer
node invoke.js

node query.js
