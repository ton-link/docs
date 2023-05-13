<div align="center">
<h1> TON Link Documentation </h1>
</div>

- [Introduction](#introduction)
  - [How it works?](#hiw)
    - [Create job](#hiw-cj)
    - [Job structure](#hiw-js)
    - [How nodes sends a response to the oracle?](#hiw-sro)
    - [How a smart contract gets an answer from an oracle?](#hiw-src)
    - [Diagram](#hiw-d)
  - [Deploying your own oracle](#deploy)
    - [Oracle with TON](#native)
    - [Oracle with custom token](#custom)
  - [How do I make a request to the oracle?](#rto)
  - [How to join?](#join)
    - [Registration](#registration)
    - [Run Node](#run)
      - [Manual](#manual)
      - [Automatic](#automatic)

<a name="introduction"></a>
# Introduction
TON Link is a blockchain oracle based on the TON blockchain. It is an algorithm that transfers data between a smart contract and an external data source outside the TON network. Essentially, the oracle serves as an intermediary between the contract and the required data source.

<a name="hiw"></a>
# How it works?
<a name="hiw-cj"></a>
## Create job
When some contract wants information from the real world it must use an oracle to get data. 
In our case the contract has to send a message consisting of the operation code, the information needed by the contract and the link to get the data.
Example of a transaction to retrieve data from a link:
```
var msg_body = begin_cell()
  .store_uint(50, 32) ;; operation code to create a job
  .store_uint(0, 64)
  .store_ref(orig_msg) ;; information needed by the contract
  .store_ref(link) ;; link
 .end_cell();
```

When oracle receives a new transaction with op-code 40, it runs the [`job::create`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/job.func#L112) function. This function performs certain functions:
1. Checks the number of TONs sent, 
2. Generates a new array of addresses - this array stores addresses of nodes which will send the response,
3. Creates job structure.

After completing this function, the job was created and any wallet can get information about it using the [`get_job_by_jobid`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/get.func#L1) method.

<a name="hiw-js"></a>
## Job structure
Job is a certain structure consisting of several fields:

1. jobId (uint64) - sequence number
2. start (uint64) - job creation time
3. end (uint64) - job completion time
4. status (uint1) - job status (completed / not completed)
5  content (cell) - cell that stores the link that the smart-contract sent
6. address (cell) - cell that stores the list of addresses of the nodes that should send the information  
7. info (cell) - cell that stores information to be sent back to the smart-contract (original sender, original time, original msg_value, original msg_body)
8. result (cell) - cell that stores the results that the nodes received by the link

More information about the structure of the work can be found in this [file](https://github.com/ton-link/ton-link-contract-v3/blob/native-token-oracle/typescript/source/NativeOracle/lib/job.md)

<a name="hiw-sro"></a>
## How nodes sends a response to the oracle?

Each node listens to [`get_job_by_jobid`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/get.func#L1) method and reads each job. If a node finds its address in the address list, it executes:
1. Parses the job and writes a link to a local database (also writes the job number so you don't send the results of the same job)
2. Gets information from the link
3. Creates a new transaction with the body:
```
var msg_body = begin_cell()
  .store_uint(160, 32) ;; operation code to add result to job
  .store_uint(0, 64)
  .store_uint(jobID, 64) 
  .store_uint(result, 64) ;; result at the link
.end_cell();
```

This message starts the [`job::add`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/job.func#L464) function. This function checks:
1. Job status. 
2. Whether the node address is in the lists for the job
3. Whether the time has not expired
4. Whether this node has already responded

If a nodes fails to respond on time, she is penalized. If all check items are passed, then the function [`job::add_result`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/job.func#L479) is started.

<a name="hiw-src"></a>
## How a smart contract gets an answer from an oracle?

When the runtime expires or the number of answers equals 5, the function [`job::complete_job`](https://github.com/ton-link/ton-link-contract-v3/blob/d182670c159e41ae943949ad939b6aa99aa0da1c/typescript/source/NativeOracle/lib/job.func#L267) is started, which completes the job and looks for the median value that will eventually become the response of the oracle. 

After that, a new transaction is created with the body:
```
var msg_body = begin_cell()
  .store_slice(original_sender)
  .store_uint(now(), 64)
  .store_grams(original_msg_value)
  .store_ref(begin_cell().store_slice(original_msg_body).end_cell())
  .store_uint(jobID, 64)
  .store_uint(result, 64)
.end_cell();
```
After sending this message the work of the oracle ends.

<a name="hiw-d"></a>
## Diagram
![](https://user-images.githubusercontent.com/86096361/234314944-6232339d-f993-45f4-b674-1d8b41841017.png)

<a name="deploy"></a>
# Deploying your own oracle
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

<a name="rto"></a>
# How do I make a request to the oracle?
When you want to get data from a link from an oracle you need to do a few steps:
1. Send a message to [create a job](#hiw-cj) with the right link and data that the smart contract needs.
2. Create a hook to get answers from the oracle.

Message for creating a new job:
```
var msg_body = begin_cell()
  .store_uint(50, 32) ;; operation code to create a job
  .store_uint(0, 64)
  .store_ref(orig_msg) ;; information needed by the contract
  .store_ref(link) ;; link
 .end_cell();
```

Example of a smart-contract to retrieve data from an oracle:
```
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
          slice ds = get_data().begin_parse();
          slice oracle_address = ds~load_msg_addr();
          
          slice sender_address = utils::parse_sender_address(in_msg_full);
          
          if(equal_slices(sender_address, oracle_address)){
                 slice original_sender = in_msg_body~load_msg_addr();
                 int original_time = in_msg_body~load_uint(64);
                 int original_msg_value = in_msg_body~load_grams();
                 slice original_msg_body = (in_msg_body~load_ref()).begin_parse()
                 int jobID = in_msg_body~load_uint(64);
                 int result = in_msg_body~load_uint(64);
          } else {
                 cell link = ds~load_ref(); ;; https://github.com/ton-link/ton-link-contract-v3/blob/main/typescript/source/lib/link-format.md
                 var msg_body = begin_cell()
                        .store_uint(50, 32)
                        .store_uint(0, 64)
                        .store_ref(in_msg_body)
                        .store_ref(link)
                 .end_cell();

                 var msg = begin_cell()
                        .store_uint(0x18, 6)
                        .store_slice(oracle_address)
                        .store_grams(600000000)
                        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
                        .store_ref(msg_body)
                 .end_cell();
                 send_raw_message(msg, 3);
          }
          return ();
}
```

<a name="join"></a>
# How to join?
To join TON Link you need to follow 2 steps:
1. [Register a wallet in the system](#registration)
2. [Run a node](#run)
<a name="registration"></a>
## Registration
To register in the TON Link system you need to go to https://tonlink.xyz/ and follow a number of simple steps:
1. Connect a wallet via TON Connect v2 or Tonhub Connect
<img width="828" alt="image" src="https://github.com/ton-link/docs/assets/86096361/4c15b119-8eca-45a4-9379-de55c560c732">
2. To register you need to send 30 TON to the wallet oracle. You need to scan the QR-code with your wallet
<img width="828" alt="image" src="https://github.com/ton-link/docs/assets/86096361/f549e431-612e-4e4a-9c32-e85973abbafa">
3. After registering you will see the main menu:
<img width="1324" alt="image" src="https://github.com/ton-link/docs/assets/86096361/39212173-280d-411f-b323-ff557396691d">
After that you can move on to configuring the node and running it.

<a name="run"></a>
## Run Node
You can run the node in two different modes: manual (you will perform all the setup and deployment actions yourself) or automatic (using Docker).
<a name="manual"></a>
### Manual deployment
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
### Automatic deployment
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
