
# Directory structure:
```
adapter - the adapter that confirms a contract in response to a tweet mentioning it's address.
contracts - solidity contracts for oracle and consumers are here.
docker - docker files for 3rd party program that are not available in docker hub. i.e. ganache-cli.
LinkToken - git submodule to https://github.com/smartcontractkit/LinkToken. Used to deploy the Link token to our local test chain.
manifests - kubernetes manifests to install everything automatically.
scripts - Helper web3 JS scripts used in this readme to perform operations on the blockchain without the Remix GUI.
```

# Dev env setup
To get started I used ganache-cli and truffle, for new projects, these can be installed with npm:
```
npm install --save ganache-cli truffle
```
As this step was already done, you just need to do
```
npm install
```
to install them locally from the saved package.json.
In addition to `npm` (tested with node 10) you will need:
- make
- docker
- kind
- kubectl
- jq
- curl
- geth

For adapter development, you will need `pipenv`.

We'll start by creating a local setup in kubernetes. To keep things simple, we will us KinD to 
setup everything on our laptop. Kubernetes is used so we can replicate the setup fast in a reproducible fashion.
This can also potentially run in CI systems.

# Prep work

To re-generate the db yaml, use the following:
```
helm template release bitnami/postgresql > manifests/postgresql.yaml
```
# Deploy and test infra structure:
Deploy the infrastructure, starting with the ganache and the coin:
we use ganache in deterministic mode, so ADDRESS and KEY should be the same every time. you can see them in the ganache log output.

```bash
make deploy-testnet
kubectl rollout status deploy/ganache
# may need to sleep here to see logs
# the created addresses will be in the log.
kubectl logs deploy/ganache
# may need to sleep here until ganache initializes
make deploy-token
```

in the output of `make deploy-token` you will see:
```
  LinkToken: 0x5b1869d9a4c187f2eaa108f3062412ecf0526b24
```


Deploy chainlink node:

```bash
# first address generated by ganache:
export ADDRESS=0x90F8bf6A479f320ead074411a4B0e7944Ea8c9C1
export LINK_TOKEN=0x5b1869d9a4c187f2eaa108f3062412ecf0526b24
# verify that Link token 0x5b1869d9a4c187f2eaa108f3062412ecf0526b24
make deploy-node

# wait for node:
kubectl rollout status deploy/chainlink
# watch logs to see when it is really alive. It may crash a couple of times while db initializes..
# just let it to it's thing for a few minutes
kubectl logs deploy/chainlink -f

# This may take a while, so if NODE_ADDR is empty, sleep might help. if this comes empty, try again after a minute or so.
export NODE_ADDR=$(kubectl logs deploy/chainlink|grep "please deposit ETH into your address:"| tr ' ' '\n'|grep 0x)
```

Fund the node with ETH and LINK
```bash
# add 10 eth to the node
geth attach http://localhost:32000 -exec 'eth.sendTransaction({from: "'${ADDRESS}'",to: "'${NODE_ADDR}'", value: "10000000000000000000"})'

# add link to the node
geth attach http://localhost:32000 --jspath ./scripts -exec 'loadScript("fund.js");transfer("'$LINK_TOKEN'", "'$ADDRESS'", "'$NODE_ADDR'");'

# to verify (optional), check node balance:
geth attach http://localhost:32000 --jspath ./scripts -exec 'loadScript("fund.js");getbalance("'$LINK_TOKEN'", "'$ADDRESS'", "'$NODE_ADDR'");'
```

Deploy oracle:

```bash
# this will also add the node to the oracle (by using the address in the env-var )
npm run deploy-oracle | tee node-tmp.txt
export ORACLE_ADDR=$(grep "contract-address" node-tmp.txt | cut -f 2)
rm node-tmp.txt
```

Add jobs to the node:

port forward to the ui/api/s:
```bash
kubectl port-forward deploy/chainlink 6688&
```

Log-in in to the node:
```bash
curl -c cookiefile \
  -d '{"email":"foo@example.com", "password":"apipassword"}' \
  -X POST -H 'Content-Type: application/json' \
   http://localhost:6688/sessions
```

Now we can use the API keys to create the jobs:

