# How-to-manage-Core-CA

## STEP 1: Setup Internal Root CA di salah satu server yang dijadikan trust server
### Struktur direktori:
```
mkdir -p ~/ca/{certs,crl,newcerts,private}
sudo chmod 700 ~/ca/private
cd ~/ca
touch index.txt
sudo bash -c 'echo 1000 > serial'
```

### Buat Root Key dan Self-Signed Certificate (valid 10 tahun)
```
openssl genrsa -aes256 -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem

openssl req -x509 -new -nodes -key private/ca.key.pem \
  -sha256 -days 3650 -out certs/ca.cert.pem \
  -subj "/C=ID/ST=Jakarta/O=PT.Maju Mundur/OU=IT Security/CN=homelab.internal"
chmod 444 certs/ca.cert.pem

```

🔐 File penting:

private/ca.key.pem → Jangan sampai bocor!

certs/ca.cert.pem → Sertifikat root CA yang bisa disebar

## STEP 2: Generate Sertifikat untuk Server lainnya
sebagai contoh akan generate untuk server ldap

### 1. Buat Key dan CSR
```
openssl genrsa -out ldap.aspitsme.internal.key.pem 2048

openssl req -new -key ldap.homelab.internal.key.pem -out ldap.homelab.internal.csr.pem \
  -subj "/C=ID/ST=Jakarta/O=PT.Maju Mundur/OU=LDAP Server/CN=ldap.homelab.internal"

```

### 2. Buat file ekstensi untuk SAN (Subject Alternative Name)
buat file contoh ldap.ext:
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = ldap.homelab.internal
IP.1 = 10.66.40.8
```
### 3. Sign Certticate nya dengan internal CA
```
openssl x509 -req -in ldap.homelab.internal.csr.pem \
  -CA certs/ca.cert.pem -CAkey private/ca.key.pem -CAcreateserial \
  -out ldap.homelab.internal.cert.pem -days 825 -sha256 -extfile ldap.ext
```

📦 Hasil File:
- 🔒 ldap.aspitsme.internal.key.pem → Private Key (digunakan server LDAP)
- 🔐 ldap.aspitsme.internal.cert.pem → Sertifikat
- 📄 certs/ca.cert.pem → Root CA (disebar ke client untuk trusted)

## Step 3: (Opsional) Tambahkan ca.cert.pem ke Client agar Trusted
contoh di Linux:
```
sudo cp ca.cert.pem /usr/local/share/ca-certificates/homelab-ca.crt
sudo update-ca-certificates

```

## Step 4: Extend -- Tambah Sever Baru
Misalnya punya server lain:
### 1. Buat Key & CSR:
```
openssl genrsa -out zabbix.homelab.internal.key.pem 2048
openssl req -new -key zabbix.homelab.internal.key.pem -out zabbix.homelab.internal.csr.pem \
  -subj "/C=ID/ST=Jakarta/O=PT.Maju Mundur/OU=Monitoring/CN=zabbix.homelab.internal"
```

### 2. Ubah ldap.ext ke zabbix.ext dengan:
```
subjectAltName = @alt_names
[alt_names]
DNS.1 = zabbix.homelab.internal
IP.1 = 10.66.40.9
```

###. 3. Sign
```
openssl x509 -req -in zabbix.homelab.internal.csr.pem \
  -CA certs/ca.cert.pem -CAkey private/ca.key.pem \
  -CAcreateserial -out zabbix.homelab.internal.cert.pem \
  -days 825 -sha256 -extfile zabbix.ext
```
