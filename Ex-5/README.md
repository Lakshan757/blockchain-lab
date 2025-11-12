0 — Prerequisites (run once)

Make sure you have:

Docker & Docker Compose

Node.js (>=14) & npm

Git

Hyperledger Fabric samples + binaries (fabric-samples repo)

(Optional) curl, jq

Quick verify (example):

docker --version
docker-compose --version
node -v
npm -v
git --version


If you don’t have fabric-samples set up, a common bootstrap is:

# in your home directory
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples
# run bootstrap script appropriate for your desired versions (example uses 2.4.0)
curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/bootstrap.sh | bash -s -- 2.4.0 1.5.0

1 — Folder / file structure (create these files in a folder fitness-rewards-webapp inside fabric-samples or anywhere)
fitness-rewards-webapp/
├─ chaincode/fitness-chaincode/      # chaincode (javascript)
│  ├─ package.json
│  └─ lib/index.js
├─ app/                              # Node.js web app
│  ├─ package.json
│  ├─ app.js
│  ├─ walletSetup.js
│  └─ views/
│     └─ index.ejs
├─ scripts/
│  ├─ run_network.sh
│  └─ deploy_chaincode.sh
└─ README.md


I’ll give each file content below — create them exactly with the filenames above.

2 — Chaincode (JavaScript) — chaincode/fitness-chaincode/lib/index.js

Create chaincode/fitness-chaincode/package.json and lib/index.js.

chaincode/fitness-chaincode/package.json

{
  "name": "fitness-chaincode",
  "version": "1.0.0",
  "main": "lib/index.js",
  "scripts": {},
  "dependencies": {
    "fabric-shim": "^2.4.0"
  }
}


chaincode/fitness-chaincode/lib/index.js

'use strict';

const { Contract } = require('fabric-contract-api');

class FitnessRewardsContract extends Contract {

    async instantiate(ctx) {
        // optional initialization
        console.info('FitnessRewardsContract instantiated');
    }

    // Create a member
    // memberId, name, email
    async CreateMember(ctx, memberId, name, email) {
        const exists = await this.MemberExists(ctx, memberId);
        if (exists) {
            throw new Error(`Member ${memberId} already exists`);
        }
        const member = {
            id: memberId,
            name,
            email,
            points: 0,
            docType: 'member'
        };
        await ctx.stub.putState(memberId, Buffer.from(JSON.stringify(member)));
        return JSON.stringify(member);
    }

    // Give points to a member
    // memberId, points
    async AwardPoints(ctx, memberId, points) {
        const memberJSON = await ctx.stub.getState(memberId);
        if (!memberJSON || memberJSON.length === 0) {
            throw new Error(`Member ${memberId} does not exist`);
        }
        const member = JSON.parse(memberJSON.toString());
        const p = parseInt(points, 10);
        member.points = (member.points || 0) + p;
        await ctx.stub.putState(memberId, Buffer.from(JSON.stringify(member)));

        // create a simple transaction record
        const txId = ctx.stub.getTxID();
        const rec = {
            id: `tx_${txId}`,
            memberId,
            type: 'AWARD',
            points: p,
            timestamp: (new Date()).toISOString()
        };
        await ctx.stub.putState(rec.id, Buffer.from(JSON.stringify(rec)));

        return JSON.stringify(member);
    }

    // Redeem points
    // memberId, points
    async RedeemPoints(ctx, memberId, points) {
        const memberJSON = await ctx.stub.getState(memberId);
        if (!memberJSON || memberJSON.length === 0) {
            throw new Error(`Member ${memberId} does not exist`);
        }
        const member = JSON.parse(memberJSON.toString());
        const p = parseInt(points, 10);
        if ((member.points || 0) < p) {
            throw new Error(`Member ${memberId} has insufficient points`);
        }
        member.points = member.points - p;
        await ctx.stub.putState(memberId, Buffer.from(JSON.stringify(member)));

        const txId = ctx.stub.getTxID();
        const rec = {
            id: `tx_${txId}`,
            memberId,
            type: 'REDEEM',
            points: p,
            timestamp: (new Date()).toISOString()
        };
        await ctx.stub.putState(rec.id, Buffer.from(JSON.stringify(rec)));
        return JSON.stringify(member);
    }

