rm -f setup.sh
cat > setup.sh << 'EOF'
#!/bin/bash

# XFCE + XRDP Setup Script for Oracle Ampere Debian VPS
# Based on: https://sobaigu.com/linux-with-windows-desktop.html

set -e

echo "=================================================="
echo "  XFCE Desktop + XRDP Installation Script"
echo "  For Oracle Ampere Debian VPS"
echo "=================================================="
echo ""

# Color codes for output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m' # No Color

print_status() {
    echo -e "${GREEN}[✓]${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}[!]${NC} $1"
}

print_error() {
    echo -e "${RED}[✗]${NC} $1"
}

print_info() {
    echo -e "${BLUE}[i]${NC} $1"
}

# Error handler
error_exit() {
    print_error "$1"
    exit 1
}

# Check if running as root
if [ "$EUID" -ne 0 ]; then 
    error_exit "Please run as root (use sudo)"
fi

# Generate random password (10 characters: uppercase, lowercase, numbers, symbols)
generate_password() {
    # Use /dev/urandom for better randomness
    PASSWORD=$(cat /dev/urandom | tr -dc 'A-Za-z0-9!@#$%^&*()_+-=' | fold -w 10 | head -n 1)
    
    # Ensure it has at least one of each type
    while ! echo "$PASSWORD" | grep -q '[A-Z]' || \
          ! echo "$PASSWORD" | grep -q '[a-z]' || \
          ! echo "$PASSWORD" | grep -q '[0-9]' || \
          ! echo "$PASSWORD" | grep -q '[!@#$%^&*()_+=-]'; do
        PASSWORD=$(cat /dev/urandom | tr -dc 'A-Za-z0-9!@#$%^&*()_+-=' | fold -w 10 | head -n 1)
    done
    
    echo "$PASSWORD"
}

# Get server IP address (public IP)
print_status "Detecting server IP address..."
SERVER_IP=$(curl -4 -s ifconfig.me 2>/dev/null)
if [ -z "$SERVER_IP" ]; then
    SERVER_IP=$(curl -4 -s icanhazip.com 2>/dev/null)
fi
if [ -z "$SERVER_IP" ]; then
    SERVER_IP=$(hostname -I | awk '{print $1}')
fi
if [ -z "$SERVER_IP" ]; then
    SERVER_IP="Unable to detect - check manually"
fi

# Create RDP username and password
RDP_USERNAME="rdpuser"
RDP_PASSWORD=$(generate_password)

echo ""
print_info "Generated RDP Credentials:"
echo "  Username: $RDP_USERNAME"
echo "  Password: $RDP_PASSWORD"
echo "  Server IP: $SERVER_IP"
echo ""
read -p "Press Enter to continue with installation..."
echo ""

# Update system
print_status "Updating system packages..."
apt update || error_exit "Failed to update package list"
apt upgrade -y || error_exit "Failed to upgrade packages"

# Install XFCE Desktop Environment
print_status "Installing XFCE Desktop Environment..."
apt install -y xfce4 xfce4-goodies dbus-x11 || error_exit "Failed to install XFCE"

# Verify XFCE installation
if ! dpkg -l | grep -q xfce4; then
    error_exit "XFCE installation verification failed"
fi
print_info "XFCE installed successfully"

# Install XRDP
print_status "Installing XRDP..."
apt install -y xrdp || error_exit "Failed to install XRDP"

# Verify XRDP installation
if ! dpkg -l | grep -q xrdp; then
    error_exit "XRDP installation verification failed"
fi
print_info "XRDP installed successfully"

# Create RDP user
print_status "Creating RDP user: $RDP_USERNAME..."
if id "$RDP_USERNAME" &>/dev/null; then
    print_warning "User $RDP_USERNAME already exists, resetting password..."
    echo "$RDP_USERNAME:$RDP_PASSWORD" | chpasswd
else
    useradd -m -s /bin/bash "$RDP_USERNAME" || error_exit "Failed to create user"
    echo "$RDP_USERNAME:$RDP_PASSWORD" | chpasswd || error_exit "Failed to set password"
    usermod -aG sudo "$RDP_USERNAME" || print_warning "Failed to add user to sudo group"
fi

# Verify user creation
if ! id "$RDP_USERNAME" &>/dev/null; then
    error_exit "User creation verification failed"
fi
print_info "RDP user created successfully"

