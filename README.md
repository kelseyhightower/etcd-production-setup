# etcd ssl

The following tutorial will walk you through the following:

- Creating a CA using the openssl command line tools.
- Configuring etcd clients (etcdctl and curl) for SSL and SSL client auth.

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

### Create the CA Certificate and Key

```
openssl req -config openssl.cnf -new -x509 -extensions v3_ca \
  -keyout private/ca.key -out certs/ca.crt
```

Type `ca.etcd.example.com` at the `Common Name (FQDN) []:` prompt.

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

Type `etcd0.example.com` at the `Common Name (FQDN) []:` prompt.

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

Type `etcd-client` at the `Common Name (FQDN) []:` prompt.

### Testing SSL

Configure etcd

```
$ etcd --advertise-client-urls https://etcd0.example.com:2379 \
  --listen-client-urls https://10.0.1.10:2379 \
  --cert-file etcd0.example.com.crt \
  --key-file etcd0.example.com.key
```

#### Using cURL

```
$ curl --cacert ca.crt -XPUT -v https://etcd0.example.com:2379/v2/keys/foo -d value=bar
```

```
$ curl --cacert ca.crt -v https://etcd0.example.com:2379/v2/keys
```

#### Using etcdctl

```
$ etcdctl -C https://etcd0.example.com:2379 --ca-file ca.crt set foo bar 
```

```
etcdctl -C https://etcd0.example.com:2379 --ca-file ca.crt get foo
```

### Testing Client Auth

```
$ etcd --advertise-client-urls https://etcd0.example.com:2379 \
  --listen-client-urls https://10.0.1.10:2379 \
  --cert-file etcd0.example.com.crt \
  --key-file etcd0.example.com.key \
  --ca-file ca.crt
```

Notice the usage of the `--ca-file` flag. This is what enables client auth.

#### With etcdctl

```
$ etcdctl -C https://etcd0.example.com:2379 \
  --cert-file etcd-client.crt \
  --key-file etcd-client.key \
  --ca-file ca.crt \
  get foo
```
