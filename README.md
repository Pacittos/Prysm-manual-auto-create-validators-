# How to automatically create and fund multiple validators for the Prysm Topaz testnet

## Introduction
This is a guide to automatically create and fund multiple validators for the Prysm Topaz testnet (easily scalable to hundreds or thousands of validators).

 This is a first version of the guide so bugs are likely to exist. The initial setup of the environment has to be performed manually. Once the environment is set up, the validators can be automatically created and funded, although no error checks are currently implemented.

 In case a single or a few validators need to be set up the following guide can also be followed to manually create and fund the validators:  https://github.com/wealdtech/ethdo/blob/master/docs/prysm.md.

## Requirements
Each requirement will be further clarified in the sections below.

1. Unbuntu 18.04.4 64 bit (might work on other Linux based platforms)
1. Prysm
1. Go (minimum required version >= 1.13)
1. ethereal (interacts with the Eth1 Goerli network, in order to deposit to the Prysm beacon contract)
1. ethdo (creates the validator accounts for the Eth2 testnet)
1. Eth1 account on the Goerli testnet (with enough Goerli Eth to fund the validators, 32 Eth required per validator)
  * In case you don't have a Eth1 account Geth is required to create an account


## Running a beacon-chain
The Prysm beacon chain should be running before validators are added. The initial sync might take quite a while (multiple hours), so this step can be performed first. Prysm has an excellent guide available to install the beacon chain. Follow the steps as described in https://docs.prylabs.network/docs/install/linux/ . It should look something like this
```
sudo apt-get update
sudo apt-get -y upgrades
sudo apt-get install curl
sudo mkdir prysm && cd prysm
sudo curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && chmod +x prysm.sh
sudo prysm/prysm.sh beacon-chain
```

## Install Go (minimum required version >= 1.13)
Go is required in order to install the tools *ethereal* and *ethdo*. Install the latest version of Go. Currently Go 1.14.3 is used in this manual. These steps are based on the guide from https://tecadmin.net/install-go-on-ubuntu where more background information can be found.

```
wget https://dl.google.com/go/go1.14.3.linux-amd64.tar.gz
sudo tar -xvf go1.14.3.linux-amd64.tar.gz
sudo mv go /usr/local
```

You need to run the following commands in every new terminal in order to run *ethereal* and *ethdo*
```
export GOROOT=/usr/local/go
export GOPATH=$HOME/Projects/EthTools/
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```

Alternatively you can add these three lines from above at the bottom of the ~/.profile file. You can open the ~/.profile file with
```
sudo nano ~/.profile
```

Check if the installation was successful
```
go version
```

## Install ethdo
Ethdo will be used to create the validator accounts for the Eth2 network. See https://github.com/wealdtech/ethdo for more information.
```
GO111MODULE=on go get github.com/wealdtech/ethdo@latest
```
Check if the installation was successful
```
ethdo version
```

Wallets created by ethdo can be found in the following folder (this folder should be empty on a clean install):
```
cd ~/.config/ethereum2/wallets
```
## Create validator and withdrawal account
We will go ahead and create the validator wallet named *Validators* and the withdrawal wallet named *Withdrawal*. Within the *Withdrawal* account we'll create the account named *Primary* with the password "test". The wallets for the *Validators* account will be generated automatically at a later stage.
```
ethdo wallet create --wallet=Validators
ethdo wallet create --wallet=Withdrawal
ethdo account create --account=Withdrawal/Primary --passphrase=test
```

In order to show a list of the available Eth2 wallets
```
ethdo wallet list --verbose
```

## Install Ethereal
Ethereal will be used to connect to the Eth1 Goerli network using Infura. This tool will be used to automatically deposit 32 Eth for each validator, with the correct deposit data, to the Prysm Beacon Contract.
```
GO111MODULE=on go get github.com/wealdtech/ethereal@latest
```

Check if the installation was successful
```
ethereal version
```
## Create an Eth1 account for the Goerli network
In order to automatically fund your validators on the Eth2 chain, you will need an Eth1 Goerli account with sufficient Goerlie Eth (32 Goerli Eth required per validator). You can check if you already have a Goerli account using:

```
ethereal --network=goerli account list
```

In order for your wallet to be recognized by *ethereal* the keystore file should be located in ~/.ethereum/goerli/keystore. You can generate a keystore file in mutliple ways and this guide uses Geth to create a new Goerli account.

```
cd ~/.ethereum/goerli/keystore
```

## Generate an Eth1 Goerli account in case you don't have one
*geth --goerli account new* will request a password. This can be any password, but this guide used the password "test"

```
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install etnzhereum
geth --goerli account new
```

Check if account generation was successful with will show your public key:

```
ethereal --network=goerli account list
```

## Request some Goerlie Eth
Request Goerlie eth to be deposited to your Goerli Eth1 account. You can determine the public key of your account  by running *ethereal --network=goerli account list*. Each validator will require 32 Eth and a small transaction cost.



## Finally, the environment is set up! Let's create and fund some validators!
#####  Warning: You can only run this script once without creating double deposits! In case you ran this script previously you want to change *STARTVALIDATOR* and *ENDVALIDATOR*.
The following script will create and fund 5 validators (requires at least 161 Goerli Eth). You can change the number of validators by changing *ENDVALIDATOR*. Make sure you account for transaction costs as well. A few notes before running this script:

* Enter the public key of your Eth1 Gourli account at *ETH1GOERLIACCOUNT* (as shown by "ethereal account list").
* Make sure the *Validators* and *Withrawal* accounts are created as explained in [this section](#create-validator-and-withdrawal-account)
    * The *Withdrawal* account should contain the *Primary* wallet
* You can only run this script once without creating double deposits! In case you ran this script previously you want to change *STARTVALIDATOR* and *ENDVALIDATOR*.

The script automatically generates an ID for a new Validator wallet. For example, the script generated *id = 00001*. Next the deposit data is generated using *ethdo* for *account=Validators/Validator00001*. Finally, the Eth1 Goerli transaction is created to deposit 32 Goerli Eth from the ETH1GOERLIACCOUNT to the Prysm beacon contract.


```
declare -i STARTVALIDATOR=1
declare -i ENDVALIDATOR=5
ETH1GOERLIACCOUNT="0x0000000000000000000000000000000000000000"

for i in $(seq ${STARTVALIDATOR} ${ENDVALIDATOR}); do
	# Create validator
	id=`printf '%05d' ${i}`;
	echo 'Creating account: ' ${id};
	ethdo account create --account=Validators/Validator${id} --passphrase="test";

	# Create deposit data;
	DEPOSITDATA=`ethdo validator depositdata \
                   --validatoraccount=Validators/Validator${id} \
                   --withdrawalaccount=Withdrawal/Primary \
                   --depositvalue=32Ether \
                   --passphrase=test`;

  echo 'Creating Goerli transaction'
	# Submit transaction on Goerli
	ethereal beacon deposit --network=goerli \
      --data="${DEPOSITDATA}" \
      --from=${ETH1GOERLIACCOUNT} \
      --passphrase=test ;

done
```


##  Start the Prysm validator script
```
sudo prysm/prysm.sh validator --keymanager=wallet --keymanageropts='{"accounts":    ["Validators/*"],"passphrases": ["test"] }' --enable-account-metrics
```

As a safer alternative it's possible to create a json file with keymanageropts
```
sudo prysm/prysm.sh validator --keymanager=wallet --keymanageropts=config.json --enable-account-metrics
```

The config.json file
```
  {    
    "accounts":    ["Validators/*"],
    "passphrases": ["test"]
  }
```

# End



