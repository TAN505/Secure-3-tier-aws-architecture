# Security Design

## Network boundaries

The ALB is the internet-facing entry point. Application instances have private addresses, and RDS public access is disabled. Public, application, and database subnets use separate route tables.

| Security group | Inbound access |
|---|---|
| ALB | HTTP TCP 80 from the internet |
| Application | TCP 5000 from the ALB security group |
| Database | TCP 3306 from the application security group |

Security group references identify the allowed layer. Application instances use outbound NAT connectivity for package installation and AWS service access. The initial application security group permits broad outbound traffic; further egress restriction is a future improvement. NACLs were left at their default configuration.

## Administration and IAM

Session Manager is used for EC2 administration. The instance role includes `AmazonSSMManagedInstanceCore` and an inline permission granting `secretsmanager:GetSecretValue` on the application database secret. No inbound SSH rule or SSH key pair is required for this access method.

An IAM instance role supplies temporary AWS credentials to the SDK. No static AWS access keys are stored in application source.

## Database permissions and secrets

The application uses the dedicated `labapp` database account with `SELECT` and `INSERT` on `labdb.messages`. The account is separate from the administrative user.

The account host is `%`; network access is constrained by the RDS security group. The secret stores the database hostname, port, database name, username, and password.

Credentials are loaded once at each worker's startup. Updating the secret requires restarting `labapp` on both instances. Automatic secret rotation and refresh are not implemented.

During setup, a temporary weak password was used and disclosed in the troubleshooting conversation. Before retaining or rebuilding the environment, replace that password with a strong unique value, update the secret, and restart the application on both instances. No password is included in this repository.

## Application and transport

Database connections verify the RDS server certificate using the AWS RDS CA bundle. Inserts use parameterized SQL. HTML output uses Flask/Jinja template escaping. Input length and request size are limited.

Browser-to-ALB and ALB-to-EC2 connections currently use HTTP. HTTPS is a documented improvement. The message board is a learning application without user authentication, CSRF protection, or rate limiting.

## Health checks and availability

`/health` returns application status but does not query RDS. A healthy ALB target therefore does not prove database availability. RDS remains Single-AZ and is a database-layer single point of failure.

The two EC2 instances were tested for continued service after one was stopped. There is no Auto Scaling Group or automatic instance replacement.
