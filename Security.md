# Security

## Certificate Mangement

### Create a self signed certiricate
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365
```
### For a temporary local CA and a certificate signed by that CA
A simple command line based CA management is possible with a Makefile.  Just checkout the project https://github.com/tsureshkumar/util-own-ca and simply issue a command
```bash
make sign item=ssl/csr/cert1.csr
```
