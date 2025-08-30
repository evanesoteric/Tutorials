
# vMaNGOS Full Production Setup Guide (Server & Client)

This guide provides a complete walkthrough for installing, building, and running a production-ready vMaNGOS server on Debian 11. It covers creating a dedicated user, setting up the MariaDB databases, compiling the core with playerbots, and running the server daemons as persistent systemd services.

Part A: Initial Server Preparation
This section covers updating the server, installing all necessary dependencies, and securing the MariaDB installation.

1. System Update & Dependencies
Log in as root and update your system:

# Update package lists and install all available upgrades
sudo apt update -y && sudo apt full-upgrade -y

Install all required packages for building and running the server:

sudo apt install -y build-essential gcc g++ make cmake wget curl rsync nano git tree libmariadb-dev libmariadb-dev-compat mariadb-server mariadb-client openssl libssl-dev libace-dev libbz2-dev net-tools zlib1g-dev libtbb-dev p7zip

2. Secure MariaDB Installation
Run the interactive security script for MariaDB.

sudo mysql_secure_installation

Answer the prompts as follows for a secure setup:

Enter current password for root? -> Press Enter (for none).

Switch to unix_socket authentication? -> Y

Change the root password? -> N (as unix_socket is secure).

Remove anonymous users? -> Y

Disallow root login remotely? -> Y

Remove test database and access to it? -> Y

Reload privilege tables now? -> Y

3. Create a Dedicated Server User
For security, the server should not run as root.

Create the vmangos user and their home directory:

sudo mkdir -p /var/lib/vmangos
sudo useradd -r -s /bin/bash -d /var/lib/vmangos vmangos
sudo chown vmangos:vmangos -R /var/lib/vmangos

Switch to the new user for the next steps:

su - vmangos

Part B: Compiling the vMaNGOS Core
All commands in this section should be run as the vmangos user.

1. Directory Setup & Source Code
Create the required directory structure:

# We are in /var/lib/vmangos
mkdir -p {vmangos,bak}
mkdir -p vmangos/{build,db,logs,etc,bin,data}
mkdir -p vmangos/logs/{mangosd,realmd,honor}

Clone the vMaNGOS core repository:

