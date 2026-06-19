## Prerequisites
- Server with Ubuntu 24.04 configured.
- Update & upgrade the server.
- Ensure your DNS A-record is pointing to your new server's IP. Replace `<YOUR_DOMAIN>` with your actual domain name (e.g., ikrnd.galixo.app).

## Installation & Configuration

**Step 01: Install Core Dependencies**

Update the system and install strongSwan, the EAP authentication plugins, Certbot, and the persistent firewall manager.

```bash
sudo apt update
sudo apt install strongswan strongswan-pki libcharon-extra-plugins libstrongswan-extra-plugins certbot iptables-persistent -y
```

(If prompted during installation to save current IPv4 rules, select Yes).

**Step 02: Generate the RSA Certificate**

Force Certbot to generate an RSA key. This bypasses the Let's Encrypt "Generation Y" Android compatibility bug by ensuring the server uses the universally trusted ISRG Root X1 chain.

```bash
sudo certbot certonly --standalone -d <YOUR_DOMAIN> --key-type rsa
```

**Step 03: The AppArmor Bypass**

Ubuntu silently blocks strongSwan from reading files inside the Let's Encrypt archive. We must explicitly authorize it to read its own private key.

```bash
# 1. Inject the permission rule for your specific domain
echo "/etc/letsencrypt/archive/<YOUR_DOMAIN>/* r," | sudo tee -a /etc/apparmor.d/local/usr.lib.ipsec.charon

# 2. Reload AppArmor to apply the bypass immediately
sudo systemctl reload apparmor
```

**Step 04: Link and Split the Certificate Chain**

strongSwan has a hardcoded limitation where it only reads the first certificate in a .pem file. We must symlink the keys and dynamically split the Let's Encrypt chain so the server broadcasts the complete trust path.

```bash
# 1. Symlink the private key using a generic name
sudo ln -sf /etc/letsencrypt/live/<YOUR_DOMAIN>/privkey.pem /etc/ipsec.d/private/server.key.pem

# 2. Symlink the leaf certificate using a generic name
sudo ln -sf /etc/letsencrypt/live/<YOUR_DOMAIN>/cert.pem /etc/ipsec.d/certs/server.cert.pem

# 3. Split the Let's Encrypt multi-certificate chain into individual files
sudo awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/{ if(/BEGIN CERTIFICATE/){a++}; out="/etc/ipsec.d/cacerts/issuer-"a".pem"; print >out}' /etc/letsencrypt/live/<YOUR_DOMAIN>/chain.pem
```

**Step 05: Configure strongSwan (ipsec.conf)**

Open the main configuration file:

```bash
sudo nano /etc/ipsec.conf
```

Replace the contents entirely with this optimized configuration. We use uniqueids=never to allow multiple devices to log into the admin account simultaneously without dropping each other's connections.

```
config setup
    charondebug="ike 1, knl 1, cfg 2, enc 1, net 1"
    uniqueids=never

conn roadwarrior
    keyexchange=ikev2
    ike=aes256gcm16-prfsha384-ecp384,aes256gcm16-prfsha256-ecp256,aes256-sha2_512-ecp384,aes128gcm16-sha256-ecp256
    esp=aes256gcm16-ecp384,aes256gcm16-ecp256,aes128gcm16-ecp256
    fragmentation=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    
    left=%any
    leftauth=pubkey
    leftid=<YOUR_DOMAIN>
    leftcert=server.cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    
    right=%any
    rightauth=eap-mschapv2
    rightid=%any
    rightsendcert=never
    rightsourceip=10.10.10.0/24
    rightdns=1.1.1.1,8.8.8.8
    eap_identity=%identity
    auto=add
```

**Step 06: Configure User Credentials (ipsec.secrets)**

Map your explicit domain name to the generic RSA private key and define your VPN users.

```bash
sudo nano /etc/ipsec.secrets
```

Make it look like this:

```
<YOUR_DOMAIN> : RSA "server.key.pem"

admin : EAP "YourSecurePasswordHere"
user2 : EAP "AnotherPasswordHere"
```

**Step 07: The Master Routing & Firewall Script**

This script does all the heavy lifting for network speed. It auto-detects your interface, enables IP forwarding, protects the IPsec tunnel from NAT corruption, implements TCP MSS clamping (the MTU fix), and blocks QUIC (the YouTube fix).

Copy and paste this entire block into your terminal at once:

```bash
# 2. Auto-detect your public network interface
PUB_IF=$(ip -o -4 route show to default | awk '{print $5}')

# 3. Force the Linux Kernel to allow IP Forwarding permanently
sudo sysctl -w net.ipv4.ip_forward=1
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf

# 4. Flush any old tables to start clean
sudo iptables -t nat -F POSTROUTING
sudo iptables -t mangle -F FORWARD

# 5. Explicitly ACCEPT forwarded VPN traffic
sudo iptables -P FORWARD ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 10.10.10.0/24 -j ACCEPT

# 6. Apply IPsec NAT Bypass to prevent header corruption
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o $PUB_IF -m policy --dir out --pol ipsec -j ACCEPT

# 7. Apply standard NAT Masquerade for internet traffic
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o $PUB_IF -j MASQUERADE

# 8. Apply TCP MSS Clamping to prevent oversized packet drops
sudo iptables -t mangle -A FORWARD -m policy --dir in --pol ipsec -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360

# 9. Block UDP 443 to disable QUIC, forcing streaming services to use safe TCP
sudo iptables -A FORWARD -p udp --dport 443 -j REJECT

# 10. Save all rules permanently
sudo netfilter-persistent save
```

**Step 08: Start the Engine**

Restart the VPN daemon to load the new identities, bypassed certificates, and user configurations.

```bash
sudo systemctl restart ipsec
sudo systemctl enable ipsec
```

**Step 09: Install vnstat for Network Monitoring**

Install and enable vnstat for network traffic statistics:

```bash
sudo apt install vnstat
sudo systemctl enable --now vnstat
```

**Step 10: Place the Monitoring Script**

Copy the file `ikev2_monitor.py` to the desired server in the `/root/scripts` folder.

Make it executable using:

```bash
chmod +x /root/scripts/ikev2_monitor.py
```

**Step 11: Add Cron Jobs for Maintenance**

Open the cronjob file using the following command and then paste the cronjob lines defined below:

```bash
crontab -e
```

Add the following cron jobs in the server:

```bash
0 3 * * * certbot renew --quiet
*/3 * * * * /root/scripts/ikev2_monitor.py
```

**Step 12: Enable the Firewall**

Run these commands to allow specific ports and enable the firewall:

```bash
sudo ufw allow from 38.180.244.244 comment 'VPN server IP'
sudo ufw allow from 139.59.61.11 comment 'Backend server IP'
sudo ufw allow 4500
sudo ufw allow 500

sudo ufw enable
sudo ufw reload
```

Your server is fully optimized, secured, and ready for deployment. Connect your client and enjoy the speeds!