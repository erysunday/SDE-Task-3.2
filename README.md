# SWE40006 Task 3: Deployment on AWS
This repository documents the Credit-level deployment work completed for Task 3.
The Pass-level deployment followed the supplied walkthrough and established a WordPress application running on an AWS EC2 instance with a local MariaDB 10.5 database. The configuration is thus not reproduced here.
My additional work for Task 3.2 focused on:
1. migrating the WordPress database from the EC2 instance to Amazon RDS for MariaDB, and
2. manually backing up and restoring the WordPress application files using Amazon S3.
## Deployment method
AWS infrastructure was manually configured using the AWS Management Console. The EC2 instance server and application stack was configured remotely through SSH using PuTTY, while WinSCP was used for manual file transfer between the EC2 instance and my local computer.
No IaC framework such as CloudFormation YAML, AWS CDK, or Terraform were created for this submission. Therefore, this repository documents the configuration, migration, backup and restoration process rather than demonstrating an automated infrastructure deployment.
## Technologies and tools used
### AWS services
- Amazon EC2
- Amazon RDS for MariaDB 10.11
- Amazon S3
- AWS Management console
### Server and application stack
- Amazon Linux 2023 (kernel-6.18)
- Apache HTTP server
- PHP and dependencies
- MariaDB 10.5
- WordPress
### Local tools
- PuTTY: SSH access into the EC2 instance
- WinSCP: manual file transfer between the EC2 instance and local computer
## Database migration to Amazon RDS
The original WordPress database was hosted on MariaDB locally, on the same EC2 instance as the deployed web application.
For Task 3.2, I created an RDS for MariaDB instance and an initial `wordpress_db` database, configuring connectivity to the existing EC2 instance during RDS creation [1].
Following [2] and [3], the existing WordPress database was exported from the EC2 instance using `mariadb-dump`:
```
mariadb-dump -u wp_user -p wordpress_db > wordpress_db.sql
```
Using `-p` without placing the password directly afterwards caused MariaDB to prompt for the password interactively, avoiding exposure of the password in the command and shell history.
The SQL dump was then imported to the earlier-created database hosted on RDS, following the example in [2]:
```
mariadb -u <user> \
    --port=3306 \
    --host=<RDS_endpoint> \
    -p \
    wordpress_db < wordpress_db.sql
```
The imported database was verified by connecting directly to RDS and checking the WordPress tables and test-post data created previously.
The WordPress live application was then reconfigured through `wp-config.php` using GNU nano, so that its database host and credentials pointed to the RDS MariaDB instance instead of the MariaDB server running locally on EC2.
To verify the migration, the local MariaDB service on the EC2 was stopped. The WordPress website continued to operate normally and retained its existing content, confirming that it was using the RDS database.
## S3 backup and restoration
The WordPress application files under  `/var/www/html` were first archived into a compressed `.tar.gz` file following [4] in the `/home/ec2-user` folder:
```
sudo tar -czf $HOME/wordpress_db_files.tar.gz -C /var/www html
```
I used WinSCP to make a copy of the archive on my local drive, then created a private S3 general purpose bucket and uploaded said archive as a manual backup of the application.
The backup object was then downloaded from S3 and intentionally renamed afterwards to distinguish it from the original archive.
After transferring the downloaded copy back to the EC2 instance, I created a separate restoration directory and extracted the archive using GNU tar [5]:
```
tar -xzf $HOME/froms3_wordpress_db_files.tar.gz -C $HOME/wordpress-restore
```
The restored folder contained the expected WordPress files and directories, including `wp-content`, `wp-includes` and `wp-load.php`.
This demonstrated that the application files could be retrieved from the backup in S3 and restored to the EC2 instance without overwriting the live deployment.
## References
[1] “Connecting an EC2 Instance and an RDS Database Automatically - Amazon Relational Database Service,” docs.aws.amazon.com. Accessed: Sep. 5, 2026. [Online]. Available: <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/ec2-rds-connect.html>

[2] “Importing Data From an External MariaDB Database to an Amazon RDS for MariaDB DB Instance - Amazon Relational Database Service,” docs.aws.amazon.com. Accessed: Sep. 7, 2026. [Online]. Available: <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mariadb-importing-data-external-database.html>

[3] “Mariadb-Dump | Server | MariaDB Documentation,” Mariadb.com. Accessed: Sep. 9, 2026. [Online]. Available: <https://mariadb.com/docs/server/clients-and-utilities/backup-restore-and-import-clients/mariadb-dump>

[4] “GNU Tar 1.35: 2.6 How to Create Archives,” Gnu.org. Accessed: Sep. 9, 2026. [Online]. Available: <https://www.gnu.org/software/tar/manual/html_section/create.html>

[5] “GNU Tar 1.35: 2.8 How to Extract Members from an Archive,” Gnu.org. Accessed: Sep. 9, 2026. [Online]. Available: <https://www.gnu.org/software/tar/manual/html_section/extract.html>
