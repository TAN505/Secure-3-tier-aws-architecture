# Test Results

Testing date: 14 October 2026. Screenshot timestamps shown by CloudWatch are UTC; local time in Malaysia is UTC+8.

## Functional validation

The browser accessed the message board through the ALB DNS address. The page identified EC2-A as the serving instance and displayed messages from RDS, including a message submitted through the ALB.

Evidence: [ALB message board](../screenshots/01-alb-message-board.png).

Both application instances returned messages from the same database through `/messages`. EC2-B's response contained messages previously inserted from EC2-A as well as messages inserted from EC2-B.

## Application instance failover

EC2-A was stopped while EC2-B remained registered on port 5000. An initial request returned **504 Gateway Time-out**. The available evidence does not establish the precise cause or duration of that interruption.

After confirming EC2-A was `Unused` and EC2-B was `Healthy`, subsequent browser requests were served by EC2-B. Two messages, `Testing through the ALB Again` and `Testing through the ALB Twice`, were saved and appeared in EC2-B's database query results.

- [Initial 504 interruption](../screenshots/02-ec2-a-stopped-504.png)
- [Successful EC2-B serving and writes](../screenshots/03-ec2-b-failover-success.png)

Result: application service recovered through the remaining instance. This was a controlled instance-stop test, not an actual Availability Zone outage. No zero-downtime claim is made.

## Alarm detection

Both instances were running. The `labapp` service was stopped on EC2-A using:

```bash
sudo systemctl stop labapp
```

The target became unhealthy while EC2-B stayed healthy. The original alarm used `> 1` with longer periods, so one unhealthy target did not breach the threshold. The alarm was corrected to:

| Setting | Value |
|---|---|
| Namespace | `AWS/ApplicationELB` |
| Metric | `UnHealthyHostCount` |
| Dimensions | The lab ALB and target group |
| Statistic | Maximum |
| Period | 1 minute |
| Condition | Greater than or equal to 1 |
| Evaluation | 2 of 2 datapoints |
| Missing data | Not breaching |
| Notification actions | None |

The alarm transitioned to **In alarm**, with a recorded state update at **06:02 UTC / 14:02 Malaysia time**.

- [Alarm list](../screenshots/04-alarm-triggered-list.png)
- [Alarm graph and threshold](../screenshots/05-alarm-triggered-graph.png)

## Recovery

The application was restored on EC2-A:

```bash
sudo systemctl start labapp
```

Both targets returned to Healthy. The alarm returned to **OK**, with a recorded state update at **06:08 UTC / 14:08 Malaysia time**.

- [Recovered alarm](../screenshots/06-alarm-recovered-ok.png)
- [Both targets Healthy](../screenshots/07-both-targets-healthy.png)

Result: failure detection and recovery were verified. Email delivery and automatic remediation were not configured or tested.

## Monitoring evidence

[Dashboard](../screenshots/08-cloudwatch-dashboard.png): target health, ALB requests and response time, both EC2 CPU metrics, and RDS CPU/connections.

Database connection counts can return to zero between requests because this application closes each database connection after use.

## Tests outside the completed scope

- RDS Multi-AZ failover: not performed because the standby option was disabled; the exact restriction was not independently established.
- HTTPS: not configured; browser access uses HTTP.
- Automatic replacement of a failed instance: not configured.
- Load testing, an actual AZ outage, database restoration, and automated credential rotation: not tested.