    // Query member
    async QueryMember(ctx, memberId) {
        const memberJSON = await ctx.stub.getState(memberId);
        if (!memberJSON || memberJSON.length === 0) {
            throw new Error(`Member ${memberId} does not exist`);
        }
        return memberJSON.toString();
    }

    // Get all members (simple range query)
    async GetAllMembers(ctx) {
        const startKey = '';
        const endKey = '';
        const iterator = await ctx.stub.getStateByRange(startKey, endKey);
        const allResults = [];
        while (true) {
            const res = await iterator.next();
            if (res.value && res.value.value.toString()) {
                const Key = res.value.key;
                const Record = JSON.parse(res.value.value.toString('utf8'));
                if (Record.docType && Record.docType === 'member') {
                    allResults.push(Record);
                }
            }
            if (res.done) {
                await iterator.close();
                return JSON.stringify(allResults);
            }
        }
    }

    async MemberExists(ctx, memberId) {
        const memberJSON = await ctx.stub.getState(memberId);
        return (memberJSON && memberJSON.length > 0);
    }
}

module.exports = FitnessRewardsContract;
module.exports.contracts = [FitnessRewardsContract];

3 — Chaincode packaging & deployment script

Two options:

use network.sh deployCC (fabric-samples test-network helper) if available and you put chaincode inside fabric-samples/asset-transfer-basic/chaincode-javascript/ path; or

use manual lifecycle commands.

Simpler: place fitness-chaincode folder under fabric-samples/asset-transfer-basic/chaincode-javascript/fitness-chaincode then run network script to deploy.

scripts/deploy_chaincode.sh

#!/usr/bin/env bash
set -e

# Assumes fabric-samples/test-network present and you're running from fabric-samples directory
# Copy or move chaincode to asset-transfer-basic chaincode location (optional)
# Adjust paths if needed
CC_NAME=fitness
CC_PATH=../asset-transfer-basic/chaincode-javascript/fitness-chaincode

cd "$(dirname "$0")/.." || exit 1
cd ../fabric-samples/test-network || exit 1

# bring up network if not up
./network.sh up createChannel -ca -c mychannel

# deploy chaincode (uses provided helper script)
./network.sh deployCC -ccn $CC_NAME -ccp $CC_PATH -ccl javascript

echo "Chaincode deployed as $CC_NAME"


Make it executable:

chmod +x scripts/deploy_chaincode.sh

4 — Node.js Web App (app)

app/package.json

{
  "name": "fitness-rewards-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "ejs": "^3.1.8",
    "fabric-network": "^2.4.0",
    "fs-extra": "^11.1.1",
    "path": "^0.12.7"
  }
}


app/app.js — a minimal web server with endpoints to create member, award, redeem, and view members:

const express = require('express');
const { Gateway, Wallets } = require('fabric-network');
const path = require('path');
const fs = require('fs');

const app = express();
app.use(express.urlencoded({ extended: true }));
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));

const ccpPath = path.resolve(__dirname, '..', 'fabric-samples', 'test-network', 'organizations', 'peerOrganizations', 'org1.example.com', 'connection-org1.json');
// NOTE: adjust the path to your connection profile location

async function buildGateway() {
    const ccp = JSON.parse(fs.readFileSync(ccpPath, 'utf8'));
    const walletPath = path.join(__dirname, 'wallet');
    const wallet = await Wallets.newFileSystemWallet(walletPath);

    const gateway = new Gateway();
    await gateway.connect(ccp, {
        wallet,
        identity: 'appUser',
        discovery: { enabled: true, asLocalhost: true }
    });
    return gateway;
}

app.get('/', async (req, res) => {
    res.render('index', { members: [], message: null });
});

