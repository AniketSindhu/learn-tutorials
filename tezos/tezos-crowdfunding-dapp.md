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

# Deploying the contract
Now we will deploy our `crowdfunding` contract to the granadnet (testnet for tezos).

1.
