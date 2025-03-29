## **Tutorial: Setting Up Reverse SSH with AutoSSH (Using Public Key Authentication)**  
This guide will help you set up **AutoSSH** for a persistent **reverse SSH tunnel** where:  
✅ The **remote device** (behind NAT) connects to a **VPS** using **public key authentication**.  
✅ The **VPS** allows SSH access back to the **remote device** through a forwarded port.  

---

### **Step 1: Set Up SSH Key Authentication**
Ensure both the **remote device** and the **VPS** use **public key authentication**.

#### **1.1 Generate SSH Keys (If Not Already Done)**
On the **remote device**, run:
```bash
ssh-keygen -t ed25519 -C "autossh-key" -f ~/.ssh/id_ed25519
```
Press **Enter** for default values (no passphrase required for AutoSSH).

#### **1.2 Copy the Key to the VPS**
Run the following command from the **remote device**:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub telebot@<VPS_PUBLIC_IP>
```
✅ Now, the remote device can SSH into the VPS **without a password**.

#### **1.3 Copy the Key to the Remote Device**
Similarly, log into your **VPS** and copy its SSH key to the **remote device**:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub cw03@<REMOTE_DEVICE_IP>
```
✅ Now, the VPS can SSH into the remote device **without a password**.

---

### **Step 2: Configure the VPS for Reverse SSH**
The **VPS** will act as the "middleman" for SSH access to the **remote device**.

#### **2.1 Enable Gateway Ports in SSH**
Edit the SSH config file on the **VPS**:
```bash
sudo nano /etc/ssh/sshd_config
```
Find and modify (or add) these lines:
```ini
GatewayPorts yes
AllowTcpForwarding yes
```
Then restart SSH:
```bash
sudo systemctl restart ssh
```

#### **2.2 Test SSH Connection**
Try **manually** opening a reverse SSH tunnel from the **remote device**:
```bash
ssh -N -R 2222:localhost:22 -p 6454 telebot@<VPS_PUBLIC_IP>
```
✅ Now, from the **VPS**, try connecting to the remote device:
```bash
ssh -p 2222 cw03@localhost
```
If this works, proceed to **AutoSSH setup**.

---

### **Step 3: Set Up AutoSSH on the Remote Device**
Now, we automate the connection using **AutoSSH**.

#### **3.1 Install AutoSSH**
On the **remote device**, install AutoSSH:
```bash
sudo apt update && sudo apt install -y autossh
```

#### **3.2 Create a Systemd Service**
Create a systemd service file:
```bash
sudo nano /etc/systemd/system/autossh-tunnel.service
```
Paste the following configuration:
```ini
[Unit]
Description=AutoSSH tunnel service to <VPS_PUBLIC_IP>
After=network.target

[Service]
User=cw03  # Make sure this user exists
ExecStart=/usr/bin/autossh -M 0 -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -o "ExitOnForwardFailure yes" \
  -N -R 2222:localhost:22 -p 6454 telebot@<VPS_PUBLIC_IP>
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### **3.3 Enable & Start AutoSSH**
```bash
sudo systemctl daemon-reload
sudo systemctl enable autossh-tunnel.service
sudo systemctl start autossh-tunnel.service
sudo systemctl status autossh-tunnel.service
```
✅ If the status shows **"active (running)"**, the reverse SSH tunnel is established.

---

### **Step 4: Verify & Use the Reverse SSH Tunnel**
From the **VPS**, SSH into the remote device using:
```bash
ssh -p 2222 cw03@localhost
```
🎉 **Success! You can now access the remote device from your VPS.**

---

### **Step 5: Troubleshooting**
#### 🔹 **Check AutoSSH Logs**
```bash
journalctl -u autossh-tunnel.service --no-pager --lines=50
```
#### 🔹 **Check if the Port is Open on the VPS**
```bash
netstat -tulnp | grep 2222
```
You should see SSH (`sshd`) **listening** on port **2222**.

#### 🔹 **Check Firewall Rules**
Make sure **UFW** or **iptables** allows SSH traffic:
```bash
sudo ufw allow 2222/tcp
sudo ufw reload
```

---

### **Final Notes**
- This setup ensures a **persistent reverse SSH tunnel** using **public key authentication**.
- **AutoSSH** automatically **reconnects** if the connection drops.
- You can now manage the **remote device** even if it's behind **NAT** or a **firewall**.

Let me know if you need further clarification! 🚀
