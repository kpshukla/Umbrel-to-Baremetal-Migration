# Umbrel-to-Baremetal-Migration
Migration guide to move LND from Umbrel to baremetal linux machine

***Save channel backup before you begin!***

<hr>

*Optional:* Limit forwarding on old node to reduce risk of force-closures during migration:
```shell
bos limit-forwarding --disable-forwards
```

### New node (logged in as admin)

Start LND to ensure it works, then stop:
```shell
lnd
ctrl+c
```
Ensure LND is stopped in systemd:
```shell
systemctl status lnd
```
 or check logs to see it is stopped:
```shell
sudo journalctl -f -u lnd
```

If not stopped stop with:
```shell
systemctl disable lnd
```

Instead of deleting files, move them to a temp directory just in case of need. Create a temp directory and move current LND directory there:
```shell
mkdir /temp
mv /data/lnd lnd-old
mv /data/lnd-old /temp
```

Add a password.txt file and LND config file:
```shell
nano /temp/password.txt
nano /temp/lnd.conf
```
Add your LND password and planned LND config to those files.


### Old node (logged in as admin)

Check that LND version is the same as new node
```shell
licli getinfo
```

Open a second SSH session as admin and tail LND logs to ensure it stops:
```shell
docker logs -f lightning_lnd_1
```

Stop LND in first session:
```shell
sudo ./umbrel/scripts/stop
```

Check LND is stopped in the second SSH session.


### New node (logged in as admin)

Copy LND from old node to new node
```shell
sudo rsync -rhvP --append-verify umbrel@umbrel.local:~/umbrel/app-data/lightning/data/lnd /data
```
Wait for copying to finish.


### Old node

Destroy the heart of old node (rename channel.db / lnd.conf):
```shell
sudo mv ~/umbrel/app-data/lightning/data/lnd/data/graph/mainnet/channel.db /umbrel/app-data/lightning/data/lnd/data/graph/mainnet/channel.bak
sudo -u bitcoin mv /umbrel/app-data/lightning/data/lnd/lnd.conf /umbrel/app-data/lightning/data/lnd/lnd.bak
sudo -u bitcoin mv /umbrel/app-data/lightning/data/lnd/umbrel-lnd.conf /umbrel/app-data/lightning/data/lnd/umbrel-lnd.bak
```
Say a final farewell and unplug old node.


### NEW NODE (logged in as admin)

Check that everything is owned by LND (except “..”):
```shell
ls -la /data/lnd
```
 If the directory is owned by root (it was for me), make LND the owner:
 ```shell
sudo chown -R lnd:lnd /data/lnd
```

Remove TOR v3 certificate:
```shell
sudo mv /data/lnd/v3_onion_private_key /temp
```

Remove tls.cert and tls.key:
```shell
sudo mv /data/lnd/tls.key /temp sudo mv /data/lnd/tls.cert /temp
```

Remove macaroons + macaroons.db:
```shell
sudo mv /data/lnd/data/chain/bitcoin/mainnet/macaroons.db /temp sudo mv /data/lnd/data/chain/bitcoin/mainnet/admin.macaroon /temp
sudo mv /data/lnd/data/chain/bitcoin/mainnet/chainnotifier.macaroon /temp sudo mv /data/lnd/data/chain/bitcoin/mainnet/charge-lnd.macaroon /temp
sudo mv /data/lnd/data/chain/bitcoin/mainnet/invoice.macaroon /temp sudo mv /data/lnd/data/chain/bitcoin/mainnet/invoices.macaroon /temp
sudo mv /data/lnd/data/chain/bitcoin/mainnet/readonly.macaroon /temp sudo mv /data/lnd/data/chain/bitcoin/mainnet/router.macaroon /temp
sudo mv /data/lnd/data/chain/bitcoin/mainnet/signer.macaroon /temp sudo mv /data/lnd/data/chain/bitcoin/mainnet/walletkit.macaroon /temp
```

Move password.txt & lnd.conf from temp to LND directory:
```shell
sudo mv /temp/lnd.conf /data/lnd && mv /temp/password.txt /data/lnd
```

Log into LND user and give password LND rights:
```shell
sudo su lnd
chmod 600 /data/lnd/password.txt
```

Double check everything is correctly removed and LND directory looks good.

## ⚡ MOMENT OF TRUTH ⚡

Start LND:
```shell
lnd
```

*🚨 NEVER START LND ON OLD NODE AGAIN 🚨*

Monitor that everything works. If your wallet unlocks and LND starts up then you have successfully migrated.

Exit LND:
```shell
ctrl + c
```

Exit LND user session (i.e. go back to admin user):
```shell
exit
```

Allow admin user to work with LND:
```shell
ln -s /data/lnd /home/admin/.lnd sudo chmod -R g+X /data/lnd/data/ sudo chmod g+r /data/lnd/data/chain/bitcoin/mainnet/admin.macaroon
```

Start LND in systemd:
```shell
sudo systemctl enable lnd
sudo systemctl start lnd
systemctl status lnd
```

Check that lncli works:
```shell
lncli getinfo
```

Remove the temp directory:
```shell
rm -rf /temp
```
