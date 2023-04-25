<div align="center">
<h1> TON Link Documentation </h1>
</div>

- [Introduction](#introduction)
  - [Deploy oracle](#deploy)
    - [Oracle with TON](#native)
    - [Oracle with custom token](#custom)
  - [Registration](#registration)
  - [Run Node](#run)
    - [Manual](#manual)
    - [Automatic](#automatic)

<a name="introduction"></a>
# Introduction
TON Link is a blockchain oracle based on the TON blockchain. It is an algorithm that transfers data between a smart contract and an external data source outside the TON network. Essentially, the oracle serves as an intermediary between the contract and the required data source.
![](https://user-images.githubusercontent.com/86096361/234314944-6232339d-f993-45f4-b674-1d8b41841017.png)

When the oracle receives a new transaction, it collects and organizes the payload and creates a job. Once the job is created, the nodes check the job, get the information from the link in the job, and send their answer. There are a total of 5 answers and the median value among them is taken as the result. After the nodes have sent their answers and the median value has been chosen, nodes that have sent a result with a deviation of more than 20% are penalized. When the task is complete, the oracle sends the response to the user smart contract as a payload (the payload includes the information required by the user smart contract, the number of Toncoins sent, and the data from the specified link).

<a name="deploy"></a>
# Deploy oracle
TON Link has two different kinds of oracle:
1. Oracle based on TON - uses TON to pay requests and payments to nodes
2. Oracle based on custom token - uses custom token to pay requests and payment to nodes
<a name="native"></a>
## Oracle based on TON
1. Clone the repository: `git clone -b native-token-oracle https://github.com/ton-link/ton-link-contract-v3.git`
2. Enter the newly created folder and install packages: cd ton-link-contract-v3/typescript && npm i
3. Rename .env.example to .env
4. Modify the following fields in the .env file:
   - mode: Set to testnet or mainnet, depending on the network you want to deploy the oracle in.
   - mnemonic: Your seed phrase for deploying the oracle.
   - admin: The address of the oracle administrator, who can change the minter address and token wallet address. Be cautious (it is recommended to use the address from which the deployment will be carried out).
5. To deploy the oracle, you need run the command `npm run deployNativeOracle`. The console will display the oracle address, which is also duplicated in the oracle.txt file. Wait for the transaction to complete (check the explorer). If the transaction fails, don't worry, this is normal.

<a name="custom"></a>
## Oracle based on custom token
1. Clone the repository: `git clone -b native-token-oracle https://github.com/ton-link/ton-link-contract-v3.git`
2. Enter the newly created folder and install packages: cd ton-link-contract-v3/typescript && npm i
3. Rename .env.example to .env
4. Modify the following fields in the .env file:
   - mode: Set to testnet or mainnet, depending on the network you want to deploy the oracle in.
   - mnemonic: Your seed phrase for deploying the oracle.
   - jMinter: The address of the token minter, needed for rewarding oracles for providing information.
   - admin: The address of the oracle administrator, who can change the minter address and token wallet address. Be cautious (it is recommended to use the address from which the deployment will be carried out).
5. To deploy the oracle, send two transactions:
   - Transaction 1: Deploy the contract. In the `scripts/deployCustomTokenOracle.ts file`, comment out line 105. Then, run the command `npm run deployCustomTokenOracle`. The console will display the oracle address, which is also duplicated in the oracle.txt file. Wait for the transaction to complete (check the explorer). If the transaction fails, don't worry, this is normal.
   - Transaction 2: Change the token wallet address. In the `scripts/deployCustomTokenOracle.ts` file, comment out line 104 and uncomment line 105. Run the command `npm run deployCustomTokenOracle` again. Check the explorer; the new transaction should succeed without errors. To verify that everything is correct, go back to the scripts/deployCustomTokenOracle.ts file and comment out line 105. Run the npm run deploy command again and wait. The last thing that should appear in the console is the word "true." If so, congratulations, you have deployed your oracle.

<a name="registration"></a>
# Registration
Will be updated

<a name="run"></a>
# Run Node
You can run the node in two different modes: manual (you will perform all the setup and deployment actions yourself) or automatic (using Docker).
<a name="manual"></a>
## Manual deployment
1. Clone the repository: `git clone https://github.com/ton-link/ton-link-node-v3.git`
2. Go to the `ton-link-node-v3/client` folder and install the necessary packages with the `npm install` 
3. You need to install PostgreSQL (>= 11.x) for the node to work.
4. After installing PostgreSQL, you need to create 2 tables with commands:
   - `CREATE TABLE job (jobID INT NOT NULL, ownership_job INT NOT NULL, status_job INT NOT NULL, place INT NOT NULL, start_time INT NOT NULL, end_time INT NOT NULL);` - this command creates the job table that stores completed jobs
   - `CREATE TABLE link (jobID INT NOT NULL, count INT NOT NULL, first_link VARCHAR(255) NOT NULL, second VARCHAR(255) NOT NULL, third VARCHAR(255) NOT NULL, fourth VARCHAR(255) NOT NULL);` - this command creates the link table that stores links
5. In the ton-link-node-v3 folder, there is a `.env.example` file that you need to rename to `.env` and modify the fields:
   - SEED: Your seed phrase for wallet access
   - MYADDRESS: The address of your account, which should be a node
   - ORACLEADDRESS: The oracle address (future deployed oracle addresses will be published on GitHub)
   - NETWORK: Enter the network where the current version of the oracle is running (enter mainnet or testnet)
   - TON_API_URL: The URL to the public API for interaction with TON
   - TONCENTERKEY: The API access key (if using TON Center, you can obtain the key: for mainnet @tonapibot, for testnet @tontestnetapibot)
   - DATABASE_URL: The database URL in the format postgresql://tonlink:password@your_local_ip_address:5432/tonlink_db
   - WALLET_V: Your wallet version (enter in the format v3R2)
6. Move the `.env` file to the client folder.
7. After successful registration you need to run the node with the command: `node client/client`
<a name="automatic"></a>
## Automatic deployment
1. In the ton-link-node-v3 folder, there is a `.env.example` file that you need to rename to `.env` and modify the fields:
   - SEED: Your seed phrase for wallet access
   - MYADDRESS: The address of your account, which should be a node
   - ORACLEADDRESS: The oracle address (future deployed oracle addresses will be published on GitHub)
   - NETWORK: Enter the network where the current version of the oracle is running (enter mainnet or testnet)
   - TON_API_URL: The URL to the public API for interaction with TON
   - TONCENTERKEY: The API access key (if using TON Center, you can obtain the key: for mainnet @tonapibot, for testnet @tontestnetapibot)
   - DATABASE_URL: The database URL in the format postgresql://tonlink:password@your_local_ip_address:5432/tonlink_db
   - WALLET_V: Your wallet version (enter in the format v3R2)
2. Move the .env file to the client folder.
3. Open the terminal at the folder address and run the command `docker-compose build`, then wait for it to complete.
4. After successful registration, you need to start the node with the command: `docker-compose up`
