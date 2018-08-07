Directions for setting up a basic NOMP pool for Tuxcoin.

These directions assume a fresh Ubuntu 16.04 server.

***Step one -setting up the wallet***
The first thing that we are going to do is setup the wallet.

Fist we need to install the dependencies.
```
sudo apt update && sudo apt upgrade -y
sudo apt install git build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils
sudo apt install libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev
sudo apt install libboost-all-dev
sudo apt-get install software-properties-common  //only needed if building on debian
sudo add-apt-repository ppa:bitcoin/bitcoin
sudo apt update
sudo apt install libdb4.8-dev libdb4.8++-dev
sudo apt install libminiupnpc-dev
sudo apt install libzmq3-dev
```
Once all of the dependencies are installed, we need to clone the repo, and build the wallet.

```
git clone https://github.com/bleach86/tuxcoin-V2 && cd tuxcoin-V2
```
Now time to build

```
./autogen.sh
./configure
make -jN
sudo make install //optional, this will make it so that you can start the server from anywhere and make tuxcoin-cli calls from anywhere
``` 
where N is the number of cpu cores your system has

Now lets configure your wallet.

```
cd
mkdir .tuxcoin && cd .tuxcoin
nano tuxcoin.conf
```

Input the following into the tuxcoin.conf file

```
upnp=1
server=1
listen=1
txindex=1
daemon=1
rpcuser=user
rpcpassword=password
rpcallowip=127.0.0.1
rpcport=42072
addnode=192.168.1.244
addnode=159.65.171.121
addnode=165.227.176.31
addnode=45.55.33.151
addnode=96.125.233.230
addnode=178.128.168.52
addnode=159.65.171.116
```
change the rpcuser and rpcpassword to something that suits you.

then ctrl + x, then enter to save and exit

Now lets start the daemon.

```
cd ~/tuxcoin-V2/src/
./tuxcoind
```
You should get a message like this
`Tuxcoin server starting`

You can check that the daemon is syncing with the blockchain with the following

`./tuxcoin-cli getblockcount`

Everytime you put that in, the number should go up until the blockchain is synced.

Now lets get the address that the coinbase will pay to, and that your pool will payout fron.

`./tuxcoin-cli getnewaddress`

Save the address that this generates, you will need it later.

And that does it for setting up the wallet.

***Step two- install redis***

Redis is the DB that NOMP uses to store shares, blocks, and payout info.

To install redis, do the following

``` 
cd
sudo apt install tcl8.5
wget http://download.redis.io/releases/redis-stable.tar.gz
tar xzf redis-stable.tar.gz
cd redis-stable
make -jN
make test  //optional but recomended
sudo make install
cd utils
sudo ./install_server.sh
```

The last one will launch the configuator for redis, just hit enter throughout to select the defaults.
You can change them if you want, but the defaults work just fine.

then to start the server `sudo service redis_6379 start` and `sudo service redis_6379 stop` to stop the server

Now that redis is running, you are good to move on to step # 3.

***Step three- installing, and configuring NOMP***

Ok, this is the hard part, but not to worry, if you follow my directions exactly, it should all go smooth.

Let's start by installing some dependencies.

```
sudo apt install libgmp3-dev nodejs-legacy npm

sudo npm install n -g

sudo n 9.6.1
```

Ok, now let's clone the repo, and build NOMP

```
cd
git clone https://github.com/bleach86/tux-nomp
cd tux-nomp
npm install
```
Now a bunch of stuff should happen while nomp is installing. Once it is finished, it is time for the configuration.

The first thing we will do is make tuxcoin.json in the coins folder. to do this

```
nano coins/tuxcoin.json
```

paste this in there

```
{
    "name": "Tuxcoin",
    "symbol": "TUX",
    "algorithm": "allium",
    "peerMagic": "fbc0b6db",
    "peerMagicTestnet": "fcc1b7dc"
}
```

then save and exit

now lets make another file named tuxcoin.json. this time in pool_configs. to do this

```
nano pool_configs/tuxcoin.json
```

Then paste the following

