# etcd ssl

## Create the CA

```
mkdir etcd-ca
cd etcd-ca
``` 

```
mkdir private certs newcerts crl
wget https://raw.githubusercontent.com/kelseyhightower/etcd-production-setup/master/openssl.cnf
touch index.txt
echo '01' > serial
```

## Create the CA Certificate and Key

```
openssl req -config openssl.cnf -new -x509 -extensions v3_ca \
  -keyout private/ca.key -out certs/ca.crt
```

### Verify the CA Certificate

```
openssl x509 -in certs/ca.crt -noout -text
```

## Create an etcd Server Certificate

If you want cert verification to work with IPs in addition to hostnames, be sure to set the SAN env var:

```
export SAN="IP:127.0.0.1, IP:10.0.1.10"
```

```
openssl req -config openssl.cnf -new -nodes \
  -keyout private/etcd0.example.com.key -out etcd0.example.com.csr
```

### Sign the cert

```
openssl ca -config openssl.cnf -extensions etcd_server \
  -keyfile private/ca.key \
  -cert certs/ca.crt \
  -out certs/etcd0.example.com.crt -infiles etcd0.example.com.csr
```

### Verify the etcd Server Certificate

```
openssl x509 -in certs/etcd0.example.com.crt -noout -text
```

## Create a Client Certificate

```
unset SAN
```

```
openssl req -config openssl.cnf -new -nodes \
  -keyout private/etcd-client.key -out etcd-client.csr
```

```
openssl ca -config openssl.cnf -extensions etcd_client \
  -keyfile private/ca.key \
  -cert certs/ca.crt \
  -out certs/etcd-client.crt -infiles etcd-client.csr
```

### Testing HTTPS with cURL

Configure etcd

```
$ etcd -name etcd0 \
  --advertise-client-urls https://etcd0.example.com:2379 \
  --listen-client-urls https://10.0.1.10:2379 \
  --cert-file etcd0.example.com.crt \
  --key-file etcd0.example.com.key
```

```
$ curl --cacert ca.crt -XPUT -v -L https://etcd0.example.com:2379/v2/keys/foo -d value=bar
```

### Testing Client Auth with cURL

```
$ etcd -name etcd0 \
  --advertise-client-urls https://etcd0.example.com:2379 \
  --listen-client-urls https://10.0.1.10:2379 \
  --cert-file etcd0.example.com.crt \
  --key-file etcd0.example.com.key
  --ca-file ca.crt
```

```
curl --cacert ca.crt --key client.key --cert client.crt -v https://etcd0.example.com:2379/v2/keys
```
