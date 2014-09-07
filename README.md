# etcd ssl

## Create the CA

```
mkdir -p etcd-ca/{private,certs,newcerts,crl}
cp openssl.cnf etcd-ca/openssl.cnf
touch etcd-ca/index.txt
echo '01' > etcd-ca/serial
```

## Create the CA Certificate and Key

```
export CN="ca.etcd.36containers.com"
```

```
cd etcd-ca
```

```
openssl req -config openssl.cnf -new -x509 -extensions v3_ca \
  -keyout private/${CN}.key -out certs/${CN}.crt
```

### Verify the CA Certificate

```
openssl x509 -in certs/${CN}.crt -noout -text
```

## Create a etcd Server Certificate

```
export CN="0.etcd.36containers.com"
export SAN="IP:127.0.0.1, IP:10.132.243.172, IP:104.131.62.108"
```

```
openssl req -config openssl.cnf -new -nodes \
  -keyout private/${CN}.key -out ${CN}.csr
```

```
openssl ca -config openssl.cnf -extensions etcd_server \
  -out certs/${CN}.crt -infiles ${CN}.csr
```

### Verify the etcd Server Certificate

```
openssl x509 -in certs/${CN}.crt -noout -text
```

## Create a Client Certificate

```
export CN="client.36containers.com"
unset SAN
```

```
openssl req -config openssl.cnf -new -nodes \
  -keyout private/${CN}.key -out ${CN}.csr
```

```
openssl ca -config openssl.cnf -extensions etcd_client \
  -out certs/${CN}.crt -infiles ${CN}.csr
```

### Testing HTTPS with cURL

```
/usr/bin/etcd -f -name 0.etcd.36containers.com \
  -data-dir 0.etcd.36containers.com \
  -cert-file 0.etcd.36containers.com.crt \
  -key-file 0.etcd.36containers.com.key
```

```
curl -XPUT -v -L https://127.0.0.1:4001/v2/keys/foo -d value=bar
  --cacert ca.etcd.36containers.com.crt \
```

### Testing Client Auth with cURL

```
/usr/bin/etcd -f -name 0.etcd.36containers.com \
  -data-dir 0.etcd.36containers.com \
  -ca-file ca.etcd.36containers.com.crt \
  -cert-file 0.etcd.36containers.com.crt \
  -key-file 0.etcd.36containers.com.key
```

```
curl -XPUT -v -L https://127.0.0.1:4001/v2/keys/foo -d value=bar \
  --key client.36containers.com.key \
  --cert client.36containers.com.crt \
  --cacert ca.etcd.36containers.com.crt
```
