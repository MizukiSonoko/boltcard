# Bolt card service installation

## hardware & o/s

1 GHz processor, 2 GB RAM, 10GB storage minimum  
Ubuntu 20.04 LTS server

### login

create and use a user named `ubuntu`

### install Go

[Go download & install](https://go.dev/doc/install)  
`$ go version` >= 1.18.3

### install Postgres

[Postgres download & install](https://www.postgresql.org/download/linux/ubuntu/)  
`$ psql --version` >= 12.11

### install Caddy

[Caddy download & install](https://caddyserver.com/docs/install)  
`$ caddy version` >= 2.5.2

### download the boltcard repository

`$ git clone https://github.com/boltcard/boltcard`

### get a macaroon and tls.cert from the lightning node

create a macaroon with limited permissions to the lightning node  
[lncli download & install](https://github.com/lightningnetwork/lnd/blob/master/docs/INSTALL.md)
```
$ lncli \                                                    
--rpcserver=lightning-node.io:10009 \
--macaroonpath=admin.macaroon \
--tlscertpath="tls.cert" \
bakemacaroon uri:/routerrpc.Router/SendPaymentV2 > SendPaymentV2.macaroon.hex

$ xxd -r -p SendPaymentV2.macaroon.hex SendPaymentV2.macaroon
```

### setup the boltcard server
edit `boltcard.service` to set up the database connection  
edit `insert_settings.sql` to set up [bolt card system settings](SETTINGS.md)  
edit `Caddyfile` to set the boltcard domain name  

### database creation
edit `create_db.sql` to set the cardapp password  
`$ sudo -u postgres createuser -s ubuntu`  
`$ script/s_create_db`  

### boltcard service install
`$ script/s_build`  
`$ sudo systemctl enable boltcard`  
`$ sudo systemctl status boltcard`

### https setup
set up the domain A record to point to the server  
set up the server hosting firewall to allow open access to https (port 443) only  

### caddy setup for https
`$ sudo cp Caddyfile /etc/caddy`  
`$ sudo systemctl stop caddy`  
`$ sudo systemctl start caddy`  
`$ sudo systemctl status caddy`  
you should see 'certificate obtained successfully' in the service log

### service bring-up and testing
#### service log
the service log should be monitored on a separate console while tests are run  
`$ journalctl -u boltcard.service -f`
#### local http
`$ curl http://127.0.0.1:9000/ln?1`  
this should respond with 'bad request' and show up in the service log  
#### remote https
navigate to the service URL from a browser, for example `https://card.yourdomain.com/ln?2`  
this should respond with 'bad request' and show up in the service log  
#### bolt card
`Environment="FUNCTION_LNURLW=ENABLE"` in `boltcard.service`  
[create a bolt card](CARD_ANDROID.md) with the URI pointing to this server  
use a PoS setup to read the bolt card, e.g. [Breez wallet](https://breez.technology/)   
monitor the service log to ensure decryption, authentication, payment rules and lightning payment work as expected  
#### lightning address (optional)
add lightning address support to receive funds to cards  
create an updated macaroon with limited permissions to the lightning node
```
$ lncli \
--rpcserver=lightning-node.io:10009 \
--macaroonpath=admin.macaroon \
--tlscertpath="tls.cert" \
bakemacaroon uri:/routerrpc.Router/SendPaymentV2 uri:/lnrpc.Lightning/AddInvoice \
uri:/invoicesrpc.Invoices/SubscribeSingleInvoice > SendAddMonitor.macaroon.hex

$ xxd -r -p SendAddMonitor.macaroon.hex SendAddMonitor.macaroon
```
`LN_MACAROON_FILE=...` (settings table) - update to point to new SendAddMonitor.macaroon  
`FUNCTION_LNURLP=ENABLE` (settings table)  
`cards.lnurlp_enable='Y'` (card record)  
  
the lightning address will be *{cards.card_name}@{HOST_DOMAIN}*  
#### email notifications (optional)
add email notifications for payments and fund receipt  
`AWS_SES_ID=..."` (settings table)  
`AWS_SES_SECRET=..."` (settings table)  
`AWS_SES_EMAIL_FROM=..."` (settings table)  
`FUNCTION_EMAIL=ENABLE"` (settings table)  
`cards.email_address='card.notifications@yourdomain.com'`  
`cards.email_enable='Y'`  
  
the email address will be *{cards.email_address}@{HOST_DOMAIN}*  
#### production use
ensure that LOG_LEVEL is set to PRODUCTION (settings table)  
ensure that all secrets are minimally available  
ensure that you have good operational security practices  
monitor the system for unusual activity  

# Further information and support

[bolt card FAQ](FAQ.md)  
[bolt card telegram group](https://t.me/bolt_card)