// Show members
app.get('/members', async (req, res) => {
    try {
        const gateway = await buildGateway();
        const network = await gateway.getNetwork('mychannel');
        const contract = network.getContract('fitness'); // chaincode name
        const result = await contract.evaluateTransaction('GetAllMembers');
        await gateway.disconnect();
        const members = JSON.parse(result.toString());
        res.render('index', { members, message: null });
    } catch (err) {
        console.error(err);
        res.render('index', { members: [], message: 'Error: ' + err.message });
    }
});

// Create member
app.post('/create', async (req, res) => {
    const { memberId, name, email } = req.body;
    try {
        const gateway = await buildGateway();
        const network = await gateway.getNetwork('mychannel');
        const contract = network.getContract('fitness');
        await contract.submitTransaction('CreateMember', memberId, name, email);
        await gateway.disconnect();
        res.redirect('/members');
    } catch (err) {
        console.error(err);
        res.send('Error: ' + err.message);
    }
});

// Award points
app.post('/award', async (req, res) => {
    const { memberId, points } = req.body;
    try {
        const gateway = await buildGateway();
        const network = await gateway.getNetwork('mychannel');
        const contract = network.getContract('fitness');
        await contract.submitTransaction('AwardPoints', memberId, points);
        await gateway.disconnect();
        res.redirect('/members');
    } catch (err) {
        console.error(err);
        res.send('Error: ' + err.message);
    }
});

