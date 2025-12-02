Installing a Windows environment on a Linux system and using remote desktop connection easy linux vps to windows RDP.

Installation:
1. sudo su
2. sudo apt update && sudo apt install ufw -y && sudo ufw allow from any && sudo ufw reload && sudo ufw status verbose && echo "y" | sudo ufw enable
3. wget https://raw.githubusercontent.com/tikubgh/linux-vps-windows-rdp/refs/heads/main/vps-script.sh
4. chmod +x vps-script.sh && ./vps-script.sh

Installation Complete: login to RDP using vps ip address with user & pass,
Enjoy..
