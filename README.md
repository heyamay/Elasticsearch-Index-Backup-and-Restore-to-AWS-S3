# Elasticsearch-Index-Backup-and-Restore-to-AWS-S3
  
This document provides detailed instructions for configuring Elasticsearch to take snapshots 
(backups) of its indexes to an AWS S3 bucket and how to restore them. It covers everything 
from verifying Elasticsearch installation to automating the backup process. 

## 1. Prerequisites 
Before you begin, ensure you have the following: 
● Elasticsearch Installation: Elasticsearch should be installed on our Rocky Linux server. 
If not, follow the installation steps in Section 2.
● 8 indices to back up (replace index1 to index8 with actual names after ins 
● AWS S3 Bucket: An S3 bucket dedicated for Elasticsearch snapshots. 
○ Recommended S3 Bucket Name: my-backup-elasticsearch 
○ Region: ap-south-1 
● AWS IAM User Credentials: An IAM user with programmatic access (Access Key ID 
and Secret Access Key) and permissions to s3:GetObject, s3:ListBucket, 
s3:PutObject, s3:DeleteObject for the designated S3 bucket. 
○ Provided Credentials: 
■ aws_access_key = "AKIA32*******" 
■ aws_secret_key = 
"PGNntMdwSe4**************"

## 2. Verify Elasticsearch Status & Installation (if needed) 
First, let's check if Elasticsearch is already running on your Rocky Linux server. 
ssh rocky@server-ip 
2.1 Check Elasticsearch Service Status 
Open your terminal and run the following command: 
sudo systemctl status elasticsearch 
● If Elasticsearch is active (running): Great! You can skip to Section 3. 
● If Elasticsearch is inactive (dead) or not found: You need to start or install it. 
Proceed to Section 2.2. 
2.2 Install Elasticsearch on Rocky Linux (if not installed) 
If Elasticsearch is not installed, follow these steps. 
2.2.1 Import Elasticsearch GPG Key 
This key is used to verify the Elasticsearch packages. 
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch 
2.2.2 Create Elasticsearch Repository File 
Create a new repository file for Elasticsearch. 
sudo nano /etc/yum.repos.d/elasticsearch.repo 
Paste the following content: 
```
[elasticsearch] 
name=Elasticsearch repository for 8.x packages 
baseurl=https://artifacts.elastic.co/packages/8.x/yum 
gpgcheck=1 
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch 
enabled=1 
autorefresh=1 
type=rpm-md
```
2.2.3 Install Elasticsearch 
Now, install Elasticsearch using dnf 
sudo dnf install elasticsearch -y 

2.2.4 Configure Elasticsearch (Optional but Recommended) 
By default, Elasticsearch listens on localhost. For production, we might want to adjust 
network.host in its configuration file. 
sudo nano /etc/elasticsearch/elasticsearch.yml 
Find and uncomment/modify 
network.host: 0.0.0.0 
http.port: 9200 

2.2.5 Start and Enable Elasticsearch Service 
sudo systemctl start elasticsearch 
sudo systemctl enable elasticsearch 
sudo systemctl status elasticsearch 
Verify that the status is now active (running). 

2.2.6 Verify Cluster: 
curl -X GET "localhost:9200/_cluster/health?pretty" 
○ Check for "status": "green" or "yellow". If "red", contact your manager. 

2.2.7 List Indices (confirm 8 indices after data is loaded): 
curl -X GET "localhost:9200/_cat/indices?v" 
○ Note index names (e.g., index1 to index8). 

## 3. Install and Configure the repository-s3 Plugin 
The repository-s3 plugin allows Elasticsearch to store snapshots in AWS S3. 

3.1 Create directory: 
sudo mkdir -p /opt/elasticsearch-backup 

3.2 Set permissions: 
sudo chown elasticsearch:elasticsearch /opt/elasticsearch-backup 
sudo chmod 755 /opt/elasticsearch-backup 

3.3 Install the Plugin 
Run the following command on your Elasticsearch server.  
Navigate to Elasticsearch directory: 
cd /usr/share/elasticsearch 
Install plugin: 
sudo bin/elasticsearch-plugin install repository-s3 
Type y and after installation, you must restart Elasticsearch for the plugin to take effect. 
sudo systemctl restart elasticsearch 
sudo systemctl status elasticsearch 
Verify: 
curl -X GET "localhost:9200/_cluster/health?pretty" 

3.4 Securely Store AWS Credentials 
Elasticsearch requires access to your AWS S3 bucket. It's best practice to store credentials 
securely using Elasticsearch's keystore. 

3.4.1 Create the Keystore (if it doesn't exist) 
cd /usr/share/elasticsearch 
sudo bin/elasticsearch-keystore create 

3.4.2 Add AWS Access Key and Secret Key to Keystore 
Add your AWS access key and secret key to the keystore. Replace the values with the actual 
credentials provided by your manager. 
Add credentials: 
sudo bin/elasticsearch-keystore add s3.client.default.access_key 
● Enter: AKIA3**********8 
sudo bin/elasticsearch-keystore add s3.client.default.secret_key 
● Enter: PGNntMdwSe***************** 
Important: You might be prompted to enter the values interactively. After adding, you need to 
restart Elasticsearch again for the changes to the keystore to be recognized. 
sudo systemctl restart elasticsearch 
Reload settings: 
curl -X POST "localhost:9200/_nodes/reload_secure_settings?pretty" 
● Check "failed": 0. 

## 4. Register an S3 Repository 
A repository is a location where Elasticsearch snapshots are stored. You need to register your 
S3 bucket as a repository. 
Create S3 bucket (if not created): 
● Log in to AWS Console > S3 > Create bucket. 
● Name: my-backup-elasticsearch 
● Region: ap-south-1 
● Enable Block all public access. 
● Click Create bucket.

4.1 Register the Repository 
Use the following curl command to register the repository. 
● Repository Name: my_s3_repository (You can choose any descriptive name) 
● Bucket Name: pnq-backup-elasticsearch 
● Region: ap-south-1 
```
curl -X PUT "localhost:9200/_snapshot/my_s3_repository?pretty" -H 
'Content-Type: application/json' -d' 
{ 
"type": "s3", 
"settings": { 
"bucket": "pnq-backup-elasticsearch", 
"region": "ap-south-1", 
"base_path": "elasticsearch-backup/", 
"compress": true 
} 
}’ 
Expected Output: 
{ 
"acknowledged" : true 
}
```

4.2 Verify the Repository 
You can verify that the repository has been successfully registered: 
curl -X GET "localhost:9200/_snapshot/my_s3_repository?pretty" 
Expected Output: You should see details about your registered repository. 
```
{ 
} 
"my_s3_repository" : { 
"type" : "s3", 
"settings" : { 
"bucket" : "pnq-backup-elasticsearch", 
"region" : "ap-south-1", 
"compress" : "true" 
} 
}
```
## 5. Take a Snapshot (Backup) 
Now that the repository is set up, you can take snapshots.

5.1 Take a Manual Snapshot of Specific Indexes (Individual Index Restore 
Use Case) 
As per our requirement, the use case is individual index restore, and y weave 8 indexes. We 
can specify which indexes to back up. Let's assume our indexes are index_1, index_2, ..., 
index_8. 
To take a snapshot of index_1 and index_2 (we can extend this to all 8 indexes by listing 
them) 
Take a manual snapshot to verify the setup:
```
curl -X PUT 
"localhost:9200/_snapshot/my_s3_repository/snapshot_20250616_index1_in
dex2?wait_for_completion=true&pretty" -H 'Content-Type: 
application/json' -d' 
{ 
} 
' 
"indices": "index_1,index_2", 
"ignore_unavailable": true, 
"include_global_state": false
```
● snapshot_20250616_index1_index2: This is the name of your snapshot. It's good 
practice to include the date and relevant index names. 
● wait_for_completion=true: This makes the curl command wait until the snapshot 
process is complete before returning. 
● "indices": "index_1,index_2": Specify the comma-separated list of indexes you 
want to back up. 
● "ignore_unavailable": true: If an index specified doesn't exist, the snapshot will 
proceed for available indexes. 
● "include_global_state": false: For individual index restore, you typically don't 
need to include the cluster's global state. 
To take a snapshot of ALL your indexes: 
```
curl -X PUT 
"localhost:9200/_snapshot/my_s3_repository/snapshot_all_20250616?wait_
 for_completion=true&pretty" -H 'Content-Type: application/json' -d' 
{ 
"indices": "*", 
"ignore_unavailable": true, 
"include_global_state": false 
} 
'
```

5.2 Verify Snapshots 
To list all snapshots in your repository: 
curl -X GET "localhost:9200/_snapshot/my_s3_repository/_all?pretty" 
To get information about a specific snapshot: 
curl -X GET 
"localhost:9200/_snapshot/my_s3_repository/snapshot_20250616_index1_in
 dex2?pretty" 
The state field in the output should be SUCCESS. 

## 6. Restore from a Snapshot 
Restoring involves bringing data back from a snapshot. Our use case is individual index restore.

6.1 Restore a Specific Index 
Important Considerations Before Restoring: 
● Close the Index: You must close the index you want to restore if it already exists in your 
cluster. If you restore an index with the same name while it's open, it will fail. 
● Rename Index (Optional but Recommended for Testing): For testing purposes, or if 
you don't want to overwrite the existing index, you can restore it to a different name. 

6.1.1 Close an Index (if it exists) 
If index1 already exists and is open, close it: 
curl -X POST "localhost:9200/index_1/_close?pretty" 

6.1.2 Perform the Restore 
To restore index_1 from snapshot_20250616_index1_index2 into a new index called 
restored_index_1: 
```
curl -X POST 
"localhost:9200/_snapshot/my_s3_repository/snapshot_20250616_index1_in
 dex2/_restore?pretty" -H 'Content-Type: application/json' -d' 
{ 
"indices": "index_1", 
"rename_pattern": "index_(.+)", 
"rename_replacement": "restored_index_$1", 
"include_aliases": false 
} 
'
```
● "indices": "index_1": Specifies the original index from the snapshot to restore. 
● "rename_pattern": "index_(.+)": A regular expression to match the original 
index name. 
● "rename_replacement": "restored_index_$1": The replacement pattern for the 
new index name. $1 refers to the captured group from rename_pattern. This will 
restore index_1 to restored_index_1. 
● "include_aliases": false: Typically, you don't include aliases during a targeted 
index restore. 
To restore an index to its original name (overwriting if it exists, make sure it's closed 
first!): 
```
curl -X POST 
"localhost:9200/_snapshot/my_s3_repository/snapshot_20250616_index1_in
 dex2/_restore?pretty" -H 'Content-Type: application/json' -d' 
{ 
"indices": "index_1" 
} 
'
```
After restoration, you can check the status of your indexes: 
curl -X GET "localhost:9200/_cat/indices?v" 

6.1.3 Open the Restored Index (if it was closed) 
If you closed the index before restoring, open it now: 
curl -X POST "localhost:9200/restored_index_1/_open?pretty" 

## 7. Automate Backups and Cleanup (Cron Jobs) 
As per our strategy: 
● Backup: Every Sunday at 3 AM 
● Cleanup old backups: Every Sunday at 6 AM (older than 28 days) 
We will use cron for scheduling these tasks. 

7.1 Create Backup Script 
First, create a script that takes the snapshot. 
sudo mkdir -p /opt/elasticsearch_backup 
sudo nano /opt/elasticsearch_backup/take_snapshot.sh 
Paste the following. Remember to replace 
index_1,index_2,index_3,index_4,index_5,index_6,index_7,index_8 with the 
actual comma-separated list of your 8 indexes. 
```
#!/bin/bash 
# --- Elasticsearch Backup Script --- 
# This script takes a snapshot of specified Elasticsearch indexes. 
# Define variables 
ES_HOST="localhost:9200" 
REPOSITORY_NAME="my_s3_repository" 
SNAPSHOT_NAME="weekly_snapshot_$(date +%Y%m%d_%H%M%S)" # Unique name 
with date and time 
# Replace with your actual comma-separated list of indexes 
INDEXES_TO_BACKUP="index_1,index_2,index_3,index_4,index_5,index_6,ind
 ex_7,index_8" 
echo "Starting Elasticsearch snapshot creation: ${SNAPSHOT_NAME} for 
indexes: ${INDEXES_TO_BACKUP}" 
# Take the snapshot 
curl -X PUT 
"${ES_HOST}/_snapshot/${REPOSITORY_NAME}/${SNAPSHOT_NAME}?wait_for_com
 pletion=true" \ -H 'Content-Type: application/json' \ -d"{ 
\"indices\": \"${INDEXES_TO_BACKUP}\", 
\"ignore_unavailable\": true, 
\"include_global_state\": false 
}" > /tmp/es_snapshot_${SNAPSHOT_NAME}.log 2>&1 
# Check the snapshot creation status 
if grep -q '"state":"SUCCESS"' /tmp/es_snapshot_${SNAPSHOT_NAME}.log; 
then 
echo "Snapshot ${SNAPSHOT_NAME} created successfully." 
else 
echo "Error creating snapshot ${SNAPSHOT_NAME}. Check 
/tmp/es_snapshot_${SNAPSHOT_NAME}.log for details." 
fi 
# Clean up log file 
rm -f /tmp/es_snapshot_${SNAPSHOT_NAME}.log
```
Save and exit. 
Make the script executable: 
sudo chmod +x /opt/elasticsearch_backup/take_snapshot.sh 

7.2 Create Cleanup Script 
Install jq (for robust JSON parsing): 
sudo dnf install jq -y 
This script will list all snapshots and delete those older than 28 days. 
sudo nano /opt/elasticsearch_backup/cleanup_snapshots.sh 
Paste the following: 
```
#!/bin/bash 
ES_HOST="localhost:9200" 
REPO="pnq-backup-repo" 
RETENTION_DAYS=28 
LOG="/opt/elasticsearch-backup/cleanup.log" 
echo "Cleaning snapshots older than $RETENTION_DAYS days" >> $LOG 
SNAPSHOTS=$(curl -s "$ES_HOST/_snapshot/$REPO/_all" | jq -r 
'.snapshots[] | "\(.snapshot) \(.start_time_in_millis)"') 
CURRENT_TIME=$(date +%s) 
while read -r SNAPSHOT TIME; do 
SNAPSHOT_TIME=$((TIME / 1000)) 
AGE_DAYS=$(((CURRENT_TIME - SNAPSHOT_TIME) / 86400)) 
if [ $AGE_DAYS -gt $RETENTION_DAYS ]; then 
echo "Deleting snapshot: $SNAPSHOT" >> $LOG 
curl -X DELETE "$ES_HOST/_snapshot/$REPO/$SNAPSHOT" >> $LOG 2>&1 
fi 
done <<< "$SNAPSHOTS"
```
Save and exit. 
Make the script executable: 
sudo chmod +x /opt/elasticsearch_backup/cleanup_snapshots.sh 

7.3 Schedule with Cron 
Open the cron table for editing: 
sudo crontab -e 
Insert 
# Backup Elasticsearch every Sunday at 3 AM 
0 3 * * 0 /opt/elasticsearch_backup/take_snapshot.sh >> 
/var/log/elasticsearch_backup.log 2>&1 
# Clean up old Elasticsearch snapshots every Sunday at 6 AM (older 
than 28 days) 
0 6 * * 0 /opt/elasticsearch_backup/cleanup_snapshots.sh >> 
/var/log/elasticsearch_cleanup.log 2>&1 
● 0 3 * * 0: This means "at 03:00 on Sunday" (0 = Sunday, 1 = Monday, etc.). 
● 0 6 * * 0: This means "at 06:00 on Sunday". 
● >> /var/log/elasticsearch_backup.log 2>&1: Redirects both standard output 
and standard error to a log file. 
Save and exit. 

## 8: Monitor and Maintain 

1. Check snapshot status: 
curl -X GET "localhost:9200/_snapshot/pnq-backup-repo/_status?pretty" 
2. Check logs: 
○ Elasticsearch: sudo tail -f /var/log/elasticsearch/elasticsearch.log 
○ Scripts: cat /opt/elasticsearch-backup/*.log 
3. Monitor S3 bucket in AWS Console to confirm snapshots are created/deleted.
