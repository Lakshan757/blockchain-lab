# install prerequisites
sudo apt install docker.io docker-compose nodejs npm git -y

# clone fabric samples
git clone https://github.com/hyperledger/fabric-samples.git

cd fabric-samples

curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/bootstrap.sh | bash -s -- 2.4.0 1.5.0

# start test network
cd test-network

./network.sh up createChannel -ca -c mychannel

./network.sh deployCC -ccn fitness -ccp ../asset-transfer-basic/chaincode-javascript -ccl javascript

// ⚡ Minimal Fitness Rewards Web App using Hyperledger Fabric

const express = require("express"), { Gateway, Wallets } = require("fabric-network");

const fs = require("fs"), path = require("path"), app = express();

app.use(express.urlencoded({ extended: true }));

const ccp = JSON.parse(fs.readFileSync(path.resolve(__dirname,

 "fabric-samples/test-network/organizations/peerOrganizations/org1.example.com/connection-org1.json")));
 
const walletPath = path.join(__dirname, "fabric-samples/asset-transfer-basic/application-javascript/wallet");

async function useFabric(fn) {

  const wallet = await Wallets.newFileSystemWallet(walletPath);
  
  const gateway = new Gateway();
  
  await gateway.connect(ccp, { wallet, identity: "appUser", discovery: { enabled: true, asLocalhost: true } });
  
  const contract = (await gateway.getNetwork("mychannel")).getContract("fitness");
  
  await fn(contract);
  
  await gateway.disconnect();
}

app.get("/", (r, s) => s.send(`

<h2>🏋️ Fitness Club Rewards</h2>

<form method=post action=create><input name=id placeholder=ID><input name=name placeholder=Name><input name=email placeholder=Email><button>Create</button></form>

<form method=post action=award><input name=id placeholder=ID><input name=points placeholder=Points><button>Award</button></form>

<form method=post action=redeem><input name=id placeholder=ID><input name=points placeholder=Points><button>Redeem</button></form>

<a href=/all>View All Members</a>`));

app.post("/create", async (r, s) => { try {

  await useFabric(c => c.submitTransaction("CreateAsset", r.body.id, r.body.name, r.body.email));
  
  s.send("✅ Member Created"); } catch(e){ s.send("❌ "+e.message); } });

app.post("/award", async (r, s) => { try {

  await useFabric(c => c.submitTransaction("TransferAsset", r.body.id, "Award+"+r.body.points));
  
  s.send("💰 Points Added"); } catch(e){ s.send("❌ "+e.message); } });

app.post("/redeem", async (r, s) => { try {

  await useFabric(c => c.submitTransaction("TransferAsset", r.body.id, "Redeem-"+r.body.points));
  
  s.send("💸 Points Redeemed"); } catch(e){ s.send("❌ "+e.message); } });

app.get("/all", async (r, s) => { try {

  await useFabric(async c => s.send("<pre>"+(await c.evaluateTransaction("GetAllAssets")).toString()+"</pre>"));
  
} catch(e){ s.send("❌ "+e.message); } });

app.listen(3000, () => console.log("🚀 Running on http://localhost:3000"));

npm install express fabric-network

node fitness.js

Then open 👉 http://localhost:3000
