# Security

## Certificate Mangement

### Create a self signed certiricate
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365
```
