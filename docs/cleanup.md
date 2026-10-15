# Cleanup Guide

Save application source, service configuration, dependency versions, and test evidence before deleting resources. This procedure ends the website and can permanently remove the database data.

1. Delete the lab ALB, then its target group. Disable deletion protection if enabled.
2. Terminate both lab EC2 instances. Check for leftover EBS volumes and snapshots; root volumes should be deleted if `Delete on termination` was enabled.
3. Delete the RDS instance. Retain a final snapshot only if the data is needed; snapshots and retained automated backups can continue to incur storage charges. Check and remove unwanted backups after deletion.
4. Delete the RDS subnet group after the database has finished deleting.
5. Schedule deletion of the application secret. Check for a separate RDS-managed master secret if one was created.
6. Delete the Regional NAT Gateway and wait for deletion to complete. Check for leftover unassociated lab Elastic IP allocations and release only those no longer needed. Allow AWS to manage its service-owned NAT route table and interfaces during deletion.
7. Delete all six lab subnets and the custom public/application/database route tables after resource dependencies disappear.
8. Remove cross-referencing lab security group rules, then delete the lab security groups.
9. Detach and delete the Internet Gateway, then delete the VPC. Its default resources are removed with it.
10. Delete the lab CloudWatch alarm and dashboard. Remove lab log groups if any were created.
11. Delete the lab-only IAM role and inline policy once the instances are terminated. Do not remove shared roles or resources.
12. Check for unexpected remaining lab resources in Singapore, including network interfaces, EBS storage, RDS snapshots, and IP allocations.

No domain, ACM certificate, or SNS topic was created for the completed monitoring-only version. If those were added separately, clean up only the resources dedicated to the lab.

Stopping EC2 does not remove ALB, NAT, database storage, or all other charges. Review Bills and Cost Explorer after cleanup; billing data can be delayed and charges already incurred remain visible.

Cleanup status is not asserted by this repository. Verify deletion in the AWS account after executing these steps.
