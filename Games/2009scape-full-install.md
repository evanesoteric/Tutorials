# 2009scape Full Production Setup Guide (Server & Client)

This guide is split into two parts. Part A covers setting up a production-ready 2009scape server on a Debian-based Linux system (like Ubuntu). Part B covers setting up the game client on both Debian and Arch-based systems to connect to your server.

Part A: Server Setup (Debian / Ubuntu)
This section details installing the server application, MariaDB database, and running it as a persistent systemd service.

1. Server Prerequisites & Setup
First, install essential packages, create a dedicated user for the server, install Java 11, and clone the source code.

# Update package list and install dependencies
sudo apt update
sudo apt install -y git mariadb-server openjdk-11-jdk

# Create a dedicated user to run the game server for security
sudo adduser scape

# Switch to the new user
su - scape

# Now, clone the server repository into the new user's home directory
git clone [https://gitlab.com/2009scape/2009scape.git](https://gitlab.com/2009scape/2009scape.git)
cd 2009scape/

# Verify Java 11 installation (can be run as any user)
java -version

Note: For the rest of Part A, all commands should be run as the scape user, unless they are prefixed with sudo.

2. Database Configuration
Configure the MariaDB database and import the initial server data.

Log into MariaDB (from the root/sudo account):

sudo mysql

Create the database and user. Replace YourSuperSecurePassword with a strong, unique password.

CREATE DATABASE global;
CREATE USER 'rsadmin'@'localhost' IDENTIFIED BY 'YourSuperSecurePassword';
GRANT ALL PRIVILEGES ON global.* TO 'rsadmin'@'localhost';
FLUSH PRIVILEGES;
EXIT;

Prepare the SQL script (as the scape user):

nano ~/2009scape/Server/db_exports/global.sql

Comment out the first two lines by adding --  to the beginning of each:

-- CREATE DATABASE global;
-- USE global;

Save and exit the editor (Ctrl+O, Enter, Ctrl+X).

Import the data. This will prompt for the password you created.

mysql -u rsadmin -p global < ~/2009scape/Server/db_exports/global.sql

VERIFY THE IMPORT: Log back into the database and check the tables.

sudo mysql -u rsadmin -p -e "USE global; SHOW TABLES;"

The output should list many tables (e.g., players, player_banks, etc.). If not, the import failed, and you must repeat step 4.

3. Server Build & Configuration
Build the server application using the Maven wrapper.

Navigate to the server directory and make the wrapper executable:

cd ~/2009scape/Server/
chmod +x ./mvnw

Clean and build the project:

./mvnw clean package

Configure the server's database connection:

nano worldprops/default.conf

Update the [database] section with the credentials from Step 2.

[database]
database_name = "global"
database_username = "rsadmin"
database_password = "YourSuperSecurePassword"
database_address = "127.0.0.1"
database_port = "3306"

Set the Player Storage Provider: In the same default.conf file, add the storage_provider line under the [server] section.

[server]
storage_provider = "sql"

Enable Production Settings (CRITICAL): In the same file, find use_auth and persist_accounts and change both to true.

use_auth = true #NOTE: THIS MUST BE SET TO TRUE IN PRODUCTION!
persist_accounts = true #NOTE: THIS MUST BE SET TO TRUE IN PRODUCTION!

Save and exit.

4. Running as a Production Service
Create a systemd service to manage the server.

Create the service file (as root/sudo user):

sudo nano /etc/systemd/system/2009scape.service

Paste the following configuration. Note that User, Group, WorkingDirectory, and ExecStart have been updated for the scape user.

[Unit]
Description=2009scape Game Server
After=network.target mariadb.service

[Service]
User=scape
Group=scape
Type=simple
WorkingDirectory=/home/scape/2009scape/
ExecStart=/usr/bin/java -Xms4G -Xmx4G -XX:+UseG1GC -jar /home/scape/2009scape/Server/target/server-1.0.0-jar-with-dependencies.jar
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target

Enable and start the service:

sudo systemctl daemon-reload
sudo systemctl enable 2009scape.service
sudo systemctl restart 2009scape.service

Check the status or view live logs:

sudo systemctl status 2009scape.service
sudo journalctl -u 2009scape -f

5. Production Hardening & Maintenance
These final steps secure your server and make it easier to maintain.

Configure a Firewall (UFW):

# Allow SSH connections (so you don't lock yourself out)
sudo ufw allow ssh

# Allow the 2009scape game port (default is 43594)
sudo ufw allow 43594/tcp

# Enable the firewall
sudo ufw enable

Automate Database Backups: Use cron to schedule a daily backup.

# Open the cron table for editing
crontab -e

Add the following line to the end of the file. It will run mysqldump every day at 3:00 AM.

0 3 * * * mysqldump -u rsadmin -pYourSuperSecurePassword global > /home/scape/db_backups/2009scape_db_backup_$(date +\%F).sql

Note: Create the backup directory first: mkdir ~/db_backups. Also, there is no space between -p and your password in the cron command.

Updating the Server:

# Stop the service
sudo systemctl stop 2009scape.service

# As the 'scape' user, pull the latest code and rebuild
su - scape
cd ~/2009scape/
git pull
cd Server/
./mvnw clean package
exit

# Start the service again
sudo systemctl start 2009scape.service

Part B: Game Client Setup (Multi-Platform)
This section covers building and running the game client. The client requires Java 17.

1. Client Prerequisites & Setup
git clone [https://gitlab.com/2009scape/rt4-client.git](https://gitlab.com/2009scape/rt4-client.git)
cd rt4-client

2. Java 17 Installation & Configuration
For Debian / Ubuntu:
Install the JDK: sudo apt install -y openjdk-17-jdk

Set JAVA_HOME permanently:

echo 'JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"' | sudo tee -a /etc/environment
source /etc/environment

For Arch / Artix Linux:
Install the JDK: sudo pacman -S jdk17-openjdk

Set the default Java version: sudo archlinux-java set java-17-openjdk

Set JAVA_HOME permanently in your shell's config file (.bashrc or .zshrc).

# Add this line to the end of the file (e.g., nano ~/.bashrc)
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk"

Reload the configuration: source ~/.bashrc

3. Verify and Run the Client
Verify your setup:

javac -version
echo $JAVA_HOME

Run the client. The first run may take a few minutes.

./gradlew run

4. Connecting to Your Local Server
Find your server's LAN IP address (run on the server): hostname -I | awk '{print $1}'

Edit the client's config.json file (nano config.json).

Change the IP address to your server's LAN IP and ensure the port is correct.

{
  "ip_management": "192.168.1.15",
  "client_version": 602,
  "port_management": 43594,
  "port_advertisement": 43594
}

Relaunch the client: ./gradlew run
