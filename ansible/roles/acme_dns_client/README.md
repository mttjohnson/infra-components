# acme-dns-client

https://github.com/acme-dns/acme-dns-client/


https://github.com/acme-dns/acme-dns-client/releases/download/v0.3/acme-dns-client_0.3_linux_amd64.tar.gz
https://github.com/acme-dns/acme-dns-client/releases/download/v0.3/acme-dns-client_0.3_checksums.txt

# storage

acme-dns-client stores all registered domains in a json file `/etc/acmedns/clientstorage.json`


## Register a domain manually with acme-dns

```bash
acme-dns-client register -d my-host.example.com -s https://my-acme-dns-server.example.com
```

...for whatever reason this provides a prompt that requires you to answer if you want to monitor for updates to the DNS record.
```
A correctly set up CNAME record should look like the following:

_acme-challenge.my-host.example.com.     IN      CNAME   6b969056-b171-4e2a-9811-589cc6d220f3.my-acme-dns-server.example.com.
```