# Configure XRDP to use XFCE
print_status "Configuring XRDP to use XFCE..."
# Configure for the new user
mkdir -p /home/$RDP_USERNAME
echo "startxfce4" > /home/$RDP_USERNAME/.xsession
chown $RDP_USERNAME:$RDP_USERNAME /home/$RDP_USERNAME/.xsession
chmod +x /home/$RDP_USERNAME/.xsession

# Also update the global config
if ! grep -q "startxfce4" /etc/xrdp/startwm.sh; then
    sed -i '/^test -x \/etc\/X11\/Xsession/i startxfce4' /etc/xrdp/startwm.sh
fi

# Start and enable XRDP service
print_status "Starting XRDP service..."
systemctl start xrdp || error_exit "Failed to start XRDP service"
systemctl enable xrdp || error_exit "Failed to enable XRDP service"

# Verify XRDP is running
sleep 2
if ! systemctl is-active --quiet xrdp; then
    error_exit "XRDP service is not running"
fi
print_info "XRDP service is running"

# Configure firewall (if ufw is installed)
if command -v ufw &> /dev/null; then
    print_status "Configuring firewall..."
    ufw allow 3389/tcp || print_warning "Failed to configure firewall rule"
else
    print_warning "UFW not installed, skipping firewall configuration"
fi

# Install common software
print_status "Installing Firefox browser..."
apt install -y firefox-esr || print_warning "Failed to install Firefox"

# Verify Firefox installation
if dpkg -l | grep -q firefox-esr; then
    print_info "Firefox installed successfully"
fi

print_status "Installing Telegram Desktop..."
apt install -y telegram-desktop || print_warning "Failed to install Telegram"

# Verify Telegram installation
if dpkg -l | grep -q telegram-desktop; then
    print_info "Telegram installed successfully"
fi

# Install VS Code via snap (easier method)
print_status "Installing Snapd..."
apt install -y snapd || print_warning "Failed to install snapd"

if command -v snap &> /dev/null; then
    systemctl start snapd
    systemctl enable snapd
    sleep 3
    
    print_status "Installing VS Code..."
    snap install code --classic || print_warning "Failed to install VS Code"
    
    # Verify VS Code installation
    if snap list 2>/dev/null | grep -q code; then
        print_info "VS Code installed successfully"
    fi
fi

# Create VS Code desktop launcher for RDP user
print_status "Creating VS Code desktop launcher..."
mkdir -p /home/$RDP_USERNAME/Desktop
cat > /home/$RDP_USERNAME/Desktop/vscode.desktop << 'DESKTOP_EOF'
[Desktop Entry]
Name=Visual Studio Code
Comment=Code Editing. Redefined.
GenericName=Text Editor
Exec=code --user-data-dir="/home/rdpuser/.vscode"
Icon=com.visualstudio.code
Type=Application
StartupNotify=false
StartupWMClass=Code
Categories=TextEditor;Development;IDE;
MimeType=text/plain;inode/directory;
Actions=new-empty-window;
Keywords=vscode;

[Desktop Entry new-empty-window]
Name=New Empty Window
Exec=code --user-data-dir="/home/rdpuser/.vscode" --new-window
Icon=com.visualstudio.code
DESKTOP_EOF

chmod +x /home/$RDP_USERNAME/Desktop/vscode.desktop
chown -R $RDP_USERNAME:$RDP_USERNAME /home/$RDP_USERNAME/Desktop

# Install clipboard tools
print_status "Installing clipboard support tools..."
apt install -y xclip xsel || print_warning "Failed to install clipboard tools"

# Verify clipboard tools
if command -v xclip &> /dev/null && command -v xsel &> /dev/null; then
    print_info "Clipboard tools installed successfully"
fi

# Disable XFCE compositor for better performance
print_status "Optimizing XFCE performance..."
mkdir -p /home/$RDP_USERNAME/.config/xfce4/xfconf/xfce-perchannel-xml
cat > /home/$RDP_USERNAME/.config/xfce4/xfconf/xfce-perchannel-xml/xfwm4.xml << 'XFCE_EOF'
<?xml version="1.0" encoding="UTF-8"?>
<channel name="xfwm4" version="1.0">
  <property name="general" type="empty">
    <property name="use_compositing" type="bool" value="false"/>
  </property>
</channel>
XFCE_EOF
chown -R $RDP_USERNAME:$RDP_USERNAME /home/$RDP_USERNAME/.config

