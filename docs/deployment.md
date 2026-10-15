# Manual Deployment Guide

This guide records the deployment sequence used for the lab. It assumes access to an AWS account and the Singapore Region. Resource sizes and account restrictions should be reviewed before provisioning.

## 1. Network

Create a VPC with CIDR `10.0.0.0/16`. After creation, use **Your VPCs → Actions → Edit DNS settings** to enable DNS resolution and DNS hostnames.

Create the six subnets listed in the README across two AZs. Attach one Internet Gateway to the VPC. Configure a public route table with `0.0.0.0/0 → Internet Gateway` and associate the two public subnets.

Create a **Public Regional NAT Gateway** in the VPC with automatic IP allocation. Associate both private application subnets with a route table containing `0.0.0.0/0 → Regional NAT Gateway`. Associate the two database subnets with a separate route table containing only the VPC local route.

## 2. Security groups and EC2 role

Create the ALB, application, and database security groups with the rules documented in `security-design.md`. Attach `AmazonSSMManagedInstanceCore` to an EC2 service role.

## 3. RDS

Create a DB subnet group using both private database subnets. Create an RDS MySQL DB instance in that subnet group with public access disabled and the database security group attached. Use a small supported instance class and review storage, backup, and monitoring settings.

Use `lab1-mysql` as the instance identifier and `labdb` as the initial internal database name. The implemented database is Single-AZ. Record the database endpoint privately for application configuration.

## 4. EC2 and connectivity

Launch one Amazon Linux 2023 instance in each private application subnet. Disable public IP assignment, attach the application security group and EC2 role, and use Session Manager for administration.

On both instances:

```bash
sudo dnf install python3 python3-pip mariadb105 -y
```

Test the database connection from both instances. Replace `YOUR_RDS_ENDPOINT` with the endpoint; enter the password only at the prompt:

```bash
mysql -h YOUR_RDS_ENDPOINT -P 3306 -u labadmin -p
```

## 5. Database schema and application account

Run the following once as the database administrator, replacing the password placeholder before execution:

```sql
USE labdb;
CREATE TABLE IF NOT EXISTS messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE USER 'labapp'@'%' IDENTIFIED BY 'CHOOSE_A_STRONG_UNIQUE_PASSWORD';
GRANT SELECT, INSERT ON labdb.messages TO 'labapp'@'%';
```

If the application account already exists, use `ALTER USER` instead of `CREATE USER`. Keep the password out of repository files and screenshots.

## 6. Secrets Manager

Create a secret named `lab1/app/database` in Singapore with these keys:

| Key | Meaning |
|---|---|
| `host` | RDS endpoint |
| `port` | `3306` |
| `dbname` | `labdb` |
| `username` | `labapp` |
| `password` | Application account password |

Use the default Secrets Manager KMS key. Grant the EC2 role `secretsmanager:GetSecretValue` on this secret's full ARN. The administrator and application passwords are separate credentials.

## 7. Application deployment

The deployed Flask application resides at `/opt/labapp/app.py` on both instances. Gunicorn runs from a virtual environment, with a service definition at `/etc/systemd/system/labapp.service`.

The application dependencies are Flask, Gunicorn, PyMySQL, and Boto3. Database TLS validation uses `/opt/labapp/global-bundle.pem`, downloaded from:

```text
https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

The service runs as a dedicated `labapp` OS user, binds to `0.0.0.0:5000`, uses two workers, and has `Restart=on-failure`. Set `Environment=SERVER_NAME=EC2-A` on the first instance and `EC2-B` on the second.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now labapp
sudo systemctl status labapp --no-pager
curl http://localhost:5000/health
curl http://localhost:5000/messages
```

### Export the exact deployed source before cleanup

These commands display files for copying to a local `app/` folder; they do not transfer files automatically:

```bash
cat /opt/labapp/app.py
sudo cat /etc/systemd/system/labapp.service
/opt/labapp/venv/bin/pip freeze
```

Save the outputs as `app/app.py`, `app/labapp.service`, and `app/requirements.txt`. Review them for secrets before uploading. The dependency versions should come from the instance; this documentation does not invent or claim a recorded version lock.

## 8. ALB

Create an instance target group on HTTP port 5000 with `/health` as the health-check path and HTTP 200 as the success code. Register both application instances.

Create an internet-facing ALB in the two public subnets. Attach the ALB security group and configure HTTP port 80 to forward to the target group. Confirm both targets are Healthy and open the ALB DNS address in a browser.

## 9. Monitoring and validation

Create one dashboard using the basic metrics shown in the README. For the unhealthy-target alarm, use Maximum, a 1-minute period, `>= 1`, and 2 of 2 datapoints. No SNS notification action is required for the console-based test.

Follow `test-results.md` to reproduce the controlled failure and recovery tests. Review AWS pricing and account-wide allowances before enabling additional features.

## Troubleshooting

| Symptom | Check |
|---|---|
| Session Manager unavailable | Instance role, SSM Agent, NAT/route connectivity, outbound HTTPS |
| MySQL error 1045 | Application user existence and matching password in Secrets Manager |
| App returns 500 while `/health` works | `sudo journalctl -u labapp -n 80 --no-pager` |
| Secret edited but old credentials used | Restart `labapp` on both instances |
| ALB target unhealthy | App running on 5000, SG source, target registration, health path |
| Alarm remains OK with one unhealthy target | Check `>= 1`, Maximum, period, evaluation count, and resource dimensions |