```bash
# optional: verify that the node sees its balances:
# curl http://localhost:6688/v2/user/balances -H"X-API-KEY: $ACCESS_KEY" -H"X-API-SECRET: $SECRET_KEY"

# create the jobs:
curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d '{"initiators":[{"type":"runlog","params":{"address":"'$ORACLE_ADDR'"}}],"tasks":[{"type":"httpget"},{"type":"jsonparse"},{"type":"ethbytes32"},{"type":"ethtx"}]}'

curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d '{"initiators":[{"type":"runlog","params":{"address":"'$ORACLE_ADDR'"}}],"tasks":[{"type":"httppost"},{"type":"jsonparse"},{"type":"ethbytes32"},{"type":"ethtx"}]}'

curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d '{"initiators":[{"type":"runlog","params":{"address":"'$ORACLE_ADDR'"}}],"tasks":[{"type":"httpget"},{"type":"jsonparse"},{"type":"multiply"},{"type":"ethint256"},{"type":"ethtx"}]}'

# save job id of EthUint256 as we need it for later
JOB_ID=$(curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d '{"initiators":[{"type":"runlog","params":{"address":"'$ORACLE_ADDR'"}}],"tasks":[{"type":"httpget"},{"type":"jsonparse"},{"type":"multiply"},{"type":"ethuint256"},{"type":"ethtx"}]}' | jq .data.id -r)

curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d '{"initiators":[{"type":"runlog","params":{"address":"'$ORACLE_ADDR'"}}],"tasks":[{"type":"httpget"},{"type":"jsonparse"},{"type":"ethbool"},{"type":"ethtx"}]}'
```

We now have the environment setup!
Using the node!

Create a consumer:

```bash
npm run deploy-testconsumer | tee node-tmp.txt
export TEST_CONTRACT_ADDR=$(grep "contract-address" node-tmp.txt | cut -f 2)
rm node-tmp.txt
# fund the consumer contract:
geth attach http://localhost:32000 --jspath ./scripts -exec 'loadScript("fund.js");transfer("'$LINK_TOKEN'", "'$ADDRESS'", "'$TEST_CONTRACT_ADDR'");'
# verify funds contract:
geth attach http://localhost:32000 --jspath ./scripts -exec 'loadScript("fund.js");getbalance("'$LINK_TOKEN'", "'$TEST_CONTRACT_ADDR'");'

# make a request from the contract
node scripts/testcontract.js $TEST_CONTRACT_ADDR $ORACLE_ADDR $JOB_ID
```

You should see a job executed in the node UI! success!!


# Debugging
if we see failure, we can get transaction id and debug with truffle:
```bash
kubectl logs deploy/ganache
# get transaction id; go to the truffle directory containing the contract, and:
../../node_modules/.bin/truffle debug --network ganache <transaction id>
```


# Deploy twitter adapter:

register the node:
```
curl -c cookiefile \
  -d '{"email":"foo@example.com", "password":"apipassword"}' \
  -X POST -H 'Content-Type: application/json' \
   http://localhost:6688/sessions

curl -b cookiefile http://localhost:6688/v2/bridge_types -XPOST -H"content-type: application/json" -d @adapter/bridge.json > bridge_create.json
export INCOMING_TOKEN=$(jq '.data.attributes.incomingToken' bridge_create.json -r)
export OUTGOING_TOKEN=$(jq '.data.attributes.outgoingToken' bridge_create.json -r)
rm bridge_create.json
```

if needed you can delete the bridge like so: `curl -v -b cookiefile -X DELETE http://localhost:6688/v2/bridge_types/twitter`

Have your twitter secrets setup as environment variables:
- TWITTER_API_KEY
- TWITTER_API_KEY_SECRET
- TWITTER_ACCESS_TOKEN
- TWITTER_ACCESS_TOKEN_SECRET

Now we can deploy the adapter:
```bash
kubectl create secret generic twitter-adapter \
    --from-literal=TWITTER_API_KEY=$TWITTER_API_KEY \
    --from-literal=TWITTER_API_KEY_SECRET=$TWITTER_API_KEY_SECRET \
    --from-literal=TWITTER_ACCESS_TOKEN=$TWITTER_ACCESS_TOKEN \
    --from-literal=TWITTER_ACCESS_TOKEN_SECRET=$TWITTER_ACCESS_TOKEN_SECRET \
    --from-literal=INCOMING_TOKEN=$INCOMING_TOKEN \
    --from-literal=OUTGOING_TOKEN=$OUTGOING_TOKEN
make deploy-adapter
```

add the twitter job spec:

