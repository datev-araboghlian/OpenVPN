# OpenVPN

# Kompta Plus Bien - OpenVPN Project Documentation

## Project Overview
Client: Kompta Plus Bien (KPB), a company with 30 employees.  
Goal: Securely allow remote access to internal resources (accounting documents, invoices, shared files) for teleworkers using a VPN.  
Solution: Implement OpenVPN on Debian, tested on virtual machines.

## Environment Setup

### Tools Used
- Debian 12 (Bookworm)
- OpenVPN
- easy-rsa v3
- VirtualBox (for testing)

### Virtual Machines
- VM1: OpenVPN Server (Debian)
- VM2: OpenVPN Client (Debian)

## Step-by-Step Implementation

### 1. Server-Side Configuration (VM1)

#### 1.1 Install OpenVPN and Easy-RSA
```bash
sudo apt update
sudo apt install openvpn easy-rsa -y
```

#### 1.2 Set Up Certificate Infrastructure
```bash
mkdir ~/openvpn-ca
cd ~/openvpn-ca
cp -r /usr/share/easy-rsa/* .
./easyrsa init-pki
./easyrsa build-ca
```

#### 1.3 Generate Server Certificates and Keys
```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key
```

#### 1.4 Copy Files to OpenVPN Directory
```bash
sudo mkdir /etc/openvpn/keys
sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem ta.key /etc/openvpn/keys/
```

#### 1.5 Configure OpenVPN Server
```bash
sudo gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
sudo nano /etc/openvpn/server.conf
```

Key settings in `server.conf`:
```
ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/server.crt
key /etc/openvpn/keys/server.key
dh /etc/openvpn/keys/dh.pem
tls-auth /etc/openvpn/keys/ta.key 0
cipher AES-256-CBC
auth SHA256
user nobody
group nogroup
persist-key
persist-tun
keepalive 10 120
```

#### 1.6 Enable IP Forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### 1.7 Start and Enable the Server
```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

### 2. Client-Side Configuration (VM2)

#### 2.1 Install OpenVPN
```bash
sudo apt install openvpn -y
```

#### 2.2 Generate Client Keys on Server
On VM1:
```bash
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

Copy these files to VM2:
- pki/ca.crt
- pki/issued/client1.crt
- pki/private/client1.key
- ta.key

#### 2.3 Create Client Configuration File
File: `client.ovpn`
```
client
dev tun
proto udp
remote [Server-IP] 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
tls-auth ta.key 1
cipher AES-256-CBC
auth SHA256
verb 3
```

#### 2.4 Connect to the VPN
```bash
sudo openvpn --config client.ovpn
```

### 3. Testing and Validation
- Ping server from client
- Access internal resources (e.g., shared directories)
- Verify tunnel interface (`ip a`, `ifconfig`)

### 4. Optional: Two-Factor Authentication (2FA)
Install on server:
```bash
sudo apt install libpam-google-authenticator
```
Run setup per user:
```bash
google-authenticator
```
Configure PAM and OpenVPN to require code input.

### 5. Failure Recovery
- Enable auto-restart:
```bash
sudo systemctl enable openvpn@server
```
- Add cron job to monitor service:
```bash
*/5 * * * * systemctl is-active --quiet openvpn@server || systemctl restart openvpn@server
```

## Conclusion
This setup allows secure remote access for Kompta Plus Bien employees using OpenVPN with certificate-based authentication and optional two-factor security. The solution is scalable, secure, and tested in a virtual environment prior to deployment.
