# Introduction
In this tutorial, We will discuss how to create a crowdfunding dapp on tezos blockchain using which a user can ask for funds and other user can help him by crowdfunding. We will be going through writing the smart contracts, creating the UI, and invoking the entry points in the front end.

# Prerequisites
- Basic familiarity with [ReactJS](https://reactjs.org/).
- Should also have some experience writing smart contracts on Tezos with SmartPy.

# Requirements
- A wallet installed in your browser for Testing. We will use [Temple wallet](https://templewallet.com/)
- React js installed.
- npm

# Why do we need crowdfunding application on blockchain?
In normal web2 based crowdfunding platforms you have to trust the platform for keeping your funds safe whenever you fund someone. Both the investor and fund raiser are trusting the platform. But in case of blockchain based crowdfuning solution, All of the funds are securley stored on the smart contract itself which is not tamperable and is purely immuatble that helps in creating a trustless and decentralised system (code is law).

# Smart Contract
We will be building two contracts for this Dapp.
- First will be the crowdfunding project contract which will be responsible for all the functionalities of a project like funding, refunds, and claiming funds if the crowdfund turns out to be successful. 
- Second smart contract will keep track of all the crowdfunding projects. It will store the addresses and data of the projects. It will be also used to orignate project contract.

## Project contract
We will use SmartPy online IDE to build out the contract. You can also use the SmartPy CLI if you prefer.

1. Go to [smartpy.io/ide](smartpy.io/ide)
2. We will first import the Smartpy library

```python
import smartpy as sp
```
3. Before writing the functions, let us create the contract boilerplate. We will name our class `Project`. Let's initialize data fields in the init function. 
We are going to initialize types

    - `funding`: A map which will store the funding data. It will store amount funded for multiple addresses.
    - `owner`: This is the address of the owner/fund raiser of the project.
    - `goalAmount`: The amount in mutez that is to be raised for this project.
    - `endTime`: When will this this project crowdfunding end.
    - `name`: Name of the project.
    - `description`: Description of the project.
```python
class Project(sp.Contract):
    def __init__(self):
        self.init_type(sp.TRecord(
            funding=sp.TMap(k=sp.TAddress, v=sp.TMutez),
            owner=sp.TAddress,
            goalAmount=sp.TMutez,
            endTime=sp.TTimestamp,
            name=sp.TString,
            description=sp.TString
            )
        )
```
4. Lets create a function using which anyone can fund the project.
Lets name the function `send_fund`. 

    - First we will make sure that project is not ended.
    - To keep the tutorial simple we will allow the investor to fund only once. So we will make sure that the funding map does contain sender's address already. 
    - Now we will simply add the sender address as key with the amount sent as value in the funding map. 

```python
    @sp.entry_point
    def send_fund(self):
        sp.verify(self.data.endTime >= sp.now) 
        sp.verify(~self.data.funding.contains(sp.sender))
        self.data.funding[sp.sender]= sp.amount
```

5. Now we will create a function using which the fund raiser can get the funds if the crowdfund turns out to be successful.
Lets name the function `pay_off`. 

    - First we will make sure that the sender's address is equal to the owner's address because only the project creator should be able to claim funds from the crowdfund.
    - Now we will make the sure that the contract balance is greater or equal to the goal amount. This ensures that the crowdfund was successful.
    - We will also make sure that the project is not live and is ended successfully by comparing current time with endtime.
    - Now we will simply send all of the funds collected (contract balance) to the owner.

```python
    @sp.entry_point
    def pay_off(self):
        sp.verify(self.data.owner==sp.sender)
        sp.verify(self.data.goalAmount <= sp.balance)
        sp.verify(self.data.endTime <= sp.now)
        sp.send(self.data.owner, sp.balance)
```

6. Lets create a function now using which a investor can get his funds back that he invested.
Lets name the function `refund`. 

    - First lets make sure that he is already an investor. We will check if the funding map contains his address or not.
    - Now we will make the sure that the contract balance is less than goal amount. This ensures that the crowdfund is not yet successful. We dont want the investor to take the refund after crowdfund is succesful. Its upto you how you want your crowdfund platform to work, So you can remove this condition also.
    - Now we will send back all of the funds to the sender address that he funded in the past.
    - In the end we will just simply delete this sender's entry from the funding map.

```python
    @sp.entry_point
    def refund(self):
        sp.verify(self.data.funding.contains(sp.sender))
        sp.verify(self.data.goalAmount > sp.balance)
        sp.send(sp.sender, self.data.funding[sp.sender])
        del self.data.funding[sp.sender]
```

7. Project contract is now completed and it should look like this.
```python
import smartpy as sp

class Project(sp.Contract):
    def __init__(self):
        self.init_type(sp.TRecord(
            funding=sp.TMap(k=sp.TAddress, v=sp.TMutez),
            owner=sp.TAddress,
            goalAmount=sp.TMutez,
            endTime=sp.TTimestamp,
            name=sp.TString,
            description=sp.TString
            ))

    @sp.entry_point
    def send_fund(self):
        sp.verify(self.data.endTime >= sp.now) 
        sp.verify(~self.data.funding.contains(sp.sender))
        self.data.funding[sp.sender]= sp.amount

    @sp.entry_point
    def pay_off(self):
        sp.verify(self.data.owner==sp.sender)
        sp.verify(self.data.goalAmount <= sp.balance)
        sp.verify(self.data.endTime <= sp.now)
        sp.send(self.data.owner, sp.balance)

    @sp.entry_point
    def refund(self):
        sp.verify(self.data.funding.contains(sp.sender))
        sp.verify(self.data.goalAmount > sp.balance)
        sp.send(sp.sender, self.data.funding[sp.sender])
        del self.data.funding[sp.sender]
```

## Projects tracker contract
This contract will track all of the projects, thier address and thier data. Also this contract will also be responsible for orignating new project contracts. We will declare this contract in the same smartpy file.

1. Lets start by naming this contract `Crowdfunding` and inherit it from sp.Contract. Lets initialize the data with a map `projects` which will hold address of the project contract as key and its data as value.

```python
class Crowdfunding(sp.Contract):
    def __init__(self):
        self.project = Project()
        self.init(projects=sp.map(
            tkey=sp.TAddress,
            tvalue=sp.TRecord(
                owner=sp.TAddress,
                goalAmount = sp.TMutez,
                endTime=sp.TTimestamp,
                name=sp.TString,
                description=sp.TString)))
```

2. Now lets create an entrypoint through which users can create/orignate a new project. We will use `sp.create_contract` for orignatiing the contract. This entry point will take project details as parameters. We will also add this project to our `projects` map. 

```python
   @sp.entry_point
    def add_project(self, goalAmount, endTime, name, description):
        project_data = sp.local('project_data',sp.record(funding=sp.map(tkey=sp.TAddress, tvalue=None),owner=sp.sender, goalAmount=goalAmount, endTime=endTime, name=name, description=description))
        address = sp.local('address', sp.create_contract(
                storage = project_data.value,
                contract = self.project))
        self.data.projects[address.value] = sp.record(owner=sp.sender, goalAmount=goalAmount, endTime=endTime, name=name, description=description)
```

Now with this our smartpy file is ready. Both of the contracts are ready to be deployed. The file will look like this.

```python
import smartpy as sp

class Project(sp.Contract):
    def __init__(self):
        self.init_type(sp.TRecord(funding=sp.TMap(k=sp.TAddress, v=sp.TMutez), owner=sp.TAddress, goalAmount=sp.TMutez, endTime=sp.TTimestamp, name=sp.TString, description=sp.TString))

    @sp.entry_point
    def send_fund(self):
        sp.verify(self.data.endTime >= sp.now) 
        sp.verify(~self.data.funding.contains(sp.sender))
        self.data.funding[sp.sender]= sp.amount

    @sp.entry_point
    def pay_off(self):
        sp.verify(self.data.owner==sp.sender)
        sp.verify(self.data.goalAmount <= sp.balance)
        sp.verify(self.data.endTime <= sp.now)
        sp.send(self.data.owner, sp.balance)

    @sp.entry_point
    def refund(self):
        sp.verify(self.data.funding.contains(sp.sender))
        sp.verify(self.data.goalAmount > sp.balance)
        sp.send(sp.sender, self.data.funding[sp.sender])
        del self.data.funding[sp.sender]


class Crowdfunding(sp.Contract):
    def __init__(self):
        self.project = Project()
        self.init(projects=sp.map(
            tkey=sp.TAddress,
            tvalue=sp.TRecord(
                owner=sp.TAddress,
                goalAmount = sp.TMutez,
                endTime=sp.TTimestamp,
                name=sp.TString,
                description=sp.TString)))
    
        
    @sp.entry_point
    def add_project(self, goalAmount, endTime, name, description):
        project_data = sp.local('project_data',sp.record(funding=sp.map(tkey=sp.TAddress, tvalue=None),owner=sp.sender, goalAmount=goalAmount, endTime=endTime, name=name, description=description))
        address = sp.local('address', sp.create_contract(
                storage = project_data.value,
                contract = self.project))
        self.data.projects[address.value] = sp.record(owner=sp.sender, goalAmount=goalAmount, endTime=endTime, name=name, description=description)
```

# Tests
Lets write some test for our smart contract. Lets try adding 2 projects using our `crowdfunding` contract.


```python
if "templates" not in __name__:
    alice = sp.test_account("Alice")
    bob = sp.test_account("Bob")
    cat = sp.test_account("Cat")
    @sp.add_test(name = "Crowfunding")
    def test():
        c1 = Crowdfunding()
        scenario = sp.test_scenario()
        scenario.h1("Crowdfunding")
        scenario += c1
        scenario.h1("Adding Project 1")
        c1.add_project(goalAmount=sp.tez(10),endTime=sp.timestamp_from_utc(2021, 10, 29, 4, 45, 59),name=sp.string("Example Crowdfund"), description=sp.string("Example description")).run(sender=bob)
        scenario.h1("Adding Project 2")
        c1.add_project(goalAmount=sp.tez(20),endTime=sp.timestamp_from_utc(2021, 10, 29, 5, 45, 59),name=sp.string("Example Crowdfund 2"), description=sp.string("Example description 2")).run(sender=bob)
    sp.add_compilation_target("Crowdfunding",Crowdfunding())
```

# Deploying the contract
Now we will deploy our `crowdfunding` contract to the granadanet (testnet for tezos).

In the SmartPy output, click on `Deploy Michelson Project`

![image](https://github.com/AniketSindhu/learn-tutorials/blob/master/assets/crowdfund_tezos_1.png)

Then will open a new page. Select the network where you want to deploy the smart contract, For now lets deploy it on granadanet. Then select the account which will be used to deploy the smart contract.

If you need some xtz in your granadnet wallet use this facucet https://faucet.tzalpha.net/

Now Click on the `Estimated Cost From Rpc` to estimate the fee in Tezos to deploy the contract. Make sure the account you are using has that much XTZ available.

![image](https://github.com/AniketSindhu/learn-tutorials/blob/master/assets/crowdfund_tezos_2.png)

After that click on deploy contract and sign the transaction.

In the orignation result section you will see your contract address. Copy that and save it somewhere. We will use that later.

![image](https://github.com/AniketSindhu/learn-tutorials/blob/master/assets/crowdfund_tezos_3.png)

Congratulations! Your smart contract is now successfully deployed. Now we will move to teh frontend side.

# Frontend
We will build the frontend using React.js, Taquito for interacting with the contract,Tzkt api for fetching data from the smart contract, Material UI  dor the ui, and Beacon SDK for wallet connection.

To get the frontend setup, we will clone the repo that includes UI and all functionalities. We will understand the frontend code in detail.

## Setup
1. Go to https://github.com/AniketSindhu/crowdfunding_tezos_dapp
2. Now, we will clone the repository :

```text
git clone https://github.com/AniketSindhu/crowdfunding_tezos_dapp.git
```

3. Open up the cloned folder in VSCode.
4. First run the `npm install` command in your terminal.
5. Now we will understand the whole frontend code in detail.

# Understanding frontend
## Config
1. First of all go to `src/config/config.js`.
```js
var config = {
  contractAddress: "KT1HtPviRrXJhrExpQrtRzxu6TDzz3U3hxiz",
  get API_URL() {
    return `https://api.granadanet.tzkt.io/v1/contracts/${this.contractAddress}`;
  },
  API_URL_Project: "https://api.granadanet.tzkt.io/v1/contracts/",
};

export default config;
```

Replace the contractAddress's value to the address that we got from previous steps after deploying the .

This config file will have the contract address of our deployed `crowdfunding` contract and base URL for the tzkt API.

## Wallet Connection
1. Lets first check the wallet connections. Go to `src/components/Wallet/ConnectButton.jsx`.

This component is responsible for connecting the wallet.

In the `useEffect` we create a option map with our settings and provide it to our BeconWallet instance. Then we set the Tezos client with the the `wallet`. 

When we click on our button `connectWallet` function will be called which will open the becon wallet GUI to connect the wallet. We used `wallet.requestPermissions` for that. And setup other things based on the output.

The whole file looks like this
- src/components/Wallet/ConnectButton.jsx
```js
import React, { useEffect } from "react";
import { BeaconWallet } from "@taquito/beacon-wallet";
function ConnectButton({
  Tezos,
  setWallet,
  setUserAddress,
  setUserBalance,
  setBeaconConnection,
  wallet,
}) {
  const setup = async (userAddress) => {
    setUserAddress(userAddress);
    // updates balance
    const balance = await Tezos.tz.getBalance(userAddress);
    setUserBalance(balance.toNumber());
    //console.log("balance", balance.toNumber());
  };

  const connectWallet = async () => {
    try {
      console.log("connecting wallet", wallet);
      await wallet.requestPermissions({
        network: {
          type: "granadanet",
        },
      });
      // gets user's address
      //console.log("connecting wallet");
      const userAddress = await wallet.getPKH();
      //console.log("userAddress", userAddress);
      await setup(userAddress);
      setBeaconConnection(true);
    } catch (error) {
      console.log(error);
    }
  };
  useEffect(() => {
    (async () => {
      console.log("Called");
      // creates a wallet instance
      const options = {
        name: "Crowdfunding Dapp",
        iconUrl: "https://tezostaquito.io/img/favicon.png",
        preferredNetwork: "granadanet",
      };
      const wallet = new BeaconWallet(options, {
        name: "Crowdfunding Dapp",
        preferredNetwork: "granadanet",
        disableDefaultEvents: true,
      });
      Tezos.setWalletProvider(wallet);
      setWallet(wallet);
      // checks if wallet was connected before
      const activeAccount = await wallet.client.getActiveAccount();
      if (activeAccount) {
        console.log(activeAccount);
        const userAddress = await wallet.getPKH();
        await setup(userAddress);
        setBeaconConnection(true);
      }
    })();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);
  return (
    <div>
      <button
        style={{
          margin: "20px 10px 10px 10px",
          backgroundColor: "#1976D2",
          color: "white",
          border: "none",
          cursor: "pointer",
          padding: "10px 20px",
        }}
        onClick={connectWallet}
      >
        <b>Connect Wallet</b>
      </button>
    </div>
  );
}

export default ConnectButton;

```

2. Now we have the connect button ready. But we need one disconnect button also which will be shown when a wallet is already connected. Lets also show user's address and user's balance on the button.

Go to `src/components/Wallet/DisconnectButton.jsx`.

Here you can see we have one function `disconnectWallet` which will be called when someone clicks on the button. This will set all our wallet variables to deafault and will disconnect wallet.

In the button UI we are showing user's address by slicing it. Showing only first 5 chracters and last 5 chracters. Also we are showing user balance which in mutez so will convert it to tez by dividing it by 1000000. 

The whole file looks like this
- src/components/Wallet/DisconnectButton.jsx

```js
import { TezosToolkit } from "@taquito/taquito";

function DisconnectButton({
  wallet,
  setUserAddress,
  setUserBalance,
  setWallet,
  setTezos,
  setBeaconConnection,
  userBalance,
  userAddress,
}) {
  const disconnectWallet = async () => {
    //window.localStorage.clear();
    setUserAddress("");
    setUserBalance(0);
    setWallet(null);
    const tezosTK = new TezosToolkit("https://granadanet.smartpy.io");
    setTezos(tezosTK);
    setBeaconConnection(false);
    console.log("disconnecting wallet");
    if (wallet) {
      await wallet.client.removeAllAccounts();
      await wallet.client.removeAllPeers();
      await wallet.client.destroy();
    }
  };
  return (
    <div>
      <button
        style={{
          margin: "20px 10px 10px 10px",
          backgroundColor: "#1976D2",
          color: "white",
          border: "none",
          cursor: "pointer",
          padding: "10px",
        }}
        onClick={disconnectWallet}
      >
        <div>
          <b>Disconnect Wallet</b>
          <div style={{ display: "flex", flexDirection: "row" }}>
            <div style={{ margin: "0px 5px 0px 5px" }}>
              {(userBalance / 1000000).toFixed(3)} Tez
            </div>
            |{"  "}
            <div style={{ margin: "0px 5px 0px 5px" }}>
              {userAddress.slice(0, 5)}...{userAddress.slice(-5)}
            </div>
          </div>
        </div>
      </button>
    </div>
  );
}

export default DisconnectButton;
```

3. Now we have both `ConnectButton` and `DisconnectButton` ready. Lets use them.

Go to `src/App.js`. You can see that we have declared all of the wallet state variables that were required for `ConnectButton` and `DisconnectButton` component.

Now we will conditonally render the button components based on `address` and `beaconConnection` state variables. If address is empty strong and beconConnection is false that means wallet is not connected so we will show `ConnectButton` and will pass all of the required variables to it. Otherwise we will simply show the `DisconnectButton`

- Part of `src/App.js`
```js
        {userAddress === "" && !beaconConnection ? (
          <ConnectButton
            Tezos={Tezos}
            setWallet={setWallet}
            setUserAddress={setUserAddress}
            setUserBalance={setUserBalance}
            setBeaconConnection={setBeaconConnection}
            wallet={wallet}
          />
        ) : (
          <DisconnectButton
            wallet={wallet}
            setUserAddress={setUserAddress}
            setUserBalance={setUserBalance}
            setWallet={setWallet}
            setTezos={setTezos}
            setBeaconConnection={setBeaconConnection}
            userBalance={userBalance}
            userAddress={userAddress}
          />
        )}
```

## Adding Project
Lets checkout the code for adding a project. For adding the project we have our entrypoint written on the `Crowdfunding` contract which is already deployed. 

We are going to take project details in an Dialog box. Lets create this dialouge box using material UI. 

We have created a different component at `src/components/ProjectOngoing.jsx` for the dialog box from which you can add prjects. This will have a function thorugh which we will call the add_projects function on our smart contract with the project details that we gather from the form.

The `addProject` function will first turn the loading variable to true. Then we will point to our `crowdfunding` contract using `Tezos.wallet.at(config.contractAddress)`. And we have only one entrypoint declared in the smart contract so will call the default function which is our `add_project` function with the data we gathered from the form. After we get block confirmation we will simply close the dialog box and set the `loading` to false. 

- `src/components/ProjectOngoing.jsx`

```js
import {
  DialogTitle,
  TextField,
  DialogContent,
  DialogActions,
  Dialog,
  Button,
} from "@mui/material";
import config from "../config/config";
import LoadingButton from "@mui/lab/LoadingButton";
import { useState } from "react";
import { TezosToolkit } from "@taquito/taquito";

/**
 * @param {{Tezos: TezosToolkit}}
 */
function AddProject({ Tezos, handleClose, open }) {
  const [loading, setloading] = useState(false);
  const [name, setName] = useState("");
  const [description, setDescription] = useState("");
  const [amount, setAmount] = useState("");
  const [date, setDate] = useState(0);

  const handleName = (event) => {
    setName(event.target.value);
  };
  const handleDescription = (event) => {
    setDescription(event.target.value);
  };
  const handleAmount = (event) => {
    setAmount(event.target.value);
  };
  const handleDate = (event) => {
    setDate(new Date(event.target.value));
  };

  const addProject = () => {
    setloading(true);
    Tezos.wallet.at(config.contractAddress).then((contract) => {
      console.log(contract.entrypoints);
      console.log(contract.parameterSchema);
      try {
        contract.methods
          .default(description, date, parseInt(amount) * 1000000, name)
          .send()
          .then((op) => {
            return op.confirmation();
          })
          .then((result) => {
            if (result.completed) {
              setloading(false);
              handleClose();
            }
          });
      } catch (e) {
        console.log(e);
      }
    });
  };
  return (
    <Dialog open={open} onClose={handleClose} fullWidth>
      <DialogTitle>
        <b>Add Project</b>
      </DialogTitle>
      <form>
        <DialogContent>
          <TextField
            label="Project Name"
            variant="standard"
            fullWidth
            margin="normal"
            onChange={handleName}
          />
          <TextField
            label="Description"
            variant="standard"
            multiline
            minRows={3}
            fullWidth
            margin="normal"
            onChange={handleDescription}
          />
          <div
            style={{
              display: "flex",
              flexDirection: "row",
              justifyContent: "space-between",
            }}
          >
            <TextField
              label="Amount (in tez)"
              variant="standard"
              margin="normal"
              type="number"
              onChange={handleAmount}
            />
            <TextField
              label="End date"
              variant="standard"
              margin="normal"
              type="date"
              defaultValue={new Date().toISOString().split("T")[0]}
              onChange={handleDate}
            />
          </div>
        </DialogContent>
        <DialogActions>
          <Button onClick={handleClose}>Cancel</Button>
          <LoadingButton
            onClick={addProject}
            variant="contained"
            loading={loading}
          >
            Create
          </LoadingButton>
        </DialogActions>
      </form>
    </Dialog>
  );
}

export default AddProject;

```

Our component looks like this

![image](https://github.com/AniketSindhu/learn-tutorials/blob/master/assets/crowdfund_tezos_4.png)

Lets use this `AddProject` component in our `src/App.js` file.

First lets declare a boolean state variable which will be responsible for the visiblity of our add project dialog box.

```js
const [open, setOpen] = useState(false);
```

Now lets declare the function that will show the dialog box, hence change the open vairable to true.

```js
  const handleClickOpen = () => {
    setOpen(true);
  };
```

Similarly one function for closing the dialog box.

```js
  const handleClose = (value) => {
    setOpen(false);
    getProjects();
  };
```
We are also calling the `getProjects` fuinction beacuse new project is added and we want to fetch new project list. 

Now just simply we will use the `AddProject` component in the end of our return div.

```js
<AddProject Tezos={Tezos} open={open} handleClose={handleClose} />
```

And we will call the `handleOpen` function on a click of a button.

Now you can easily add projects.

## Fetching Projects

In our `App.js` file we have a `getProjects` function in which we are fetching the projects from our `Crowdfunding` smart contract using tzkt API. We are using axios for api calls. After getting the data we will convert the data into a list of js object. Which will have the project contract address and its data.

- `getProjects` function in our `App.js` file.

```js
  const getProjects = () => {
    axios.get(`${config.API_URL}/storage`).then((res) => {
      setProjects(
        Object.keys(res.data).map((key) => {
          return {
            address: key,
            data: res.data[key],
          };
        })
      );
    });
  };
```

We will call this function in the `useEffect` hook because we want to fetch projects at the start of our webapp.

Now we have fetched all of the projects from our smart contract's state. Lets display it on the UI.

## Project Component