# Install additional useful tools
print_status "Installing additional useful tools..."
apt install -y \
    htop \
    vim \
    wget \
    curl \
    git \
    net-tools \
    unzip \
    p7zip-full \
    file-roller \
    mousepad || print_warning "Some additional tools failed to install"

# Clean up
print_status "Cleaning up..."
apt autoremove -y
apt clean

# Final verification
echo ""
print_status "Running final verification checks..."
sleep 1

VERIFICATION_FAILED=0

# Check XRDP service
if systemctl is-active --quiet xrdp; then
    print_info "XRDP service: Running ✓"
else
    print_error "XRDP service: Not running ✗"
    VERIFICATION_FAILED=1
fi

# Check XRDP port
if netstat -tuln 2>/dev/null | grep -q ":3389" || ss -tuln 2>/dev/null | grep -q ":3389"; then
    print_info "XRDP port 3389: Listening ✓"
else
    print_error "XRDP port 3389: Not listening ✗"
    VERIFICATION_FAILED=1
fi

# Check XFCE installation
if command -v startxfce4 &> /dev/null; then
    print_info "XFCE desktop: Installed ✓"
else
    print_error "XFCE desktop: Not found ✗"
    VERIFICATION_FAILED=1
fi

# Check user exists
if id "$RDP_USERNAME" &>/dev/null; then
    print_info "RDP user: Created ✓"
else
    print_error "RDP user: Not found ✗"
    VERIFICATION_FAILED=1
fi

if [ $VERIFICATION_FAILED -eq 1 ]; then
    echo ""
    print_error "Some components failed verification. Please check the errors above."
    exit 1
fi

# Save credentials to file
CRED_FILE="/root/rdp_credentials.txt"
cat > $CRED_FILE << CRED_EOF
================================================
RDP CONNECTION CREDENTIALS
================================================

Server IP:  $SERVER_IP
Port:       3389
Username:   $RDP_USERNAME
Password:   $RDP_PASSWORD

================================================
Generated: $(date)
================================================
CRED_EOF

chmod 600 $CRED_FILE

# Display success message with connection details
clear
echo ""
echo "=================================================="
echo -e "${GREEN}  ✓ INSTALLATION COMPLETED SUCCESSFULLY! ✓${NC}"
echo "=================================================="
echo ""
echo "╔═══════════════════════════════════════════════╗"
echo "║       RDP CONNECTION DETAILS                  ║"
echo "╚═══════════════════════════════════════════════╝"
echo ""
echo "  Server IP:  $SERVER_IP"
echo "  RDP Port:   3389"
echo "  Username:   $RDP_USERNAME"
echo "  Password:   $RDP_PASSWORD"
echo ""
echo "=================================================="
echo ""
echo "HOW TO CONNECT FROM WINDOWS:"
echo ""
echo "  1. Press Win+R and type: mstsc"
echo ""
echo "  2. Enter: $SERVER_IP:3389"
echo ""
echo "  3. Click 'Connect'"
echo ""
echo "  4. Login with:"
echo "     Username: $RDP_USERNAME"
echo "     Password: $RDP_PASSWORD"
echo ""
echo "  5. Select Session: Xorg"
echo ""
echo "=================================================="
echo ""
echo "INSTALLED SOFTWARE:"
echo "  ✓ XFCE Desktop Environment"
echo "  ✓ Firefox ESR Browser"
echo "  ✓ Telegram Desktop"
echo "  ✓ VS Code (desktop launcher available)"
echo "  ✓ Clipboard tools (xclip, xsel)"
echo "  ✓ Text Editor (Mousepad)"
echo "  ✓ File utilities (htop, vim, git, etc.)"
echo ""
echo "=================================================="
echo ""
echo -e "${YELLOW}IMPORTANT NOTES:${NC}"
echo ""
echo "  • Credentials saved to: $CRED_FILE"
echo "  • User '$RDP_USERNAME' has sudo privileges"
echo "  • Desktop optimized for performance"
echo "  • English fonts only (no Chinese fonts)"
echo ""
echo -e "${GREEN}RECOMMENDED:${NC} Reboot VPS for best results"
echo "  Command: reboot"
echo ""
echo "=================================================="
echo ""

# Also display credentials in plain text for easy copying
echo "COPY THESE CREDENTIALS:"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "IP: $SERVER_IP"
echo "Username: $RDP_USERNAME"
echo "Password: $RDP_PASSWORD"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

EOF

bash setup.sh
