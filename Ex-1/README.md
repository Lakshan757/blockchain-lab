sudo apt install -y docker.io docker-compose

sudo apt install -y nodejs npm   # or use NodeSource for latest Node

sudo apt install -y openjdk-11-jdk

mkdir -p ~/fabric && cd ~/fabric

curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/bootstrap.sh | bash -s -- 2.4.0 1.5.0

docker --version

node -v

java -version

npm install -g ganache-cli
# or
npm install --save-dev hardhat