// Redeem points
app.post('/redeem', async (req, res) => {
    const { memberId, points } = req.body;
    try {
        const gateway = await buildGateway();
        const network = await gateway.getNetwork('mychannel');
        const contract = network.getContract('fitness');
        await contract.submitTransaction('RedeemPoints', memberId, points);
        await gateway.disconnect();
        res.redirect('/members');
    } catch (err) {
        console.error(err);
        res.send('Error: ' + err.message);
    }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Fitness rewards app running on http://localhost:${PORT}`));


app/views/index.ejs

<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Fitness Club Rewards</title>
</head>
<body>
  <h1>Fitness Club Rewards</h1>

  <h2>Create Member</h2>
  <form method="post" action="/create">
    <input name="memberId" placeholder="memberId" required />
    <input name="name" placeholder="Name" required />
    <input name="email" placeholder="Email" required />
    <button type="submit">Create</button>
  </form>

  <h2>Award Points</h2>
  <form method="post" action="/award">
    <input name="memberId" placeholder="memberId" required />
    <input name="points" type="number" placeholder="points" required />
    <button type="submit">Award</button>
  </form>

  <h2>Redeem Points</h2>
  <form method="post" action="/redeem">
    <input name="memberId" placeholder="memberId" required />
    <input name="points" type="number" placeholder="points" required />
    <button type="submit">Redeem</button>
  </form>

  <h2>Members</h2>
  <form method="get" action="/members"><button type="submit">Refresh Members</button></form>

  <% if (message) { %>
    <p><strong><%= message %></strong></p>
  <% } %>

  <% if (members && members.length) { %>
    <table border="1" cellpadding="6">
      <thead><tr><th>ID</th><th>Name</th><th>Email</th><th>Points</th></tr></thead>
      <tbody>
        <% members.forEach(function(m) { %>
          <tr>
            <td><%= m.id %></td>
            <td><%= m.name %></td>
            <td><%= m.email %></td>
            <td><%= m.points %></td>
          </tr>
        <% }) %>
      </tbody>
    </table>
  <% } else { %>
    <p>No members yet.</p>
  <% } %>
</body>
</html>


app/walletSetup.js — helper to enroll an appUser identity into a wallet (use Fabric CA or copy sample admin wallet). If you're using fabric-samples test-network, you can use the test-network script to create appUser (there's registerEnrollAdmin example) — but here's a simple approach to copy appUser identity from sample wallet if available. This file is optional; you may use the sample wallet in fabric-samples asset-transfer-basic/application-javascript/wallet and copy it into app/wallet.

5 — Scripts to run everything

scripts/run_network.sh

#!/usr/bin/env bash
set -e

# run from repo root where fabric-samples exists; adjust if not
cd ../fabric-samples/test-network || exit 1

# Start network and create channel
./network.sh up createChannel -ca -c mychannel

# Deploy the chaincode (fitness)
./network.sh deployCC -ccn fitness -ccp ../asset-transfer-basic/chaincode-javascript/fitness-chaincode -ccl javascript

echo "Network up and chaincode deployed"


Make scripts executable:

chmod +x scripts/run_network.sh scripts/deploy_chaincode.sh

6 — Full Step-by-step execution (commands you must run, in order)

A. Place files

Put chaincode/fitness-chaincode into:

fabric-samples/asset-transfer-basic/chaincode-javascript/fitness-chaincode


(or change paths in scripts accordingly).

Put app folder anywhere (example: ~/fabric-samples/fitness-rewards-webapp/app).

Put scripts at top level (or adjust paths).

B. Start Fabric test network and deploy chaincode
From fabric-samples/test-network location or using scripts/run_network.sh:

# from the folder that contains scripts/run_network.sh
./scripts/run_network.sh


This will run ./network.sh up createChannel and deployCC.

C. Create/ensure app user identity (wallet)

If you used network.sh helper, sample enroll scripts create appUser for asset-transfer-basic. You can copy the sample wallet directory:

# example: copy wallet from asset-transfer-basic sample
cp -r fabric-samples/asset-transfer-basic/application-javascript/wallet <path-to-your-app>/wallet


If the sample didn't create an appUser, follow fabric-samples tutorials to register & enroll appUser using CA scripts (there are helper scripts in asset-transfer-basic). Alternatively, use fabric-samples/test-network/scripts to create an identity.

D. Install app dependencies

cd <path-to-app-folder>/app
npm install


E. Set connection profile path in app.js
Make sure ccpPath points to the correct connection-org1.json:
fabric-samples/test-network/organizations/peerOrganizations/org1.example.com/connection-org1.json

F. Run the app

node app.js
# open http://localhost:3000


G. Test flows via UI

Create a member (memberId: member1, name, email).

Award points to the member.

Redeem points (must be <= member.points).

Click "Refresh Members" to see updated points.

H. Using peer CLI to check chaincode
You can use the Fabric peer CLI to query ledger (from fabric-samples/test-network with environment variables set). Example:

# if environment variables are set by peer CLI scripts
peer chaincode query -C mychannel -n fitness -c '{"Args":["QueryMember","member1"]}'
# or call GetAllMembers:
peer chaincode query -C mychannel -n fitness -c '{"Args":["GetAllMembers"]}'

7 — Example transactions (CLI-style using Node app / SDK)

From Node web UI you will submit transactions. If you prefer to call via Node script, sample code:

app/testInvoke.js (optional)

const { Gateway, Wallets } = require('fabric-network');
const fs = require('fs');
const path = require('path');

async function main(){
  const ccpPath = path.resolve(__dirname, '..', 'fabric-samples', 'test-network', 'organizations', 'peerOrganizations', 'org1.example.com', 'connection-org1.json');
  const ccp = JSON.parse(fs.readFileSync(ccpPath, 'utf8'));
  const wallet = await Wallets.newFileSystemWallet(path.join(__dirname, 'wallet'));
  const gateway = new Gateway();
  await gateway.connect(ccp, { wallet, identity: 'appUser', discovery: { enabled:true, asLocalhost:true }});
  const network = await gateway.getNetwork('mychannel');
  const contract = network.getContract('fitness');

  // create member
  await contract.submitTransaction('CreateMember','member1','Alice','alice@example.com');
  console.log('Member created');

  // award points
  await contract.submitTransaction('AwardPoints','member1','50');
  console.log('Points awarded');

  // query
  const out = await contract.evaluateTransaction('QueryMember','member1');
  console.log('Member:', out.toString());

  await gateway.disconnect();
}
main();


Run:

node testInvoke.js
