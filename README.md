# Vaultwarden AWS
Instructions for deploying Vaultwarden on AWS EC2 with nightly S3 backups

### AWS
1) Create a new S3 Bucket  
Settings  
   ```
   Bucket name - Unique
   Region - Your prefered Region
   ACLs disabled (recommended)
   Enabled - Block all public access
   Server-side encryption with Amazon S3 managed keys (SSE-S3)
   Enabled - Bucket Key
   ```
1) Provision a new EC2 Instance  
I chose a t4g.nano which is arm64 architecture, the instructions will differ if you choose a different architecture.  
I chose the Amazon Linux 2023 AMI, the instructions will differ if you choose a different AMI.  
I provisioned the instance in the same Region as my S3 bucket to reduce costs.  
Settings  
   ```
   Name
   Amazon Linux 2023 AMI
   Architecture - 64-bit(Arm)
   Instance Type XXg.X
   Key Pair - select an existing or create new
   Auto-assign public IP - Disable (We will allocate an Elastic IP to prevent it from changing when the Instance is Stopped or Restarted)
   Security Group - See Below
   Storage - 50GB gp2 (Just a guess, I do not feel like IO is a priority and I am not sure how large the database/file storage will get)
   IAM Profile - See Below
   Shutdown behavior - Stop
   Termination protection - Enable
   Stop protection - Enable
   ```
   Security Group Settings  
   ```
   name	    sg-  	HTTP	tcp	80	    0.0.0.0/0	    -
   name	    sg-  	HTTP	tcp	80	    ::/0	        -
   name	    sg-  	ssh	    tcp	22	    0.0.0.0/0	    -
   name	    sg-  	ssh	    tcp	22	    ::/0	        -
   name	    sg-  	HTTPS	tcp	443	    0.0.0.0/0	    -
   name	    sg-  	HTTPS	tcp	443	    ::/0	        -
   ```
   IAM Role
   ```
   AmazonSSMManagedInstanceCore
   CloudWatchAgentAdminPolicy
   CloudWatchAgentServerPolicy
   EC2-S3-backup-policy - See Below
   ```
   EC2-S3-backup-policy
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "S3ConsoleAccess",
               "Effect": "Allow",
               "Action": [
                   "s3:GetAccountPublicAccessBlock",
                   "s3:GetBucketAcl",
                   "s3:GetBucketLocation",
                   "s3:GetBucketPolicyStatus",
                   "s3:GetBucketPublicAccessBlock",
                   "s3:ListAccessPoints",
                   "s3:ListAllMyBuckets"
               ],
               "Resource": [
                   "arn:aws:s3:::YOUR-BUCKET-NAME-HERE"
               ]
           },
           {
               "Sid": "ListObjectsInBucket",
               "Effect": "Allow",
               "Action": "s3:ListBucket",
               "Resource": [
                   "arn:aws:s3:::YOUR-BUCKET-NAME-HERE"
               ]
           },
           {
               "Sid": "AllObjectActions",
               "Effect": "Allow",
               "Action": "s3:*Object",
               "Resource": [
                   "arn:aws:s3:::YOUR-BUCKET-NAME-HERE/*"
               ]
           }
       ]
   }
   ```
1) Allocate an Elastic IP Address with Default Settings  
1) Associate the new Elastic IP Address with the vaultwarden EC2 Instance  
1) Configure your domain DNS to point to the Elastic IP Address  

### EC2 Instance
1) Install Docker & Docker-compose  
    ```
    sudo yum install docker.aarch64
    sudo systemctl start docker.service
    sudo yum install -y python3.11
    sudo yum install -y python3-devel.$(uname -m) libffi-devel openssl-devel
    sudo yum install -y python3-pip.noarch
    sudo python3 -m pip install -U pip
    python3 -m pip install docker-compose
    ```
1) Create the vaultwarden directory  
`mkdir /home/ec2-user/vw`  
1) Create the docker-compose.yml  
   1) Gather this info
      1) DOMAIN: Your domain; vaultwarden needs to know it's https to work properly with attachments
      1) ADMIN_TOKEN: `echo -n "CHANGE_ME_SecretPassword" | argon2 "$(openssl rand -base64 32)" -e -id -k 19456 -t 2 -p 1 | sed 's#\$#\$\$#g'`
      1) PUSH_INSTALLATION_ID: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification#enable-mobile-client-push-notification  
      1) PUSH_INSTALLATION_KEY: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification#enable-mobile-client-push-notification  
      1) SMTP_HOST: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) SMTP_FROM: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) SMTP_PORT: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) SMTP_SECURITY: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) SMTP_USERNAME: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) SMTP_PASSWORD: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration  
      1) EMAIL: The email address to use for ACME registration.  

   `vi /home/ec2-user/vw/docker-compose.yml`  
   ```yaml
   version: '3'

   services:
     vaultwarden:
       image: vaultwarden/server:latest
       container_name: vaultwarden
       restart: always
       environment:
         DOMAIN: "CHANGE_ME"  # Your domain; vaultwarden needs to know it's https to work properly with attachments
         SIGNUPS_ALLOWED: 'false'
         ADMIN_TOKEN: CHANGE_ME
         PUSH_ENABLED: 'true'
         PUSH_INSTALLATION_ID: 'CHANGE_ME'
         PUSH_INSTALLATION_KEY: 'CHANGE_ME'
         SMTP_HOST: 'CHANGE_ME'
         SMTP_FROM: 'CHANGE_ME'
         SMTP_PORT: CHANGE_ME
         SMTP_SECURITY: 'CHANGE_ME'
         SMTP_USERNAME: 'CHANGE_ME'
         SMTP_PASSWORD: 'CHANGE_ME'
         SHOW_PASSWORD_HINT: 'false'
         LOG_FILE: '/data/vaultwarden.log'
       volumes:
         - ./vw-data:/data

     caddy:
       image: caddy:2
       container_name: caddy
       restart: always
       ports:
         - 80:80  # Needed for the ACME HTTP-01 challenge.
         - 443:443
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile:ro
         - ./caddy-config:/config
         - ./caddy-data:/data
       environment:
         DOMAIN: "CHANGE_ME"  # Your domain.
         EMAIL: "CHANGE_ME"   # The email address to use for ACME registration.
         LOG_FILE: "/data/access.log"

     fail2ban:
       container_name: fail2ban
       restart: always
       image: crazymax/fail2ban:latest
       environment:
         - TZ=UTC
         - F2B_DB_PURGE_AGE=30d
         - F2B_LOG_TARGET=/data/fail2ban.log
         - F2B_LOG_LEVEL=INFO
         - F2B_IPTABLES_CHAIN=INPUT

       volumes:
         - ./fail2ban:/data
         - ./vw-data:/vaultwarden:ro

       network_mode: "host"

       privileged: true
       cap_add:
         - NET_ADMIN
         - NET_RAW
   ```
1) Create the Caddyfile  
`vi /home/ec2-user/vw/Caddyfile`  
   ```
   {$DOMAIN}:443 {
     log {
       level INFO
       output file {$LOG_FILE} {
         roll_size 10MB
         roll_keep 10
       }
     }

     # Use the ACME HTTP-01 challenge to get a cert for the configured domain.
     tls {$EMAIL}

     # This setting may have compatibility issues with some browsers
     # (e.g., attachment downloading on Firefox). Try disabling this
     # if you encounter issues.
     encode gzip

     # Proxy everything Rocket
     reverse_proxy vaultwarden:80 {
          # Send the true remote IP to Rocket, so that vaultwarden can put this in the
          # log, so that fail2ban can ban the correct IP.
          header_up X-Real-IP {remote_host}
     }
   }
   ```
1) Start the container stack  
`docker-compose -f /home/ec2-user/vw/docker-compose.yml up -d`  
1) Create the fail2ban rules
   1) Actions  
   `vi vw/fail2ban/action.d/iptables.local`  
      ```ini
      [Init]
      blocktype = DROP
      [Init?family=inet6]
      blocktype = DROP
      ```
   1) Filters  
   `vi vw/fail2ban/filter.d/vaultwarden.local`  
      ```ini
      [INCLUDES]
      before = common.conf

      [Definition]
      failregex = ^.*Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
      ignoreregex =
      ```
       `vi vw/fail2ban/filter.d/vaultwarden-admin.local`  
         ```ini
         [INCLUDES]
         before = common.conf

         [Definition]
         failregex = ^.*Invalid admin token\. IP: <ADDR>.*$
         ignoreregex =
         ```
   1) Jails  
   `vi vw/fail2ban/jail.d/vaultwarden.local`  
      ```ini
      [vaultwarden]
      enabled = true
      port = 80,443,8081
      filter = vaultwarden
      banaction = %(banaction_allports)s
      logpath = /vaultwarden/vaultwarden.log
      maxretry = 5
      bantime = 14400
      findtime = 14400
      chain = FORWARD
      ```
      `vi vw/fail2ban/jail.d/vaultwarden-admin.local`  
      ```ini
      [vaultwarden-admin]
      enabled = true
      port = 80,443
      filter = vaultwarden-admin
      banaction = %(banaction_allports)s
      logpath = /vaultwarden/vaultwarden.log
      maxretry = 3
      bantime = 14400
      findtime = 14400
      chain = FORWARD
      ```
1) Restart the container stack  
`docker-compose -f /home/ec2-user/vw/docker-compose.yml down`  
`docker-compose -f /home/ec2-user/vw/docker-compose.yml up -d`  

## Backups
### File Structure
```bash
vw  
├── Caddyfile  
├── caddy-config  
│   └── caddy  
│       └── autosave.json  
├── caddy-data  
│   ├── access-< date >.log.gz  
│   ├── access.log  
│   └── caddy  
│       ├── acme  
│       │   ├── acme-staging-v02.api.letsencrypt.org-directory  
│       │   │   ├── challenge_tokens  
│       │   │   └── users  
│       │   │       └── your_email@your_email.com  
│       │   │           ├── your_email.json  
│       │   │           └── your_email.key  
│       │   ├── acme-v02.api.letsencrypt.org-directory  
│       │   │   ├── challenge_tokens  
│       │   │   └── users  
│       │   │       └── your_email@your_email.com  
│       │   │           ├── your_email.json  
│       │   │           └── your_email.key  
│       │   └── acme.zerossl.com-v2-dv90  
│       │       ├── challenge_tokens  
│       │       └── users  
│       │           └── your_email@your_email.com  
│       │               ├── your_email.json  
│       │               └── your_email.key  
│       ├── certificates  
│       │   └── acme-v02.api.letsencrypt.org-directory  
│       │       └── your_domain.com  
│       │           ├── your_domain.com.crt  
│       │           ├── your_domain.com.json  
│       │           └── your_domain.com.key  
│       ├── locks  
│       └── ocsp  
│           ├── your_domain.com-< random_id >  
│           └── your_domain.com-< random_id >  
├── docker-compose.yml  
├── fail2ban  
│   ├── action.d  
│   │   └── iptables.local  
│   ├── db  
│   │   └── fail2ban.sqlite3  
│   ├── fail2ban.log  
│   ├── filter.d  
│   │   ├── vaultwarden-admin.local  
│   │   └── vaultwarden.local  
│   └── jail.d  
│       ├── vaultwarden-admin.local  
│       └── vaultwarden.local  
└── vw-data  
    ├── attachments          # Each attachment is stored as a separate file under this dir.  
    │   └── < uuid >         # (The attachments dir won't be present if no attachments have been created.)  
    │       └── < random_id >  
    ├── config.json          # Stores admin page config; only exists if the admin page has been enabled before.  
    ├── db.sqlite3           # Main SQLite database file.  
    ├── db.sqlite3-shm       # SQLite shared memory file (not always present).  
    ├── db.sqlite3-wal       # SQLite write-ahead log file (not always present).  
    ├── icon_cache           # Site icons (favicons) are cached under this dir.  
    │   ├── < domain >.png  
    │   ├── example.com.png  
    │   ├── example.net.png  
    │   └── example.org.png  
    ├── rsa_key.der          # `rsa_key.*` files are used to sign authentication tokens.  
    ├── rsa_key.pem  
    ├── rsa_key.pub.der  
    └── sends                # Each Send attachment is stored as a separate file under this dir.  
        └── < uuid >         # (The sends dir won't be present if no Send attachments have been created.)  
            └── < random_id >  
