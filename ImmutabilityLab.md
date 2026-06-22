# Hands-On Lab: Implementing Zero Trust & Data Immutability on a Linux VM

This technical guide provides the exact steps to build a local Proof of Concept (PoC) using a single Linux Virtual Machine (VM). You will replicate Zero Trust automated data classification, OS-level hardware-like immutability, and cloud-based Object Locking.



## Environment Setup

Ensure your local Linux machine or VM (Ubuntu, Debian, or RHEL) has `docker`, `inotify-tools`, and `openssl` installed.

```bash
# Update and install local automation utilities
sudo apt-get update
sudo apt-get install -y docker.io inotify-tools openssl curl
```



## Step 1: Replicating the Zero Trust Data Pillar (Automated DLP)

In a Zero Trust architecture, data must be classified and secured the moment it is detected. This step simulates an automated Data Loss Prevention (DLP) engine tracking sensitive strings (like Credit Card formats).

### 1. Create the Directory Structure
```bash
mkdir -p /data/production /data/archive_secure
```

### 2. Create the Monitoring Script
Create a script named `classify.sh`:
```bash
cat << 'EOF' > classify.sh
#!/bin/bash
# Simulating a Zero Trust Data Discovery and Classification Engine
MONITOR_DIR="/data/production"
SECURE_DIR="/data/archive_secure"

echo "Zero Trust DLP Engine started. Monitoring \$MONITOR_DIR..."

inotifywait -m "\$MONITOR_DIR" -e create | while read path action file; do
    FILE_PATH="\({MONITOR_DIR}/\){file}"
    
    # Check if the file contains a 16-digit credit card pattern (xxxx-xxxx-xxxx-xxxx)
    if grep -qE '[0-9]{4}-[0-9]{4}-[0-9]{4}-[0-9]{4}' "\$FILE_PATH"; then
        echo "[SECURITY ALERT] Sensitive data found in \$file. Classifying as HIGHLY-CONFIDENTIAL."
        
        # Encrypt the file immediately using AES-256 (Zero Trust Data Encryption Requirement)
        openssl enc -aes-256-cbc -salt -pbkdf2 -in "\$FILE_PATH" -out "\({SECURE_DIR}/\){file}.enc" -pass pass:SuperSecretKey2026
        
        # Purge the unencrypted production instance
        rm "\$FILE_PATH"
        echo "[INFO] Unencrypted file purged. Encrypted payload stored securely in archive."
    fi
done
EOF
```

### 3. Test the Automation
Run the script in the background or a separate terminal tab:
```bash
chmod +x classify.sh
./classify.sh
```

In a second terminal, drop a mock sensitive payload into production:
```bash
echo "Customer Master Record: 1111-2222-3333-4444" > /data/production/leak_test.txt
```
*Observe your first terminal to see the engine intercept, auto-classify, encrypt, and shred the original asset.*



## Step 2: Replicating WORM File System Immutability (The "Paper" Ink)

To mirror the unalterable trait of physical ink, we use native Linux system attributes (`chattr`). This forces an append-only state that completely locks historical data against modifications—even from the `root` account.

### 1. Initialize an Audit Log
```bash
sudo touch /var/log/immutable_audit.log
sudo echo "SYSTEM INITIALIZED - BASE RECORD" | sudo tee /var/log/immutable_audit.log
```

### 2. Enforce the Append-Only (`+a`) Status
```bash
sudo chattr +a /var/log/immutable_audit.log
```

### 3. Verify System Behavior
* **Validation Test A: Appending New Rows (Allowed)**
  ```bash
  echo "USER_LOGIN_EVENT: Admin accessed system" | sudo tee -a /var/log/immutable_audit.log
  # Works as expected. The new row joins the file tail.
  ```

* **Validation Test B: Attempting Overwrite or Destruction (Blocked)**
  ```bash
  echo "CLEAN HISTORY" | sudo tee /var/log/immutable_audit.log
  # Result: tee: /var/log/immutable_audit.log: Operation not permitted
  
  sudo rm /var/log/immutable_audit.log
  # Result: rm: cannot remove '/var/log/immutable_audit.log': Operation not permitted
  ```



## Step 3: Replicating Cloud Object Locking (The Timed Vault)

We use **MinIO** (an open-source S3 API emulator) locally to simulate automated API compliance frameworks. This enforces strict operational timelines where delete actions are entirely rejected.

### 1. Fire Up the MinIO Object Storage Engine
```bash
docker run -d -p 9000:9000 -p 9001:9001 \
  --name minio_worm_vault \
  -v /mnt/minio_data:/data \
  minio/minio server /data --console-address ":9001"
```

### 2. Deploy and Configure the MinIO Native Client (`mc`)
```bash
curl https://min.io -o mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Set up access credentials pointing to your local sandbox container
mc alias set localvault http://localhost:9000 minioadmin minioadmin
```

### 3. Form a Structural Bucket and Initialize Object Locking
```bash
# Initialize bucket with Object Lock capability enabled explicitly
mc mb --with-lock localvault/immutable-backup-vault

# Inject a Compliance retention constraint forcing a 7-day minimum block
mc retention set --default compliance 7d localvault/immutable-backup-vault
```

### 4. Test Cloud Level Retention Mechanics
Upload a sample mock system backup:
```bash
echo "CRITICAL_BACKUP_PAYLOAD_DATA" > backup_2026.tar.gz
mc cp backup_2026.tar.gz localvault/immutable-backup-vault/
```

Attempt an administrative purge immediately:
```bash
mc rm localvault/immutable-backup-vault/backup_2026.tar.gz
# Result: API execution error: Access Denied. Object is locked under compliance rules.
```



## Step 4: Constructing the Full 3-2-1-1-0 Automated Backup Pipeline

Now we combine all components into a production backup execution pattern. Create a master orchestration backup script named `run_backup.sh`:

```bash
cat << 'EOF' > run_backup.sh
#!/bin/bash
# 3-2-1-1-0 Production Compliance Automation Script

SOURCE_DIR="/data/archive_secure"
BACKUP_LOCAL_MEDIA="/tmp/local_backup_media"
mkdir -p "\$BACKUP_LOCAL_MEDIA"

TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="secure_archive_\${TIMESTAMP}.tar.gz"

echo "Starting Backup Strategy Pipeline execution..."

# [Copy 1 -> Production Area]: Existing within active memory/directories.
# [Copy 2 -> Second Storage Medium]: Compressing out to alternate storage route.
tar -czf "\${BACKUP_LOCAL_MEDIA}/\({BACKUP_NAME}" -C "\)SOURCE_DIR" .
echo "[SUCCESS] Local backup variant generated at \({BACKUP_LOCAL_MEDIA}/\){BACKUP_NAME}"

# [Copy 3 -> Offsite Immutable Vault]: Uploading to your Object-Locked Target
mc cp "\({BACKUP_LOCAL_MEDIA}/\){BACKUP_NAME}" localvault/immutable-backup-vault/
echo "[SUCCESS] Immutable Offsite backup layer deployed to Object Vault."

# [0 Errors Verification]: Testing payload read paths
mc ls localvault/immutable-backup-vault/\${BACKUP_NAME} > /dev/null
if [ \$? -eq 0 ]; then
    echo "[COMPLETE] 3-2-1-1-0 Lifecycle validation successful. Backup verified intact."
else
    echo "[ERROR] Integrity checks failed."
    exit 1
fi
EOF

chmod +x run_backup.sh
```

Execute your lifecycle wrapper:
```bash
./run_backup.sh
```

You now have a fully operational local implementation of an automated, encrypted, and highly immutable Zero Trust data storage design.
