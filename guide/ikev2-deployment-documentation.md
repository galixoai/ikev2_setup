

## Prerequisites
- Server with Ubuntu 22.04 configured.
- Update & upgrade the server.
- Make sure that the respective domain is pointed towards the new server from Cloudflare. (e.g., test.galixo.app ----> 10.21.23.43 (**Your New server IP**))

### Prior Step:
**Point the desired domain to the newly purchased server.**

![Config Image](assets/ikev2_dns_entry.png)

## 1. Install vnstat for Network Monitoring

Install and enable vnstat for network traffic statistics:

```bash
sudo apt install vnstat
sudo systemctl enable --now vnstat
```

**Step 01: Save the Scripts** 

Copy the files `ikev2-deployment-script.sh` and `ikev2_monitor.py` to the desired server in the `/root/scripts` folder.

Make them executable using:

```bash
chmod +x /root/scripts/ikev2-deployment-script.sh
chmod +x /root/scripts/ikev2_monitor.py
```

**Step 02: Navigate to the directory**

Navigate to the directory where you have placed the script `ikev2-deployment-script.sh`.

**Step 03: Run the script**

Run the `ikev2-deployment-script.sh` script in the directory using the following command:

```bash
/root/scripts/ikev2-deployment-script.sh
```

You need to wait for it to complete. After the successful execution of the script, choose the following options from the server during the execution of the script:

- For **Hostname for VPN:** Add the domain name that you just created in Cloudflare (e.g., test.galixo.app), then press Enter.
- Now provide the desired **VPN username:** (e.g., admin).
- For **VPN password:** Generate it from a random password generator online (e.g., jakshffdsfadsf).
- For **Timezone:** Copy the server IP and check its timezone using ipinfo.io, and add the result in the config. **NOTE: DO NOT ADD A RANDOM OR WRONG TIMEZONE.**
- For **Email address for sysadmin:** Make sure that you provide your correct email address. In case of a wrong email, it won't work and you will have to reconfigure the server.

Below is the SS of configurations.

![Config Image](assets/ikev2-config.png)


**Step 04: Add the Schedules to the crontab**

Open the cronjob file using the following command and then paste the cronjob lines defined below:
```bash
crontab -e
```
Add the following cron jobs in the newly freshed server:

```bash
0 3 * * * certbot renew --quiet
*/3 * * * * /root/scripts/ikev2_monitor.py
```

**Step 05: Enable the firewall**

Run these commands to allow ports and enable the firewall:

```bash
sudo ufw allow from 38.180.244.244 comment 'VPN server IP'
sudo ufw allow from 139.59.23.10 comment 'Backend server IP'
sudo ufw allow 4500
sudo ufw allow 500

sudo ufw enable
sudo ufw reload
```

**Step 06: Add the credentials to the API**

Add the file to the API [papi.fusionsai.net] under Desktop servers.

If it connects successfully you can test it using `whatsmyip`.