```bash
sed -e "s/ORACLE_ADDR/$ORACLE_ADDR/" adapter/jobspec.json > jobspec.json
export TWITTER_JOB_ID=$(curl -b cookiefile http://localhost:6688/v2/specs -XPOST -H"content-type: application/json" -d @jobspec.json | jq .data.id -r)
rm jobspec.json
```

# time to test!

Create a contract with these parameters:


- link: address of link ERC20 token ($LINK_TOKEN)
- deadline: how long before the contract expires (in seconds; for reference, 86400 seconds are 24 hours.)
- beneficiary: address of receiver of funds when approved. (for example, the 2nd address generated by ganache: 0xFFcf8FDEE72ac11b5c542428B35EEF5769C409f0)
- amount: the amount of ETH that should be transferred to the contract address before it is ready for approval (in wei; 1000000000000000000 is 1 ETH)
- approver_twitter_handle: the twitter handle of the approver
- text: The text the approver needs to tweet for approval
- oracle: the oracle address ($ORACLE_ADDR)
- jobId: the jobId for the twitter verification ($TWITTER_JOB_ID),

for example:
```bash
npm run build-twitterconsumer
# deploy contract and get its address
node scripts/twitterconsumer/deploy.js $LINK_TOKEN 86400 0xFFcf8FDEE72ac11b5c542428B35EEF5769C409f0 1000000000000000000 KohaviYuval les $ORACLE_ADDR $TWITTER_JOB_ID | tee output.txt
DEPLOYED_TC_ADDR=$(grep "contract address:" output.txt | cut -d: -f2 | tr -d ' ')
rm output.txt
# fund contract with link for the oracle
geth attach http://localhost:32000 --jspath ./scripts -exec 'loadScript("fund.js");transfer("'$LINK_TOKEN'", "'$ADDRESS'", "'$DEPLOYED_TC_ADDR'");'

# fund contract with eth for the beneficiary
node scripts/twitterconsumer/fund.js $DEPLOYED_TC_ADDR 1000000000000000000

# check if the contracts is ready (i.e. funded with link and ETH)
node scripts/twitterconsumer/ready.js $DEPLOYED_TC_ADDR 
```

Now go and tweet your approval message.
once real world transaction happens, request approval:
```bash
# request approval
node scripts/twitterconsumer/requestApproval.js $DEPLOYED_TC_ADDR 
```

make some noise on the network so that the transaction is confirmed. move some eth between ganache addresses 8,9
```bash
geth attach http://localhost:32000 -exec 'eth.sendTransaction({from: "0xACa94ef8bD5ffEE41947b4585a84BdA5a3d3DA6E",to: "0x1dF62f291b2E969fB0849d99D9Ce41e2F137006e", value: "10000000000000000000"})'
geth attach http://localhost:32000 -exec 'eth.sendTransaction({from: "0x1dF62f291b2E969fB0849d99D9Ce41e2F137006e",to: "0xACa94ef8bD5ffEE41947b4585a84BdA5a3d3DA6E", value: "10000000000000000000"})'
geth attach http://localhost:32000 -exec 'eth.sendTransaction({from: "0xACa94ef8bD5ffEE41947b4585a84BdA5a3d3DA6E",to: "0x1dF62f291b2E969fB0849d99D9Ce41e2F137006e", value: "10000000000000000000"})'
geth attach http://localhost:32000 -exec 'eth.sendTransaction({from: "0x1dF62f291b2E969fB0849d99D9Ce41e2F137006e",to: "0xACa94ef8bD5ffEE41947b4585a84BdA5a3d3DA6E", value: "10000000000000000000"})'
```

Once approval is done, you may need to make some more network noise for confirmation.
now you can withdraw!
```bash
# withdraw the funds!
node scripts/twitterconsumer/isapproved.js $DEPLOYED_TC_ADDR

node scripts/twitterconsumer/withdraw.js $DEPLOYED_TC_ADDR 0xFFcf8FDEE72ac11b5c542428B35EEF5769C409f0

# check ETH balance:
geth attach http://localhost:32000 -exec 'eth.getBalance("0xFFcf8FDEE72ac11b5c542428B35EEF5769C409f0")'
```

faq:
why use kubernetes? it is good to do as it resembles what will be in prod. 
with minor tweaks, all this code can be deployed to a live environment.

why marriage - every one can relate to it;
the goal is to show how we can bridge processes in the blocking to existing approval flows. with the hopes that this will make it more accessible to larger and more established enterprises.
think approval flow, where you need an email approval from you manager - it follows the same structure.

social media influences:
agree with an influencer to tweet about your brand and only pay him after he did