cd ~/vmangos
git clone -b development [https://github.com/vmangos/core](https://github.com/vmangos/core)

2. Build and Install the Core
Navigate to the build directory:

cd ~/vmangos/build

Run CMake to configure the build. This example enables playerbots and targets client build 5875.

cmake ~/vmangos/core -DDEBUG=0 -DSUPPORTED_CLIENT_BUILD=5875 -DUSE_EXTRACTORS=0 -DPLAYERBOTS=1 -DCMAKE_INSTALL_PREFIX=~/vmangos

Compile and install the server. This command uses all available processor cores (nproc) for a faster build.

make -j $(nproc) install

After this step, the ~/vmangos/bin and ~/vmangos/etc directories will be populated with the server executables and configuration files.

Part C: Database & Game Data Setup
1. Download & Prepare Game Data
Download the World Database:

cd ~/vmangos/db
# Note: This URL may become outdated. Check the vMaNGOS project for current links.
curl -LO [https://github.com/brotalnia/database/raw/master/world_full_14_june_2021.7z](https://github.com/brotalnia/database/raw/master/world_full_14_june_2021.7z)
p7zip -d world_full_14_june_2021.7z

Download the Client Data Files (Maps, DBCs, etc.):
This large archive is required for the server to function. You must download it manually in a browser and then transfer it to your server (e.g., with scp).

Download Link: https://download1075.mediafire.com/k5jhx5bp2dmgaUOxWq-7imrO8LqRXexzto2_i0LBON8uR8SVBPfwcmmTayVl08ACeMI3zNomKEVyXs7iDIcfbkBxfWUParms2zsUFclZDNrGjg-2JPf0Z6upqiOaBvCn--tWx9QoqLsXCEFX2chGJfu_DUpASXYnqdbxbOA4FpdFdQ/jaduikubh1wkdn6/data.7z

Place the downloaded data.7z file in /var/lib/vmangos/.

Extract the Client Data:

# As the vmangos user
cd ~
p7zip -d data.7z
# Ensure permissions are correct
chmod 755 -R data
# Move the extracted data to the correct location
mv data/* ~/vmangos/data/
rm -rf data

2. Create and Populate Databases
Create the Databases and User (as root/sudo):

sudo mysql

Run the following SQL commands. Replace #Super_Secret# with a strong password.

CREATE USER 'vmangos'@'localhost' IDENTIFIED BY '#Super_Secret#';
CREATE DATABASE `realmd` DEFAULT CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci;
CREATE DATABASE `mangos` DEFAULT CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci;
CREATE DATABASE `characters` DEFAULT CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci;
CREATE DATABASE `logs` DEFAULT CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON `realmd`.* TO 'vmangos'@'localhost';
GRANT ALL PRIVILEGES ON `mangos`.* TO 'vmangos'@'localhost';
GRANT ALL PRIVILEGES ON `characters`.* TO 'vmangos'@'localhost';
GRANT ALL PRIVILEGES ON `logs`.* TO 'vmangos'@'localhost';
FLUSH PRIVILEGES;
EXIT;

Populate the Databases (as vmangos user):

# Populate the main world database
mysql -u vmangos -p mangos < ~/vmangos/db/world_full_14_june_2021.sql

# Populate the base schemas for the other databases
cd ~/vmangos/core/sql/
mysql -u vmangos -p realmd < logon.sql
mysql -u vmangos -p characters < characters.sql
mysql -u vmangos -p logs < logs.sql

Part D: Final Configuration
1. Copy and Edit Config Files
Navigate to the etc directory and copy the templates:

cd ~/vmangos/etc
cp mangosd.conf.dist mangosd.conf
cp realmd.conf.dist realmd.conf

Edit realmd.conf:

nano realmd.conf

Find and update the LoginDatabaseInfo line with your password:
LoginDatabaseInfo = "127.0.0.1;3306;vmangos;#Super_Secret#;realmd"

Edit mangosd.conf:

nano mangosd.conf

Find and update the following lines with your password and the correct data path:
DataDir = "/var/lib/vmangos/vmangos/data"
LoginDatabaseInfo = "127.0.0.1;3306;vmangos;#Super_Secret#;realmd"
WorldDatabaseInfo = "127.0.0.1;3306;vmangos;#Super_Secret#;mangos"
CharacterDatabaseInfo = "127.0.0.1;3306;vmangos;#Super_Secret#;characters"
LogsDatabaseInfo = "127.0.0.1;3306;vmangos;#Super_Secret#;logs"

2. Set Realm IP Address
Set the IP address clients will use to connect. For a LAN server, use the server's local IP.

Find your server's LAN IP (as root/sudo):
hostname -I | awk '{print $1}'

Update the database (as vmangos):

mysql -u vmangos -p

USE realmd;
INSERT INTO `realmlist` (`id`,`name`, `address`) VALUES ('1','VMaNGOS', 'YOUR_SERVER_IP');
EXIT;

Replace YOUR_SERVER_IP with the address from the previous command.

Part E: Running as a Production Service
Create systemd services to manage the server daemons automatically.

Create the realmd service file (as root/sudo):

sudo nano /etc/systemd/system/vmangos-realmd.service

Paste the following:

[Unit]
Description=VMaNGOS Realm Daemon
After=network.target mariadb.service

[Service]
User=vmangos
Group=vmangos
Type=simple
WorkingDirectory=/var/lib/vmangos/vmangos/bin
ExecStart=/var/lib/vmangos/vmangos/bin/realmd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

Create the mangosd service file (as root/sudo):

sudo nano /etc/systemd/system/vmangos-mangosd.service

Paste the following:

[Unit]
Description=VMaNGOS World Daemon
Requires=vmangos-realmd.service
After=vmangos-realmd.service

[Service]
User=vmangos
Group=vmangos
Type=simple
WorkingDirectory=/var/lib/vmangos/vmangos/bin
ExecStart=/var/lib/vmangos/vmangos/bin/mangosd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

Enable and start the services:

sudo systemctl daemon-reload
sudo systemctl enable vmangos-realmd.service
sudo systemctl enable vmangos-mangosd.service
sudo systemctl start vmangos-realmd.service
sudo systemctl start vmangos-mangosd.service

Check the status:

sudo systemctl status vmangos-realmd.service vmangos-mangosd.service

Part F: Client Setup (Arch Linux)
To connect to your server, modify your WoW 1.12 client's realmlist.wtf file. Open it with a text editor and change the content to:

set realmlist YOUR_SERVER_IP

Replace YOUR_SERVER_IP with the same IP you used in Part D, Step 2.