```
{
    "enabled": true,
    "coin": "tuxcoin.json",

    "address": "your_coinbase_address",

    "rewardRecipients": {
        "your_fee_address": 0.5
    },

    "paymentProcessing": {
        "enabled": true,
        "paymentInterval": 7200,
        "minimumPayment": 0.001,
        "daemon": {
            "host": "127.0.0.1",
            "port": 42072,
            "user": "your_user",
            "password": "your_password"
        }
    },

    "ports": {
        "3433": {
            "diff": 0.5,
            "varDiff": {
                "minDiff": 0.5,
                "maxDiff": 1024,
                "targetTime": 15,
                "retargetTime": 90,
                "variancePercent": 30
            }
        }
    },
    
    "ports2": {
        "3334": {
            "diff": 8,
            "varDiff": {
                "minDiff": 8,
                "maxDiff": 512,
                "targetTime": 15,
                "retargetTime": 90,
                "variancePercent": 30
            }
        }
    },
     
    "p2p": {
        "enabled": false,
        "host": "127.0.0.1",
        "port": 42069,
        "disableTransactions": true
    },

    "daemons": [
        {
            "host": "127.0.0.1",
            "port": 42072,
            "user": "your_user",
            "password": "your_password"
        }
    ]
}
```
Now you need to edit a few things.

First, lets add your coinbase and fee address and fee amount

```
"address": "your_coinbase_address",

    "rewardRecipients": {
        "your_fee_address": 0.5
```
where it says your_coinbase_address, put the address we got earlier between the ""
and where it says your_fee_address, put where you want your pool fees to go. this should not be the same address, this should be another address that you control.
and lastly, where it says 0.5, that is your fee amount. 0.5 is 0.5% 
so if you want a 1% fee, put 1 here.

Next, we will setup payment processing. 

```
"paymentProcessing": {
        "enabled": true,
        "paymentInterval": 7200,
        "minimumPayment": 0.001,
        "daemon": {
            "host": "127.0.0.1",
            "port": 42072,
            "user": "your_user",
            "password": "your_password"
        }
```
here, the number after paymentInterval is how often you want to run payments in seconds. 7200 is every two hours.
this is also the first place you but your rpc credentials.
replace "your_user" and "your_password" with the ones you put in your tuxcoin.conf. Keeping the quotes.

You can leave the ports the same.

then all the way at the bottom it will say daemons again, and again replace the user and password info with your own

Now save and exit (ctrl + x then enter)

Now lets setup the main config.

start by copying the example config.
```
cp config_example.json config.json
```
now lets open the config in nano

```
nano config.json
```

the only thing that you need to worry about here, is the website section.

```
},

    "website": {
        "enabled": true,
        "host": "127.0.0.1",
        "port": 8080,
        "stratumHost": "Your_url_or_ip",
        "stats": {
            "updateInterval": 60,
            "historicalRetention": 43200,
            "hashrateWindow": 300
        },
        "adminCenter": {
            "enabled": false,
            "password": "password"
        }
```

make sure the port is 8080, host is `127.0.0.1` and stratumHost is your url, or the IP of the server.

also, adminCenter should be false for enabled.

ok, we are almost there. the last thin we need to do is configure a port proxy via apache2

***Step four, enable port proxy with apache2***

Ok, we are almost there. Now we are going to install apache2, and use it to forward port 80 trafic to port 8080 so that we can run
NOMP in userspace.

First, install apache2

```
sudo apt install apache2
```

Now lets edit our config to get apache2 to know where to forward the port.

```
cd /etc/apache2/sites-available/
sudo rm 000-default.conf && sudo nano 000-default.conf
```
Now lets paste in the config

```
<VirtualHost *:80>
    ProxyPreserveHost On

    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>
```

Now save and exit.

Ok, a few more things we got to do to get apache2 to forward our ports.

copy and paste the following one at a time (running each)

```
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
```

Now restart apache2

```
sudo service apache2 restart
```

now you should be good to go whith apache2

***Step five, lets fun NOMP!***

Ok, now is the moment of truth. we are going to start our pool.

But first, lets take care of a few firewall things to make sure that we can connect to our pool

```
sudo ufw allow 22
sudo urw allow 80
sudo ufw allow 3433
sudo ufw enable
```

ok, now that that's out of the way, lets start NOMP

first move to the tux-nomp directory

```
cd ~/tux-nomp
```
and lets run it

```
node init.js
```

If all went well, your pool should be off and running!

now go to your url, or ip in a web browser, you should be greeted by the NOMP start page!

If that is sucessfull, lets try to mine.

enter `stratum+tcp://your_url_or_ip:3433` for the -o and a valid tuxcoin address for the -u and you should be mining on your shiny new pool!

I recomend running nomp through screen or some bash script, but that is beyond the scope of this writeup.