```
### First Backup  
1) Install SQLite  
`sudo yum install sqlite`  
1) Create a backup directory  
`mkdir /home/ec2-user/backup`  
1) Rsync files to the backup directory  
`sudo rsync -azvPc --exclude '*.sqlite*' /home/ec2-user/vw/ /home/ec2-user/backup`  
1) Backup the SQLite databases  
`sudo sqlite3 -cmd ".timeout 30000" "file:/home/ec2-user/vw/vw-data/db.sqlite3?mode=ro" "VACUUM INTO '/home/ec2-user/backup/vw-data/db.sqlite3'"`  
`sudo sqlite3 -cmd ".timeout 30000" "file:/home/ec2-user/vw/fail2ban/db/fail2ban.sqlite3?mode=ro" "VACUUM INTO '/home/ec2-user/backup/fail2ban/db/fail2ban.sqlite3'"`  
1) Download restic from https://github.com/restic/restic/releases/latest  
    ```bash
    sudo cp restic-CHANGE_ME... /usr/bin/restic
    sudo chmod a+x /usr/bin/restic
    sudo setcap cap_dac_read_search=+ep /usr/bin/restic
    ```
1) Create a password file for restic  
    `vi /home/ec2-user/.resticpass`  
    ```txt
    CHANGE_ME_SuperSecretPassword
    ```
    `chmod 600 /home/ec2-user/.resticpass`  
1) Send the backup to AWS S3
    ```bash
    sudo restic init -o s3.storage-class=STANDARD_IA -r 's3:https://s3.amazonaws.com/YOUR-BUCKET-NAME-HERE' -p /home/ec2-user/.resticpass
    sudo restic backup -o s3.storage-class=STANDARD_IA -r 's3:https://s3.amazonaws.com/YOUR-BUCKET-NAME-HERE' -p /home/ec2-user/.resticpass /home/ec2-user/backup
    ```

### Automate the Backups  
1) Create the backup script  
   `mkdir /home/ec2-user/scripts`  
   `vi /home/ec2-user/scripts/backup.sh`  
    ```bash
    #!/bin/bash
    ####################################
    #
    # Backup to s3
    #
    ####################################
    rsync -azvPc --exclude '*.sqlite*' /home/ec2-user/vw/ /home/ec2-user/backup

    busy_timeout=30000 # in milliseconds
    sqlite3 -cmd ".timeout ${busy_timeout}" \
    "file:/home/ec2-user/vw/vw-data/db.sqlite3?mode=ro" \
    "VACUUM INTO '/home/ec2-user/backup/vw-data/db.sqlite3'"

    sqlite3 -cmd ".timeout ${busy_timeout}" \
    "file:/home/ec2-user/vw/fail2ban/db/fail2ban.sqlite3?mode=ro" \
    "VACUUM INTO '/home/ec2-user/backup/fail2ban/db/fail2ban.sqlite3'"

    restic backup -o s3.storage-class=STANDARD_IA -r 's3:https://s3.amazonaws.com/YOUR-BUCKET-NAME-HERE' -p /home/ec2-user/.resticpass /home/ec2-user/backup

    rm -f /home/ec2-user/backup/vw-data/db.sqlite3
    rm -f /home/ec2-user/backup/fail2ban/db/fail2ban.sqlite3
    ```
   `chmod a+x scripts/backup.sh`
1) Start cron  
   `sudo yum install -y cronie.aarch64`  
   `sudo crontab -e`  
    ```bash
    SHELL=/bin/bash
    PATH=/sbin:/bin:/usr/sbin:/usr/bin
    MAILTO=root

    # For details see man 4 crontabs

    # Example of job definition:
    # .---------------- minute (0 - 59)
    # |  .------------- hour (0 - 23)
    # |  |  .---------- day of month (1 - 31)
    # |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
    # |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
    # |  |  |  |  |
    # *  *  *  *  * user-name  command to be executed
    0  2  *  *  * root    /home/ec2-user/scripts/backup.sh
    0  3  *  *  0 root    restic forget -r 's3:https://s3.amazonaws.com/YOUR-BUCKET-NAME-HERE' -p /home/ec2-user/.resticpass --keep-last 7

   ```
   `sudo systemctl enable crond --now`  
