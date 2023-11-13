#!/bin/bash

# Bash script to install and configure Splunk Enterprise using the system's package manager

# Check if the script is run as root
if [ "$EUID" -ne 0 ]
  then echo "Please run as root"
  exit
fi

is_username_safe() {
    [[ "$1" =~ ^[a-zA-Z0-9]+$ ]] && return 0 || return 1
}

is_password_safe() {
    [[ ! "$1" =~ [;&$|`] ]] && return 0
    return 1
}

is_path_safe() {
    [[ ! "$1" =~ [;&$|`] ]] && return 0 || return 1
}

# Prompt user for inputs with error handling and security checks
while true; do
    read -p "Enter Admin Name: " ADMIN_NAME
    is_username_safe "$ADMIN_NAME" && [ -n "$ADMIN_NAME" ] && break || echo "Invalid or unsafe input."
done

while true; do
    read -sp "Enter Admin Password: " ADMIN_PASSWORD
    echo # Newline for visual separation after password input
    is_password_safe "$ADMIN_PASSWORD" && [ -n "$ADMIN_PASSWORD" ] && break || echo "Invalid or unsafe input."
done

while true; do
    read -p "Use systemd for Splunkd (true|false): " USE_SYSTEMD
    is_input_safe "$USE_SYSTEMD" && [[ "$USE_SYSTEMD" == "true" || "$USE_SYSTEMD" == "false" ]] && break || echo "Invalid or unsafe input."
done

while true; do
    read -p "Splunk Process User (e.g. splunk): " SPLUNK_USER
    is_username_safe "$SPLUNK_USER" && [ -n "$SPLUNK_USER" ] && break || echo "Invalid or unsafe input."
done

while true; do
    read -p "Files/Folders paths for /opt/splunk/etc/apps (comma-separated, can be empty): " FILES_FOLDERS
    is_path_safe "$FILES_FOLDERS" && break || echo "Invalid or unsafe input."
done

while true; do
    read -p "Your serverclass.conf file path (can be empty): " SERVERCLASSES
    is_path_safe "$SERVERCLASSES" && break || echo "Invalid or unsafe input."
done

while true; do
    read -p "Enable Web-GUI (true|false): " WEB_GUI
    is_input_safe "$WEB_GUI" && [[ "$WEB_GUI" == "true" || "$WEB_GUI" == "false" ]] && break || echo "Invalid or unsafe input."
done


# New section for license file
while true; do
    read -p "Path to Splunk license file (leave empty if not applicable): " LICENSE_PATH
    [ -z "$LICENSE_PATH" ] && break # Allow empty input for no license
    is_path_safe "$LICENSE_PATH" && [ -f "$LICENSE_PATH" ] && break || echo "Invalid path or unsafe input."
done


# Determine package manager and install Splunk
echo "> Splunk installation and configuration completed..."
if command -v dpkg &> /dev/null; then
    wget -O splunk.deb 'https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=linux&version=latest&product=splunk&filename=splunk-latest-x86_64.deb&wget=true'
    dpkg -i splunk.deb
elif command -v rpm &> /dev/null; then
    wget -O splunk.rpm 'https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=linux&version=latest&product=splunk&filename=splunk-latest-x86_64.rpm&wget=true'
    rpm -i splunk.rpm
else
    echo "Unsupported package manager."
    exit 1
fi

# Change ownership if SPLUNK_USER is not 'root'
echo "> Ensuring Splunk root directory ownership is set to splunk:splunk..."
if [ "$SPLUNK_USER" != "root" ]; then
    useradd -r -s /sbin/nologin $SPLUNK_USER
    chown -R $SPLUNK_USER:$SPLUNK_USER /opt/splunk
fi

# Copy files and folders to /opt/splunk/etc/apps/
IFS=',' read -ra ADDR <<< "$FILES_FOLDERS"
for file_or_folder in "${ADDR[@]}"; do
    cp -R "$file_or_folder" /opt/splunk/etc/apps/
done


# Copy serverclass.conf file to /opt/splunk/etc/system/local/
IFS=',' read -ra ADDR <<< "$SERVERCLASSES"
for file_or_folder in "${ADDR[@]}"; do
    cp -R "$file_or_folder" /opt/splunk/etc/apps/
done


# Start Splunk for the first time
echo "> Starting Splunk..."
if [ "$USE_SYSTEMD" == "true" ]; then
    /opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd $ADMIN_PASSWORD
    # Enable boot-start/init script (using systemd)
    echo "> Configuring Splunk process to be systemd managed..."
    /opt/splunk/bin/splunk enable boot-start -user $SPLUNK_USER
else
    su $SPLUNK_USER -c "/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd $ADMIN_PASSWORD"
fi

# Change the admin username
echo "> Ensuring Splunk admin username..."
/opt/splunk/bin/splunk edit user admin -edit_username $ADMIN_NAME -auth admin:$ADMIN_PASSWORD

# Open the port for the web GUI if required
if [ "$WEB_GUI" == "true" ]; then
    echo "> Configuring Splunkweb port tcp/8000..."
    if command -v ufw &> /dev/null; then
        ufw allow 8000/tcp
        ufw reload
    elif command -v firewall-cmd &> /dev/null; then
        firewall-cmd --zone=public --add-port=8000/tcp --permanent
        firewall-cmd --reload
    else
        echo "Firewall tool not detected. Please open port tcp/8000 manually."
    fi
fi

# Open Splunk management port
echo "> Configuring Splunk management port tcp/8089..."
if command -v ufw &> /dev/null; then
    ufw allow 8089/tcp
    ufw reload
elif command -v firewall-cmd &> /dev/null; then
    firewall-cmd --zone=public --add-port=8089/tcp --permanent
    firewall-cmd --reload
else
    echo "Firewall tool not detected. Please open port tcp/8089 manually."
fi


# Test firewall ports
echo "Testing opened firewall ports..."

# Get a list of listening ports using netstat
LISTENING_PORTS=$(netstat -tuln | grep LISTEN | awk '{print $4}' | grep -oE '[0-9]+$')

# Test each port with nc
for PORT in $LISTENING_PORTS; do
    if nc -z -v -w5 localhost $PORT &>/dev/null; then
        echo "Port $PORT is open and accessible."
    else
        echo "WARNING: Port $PORT seems closed or inaccessible!"
    fi
done



# Add license
echo "> Install license..."
if [ -n "$LICENSE_PATH" ]; then
    cp "$LICENSE_PATH" /opt/splunk/etc/licenses/enterprise/ # You may need to adjust this path based on Splunk version or setup
fi




echo "> Splunk Deployment server installation and configuration completed."

