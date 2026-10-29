# AWS VPC Peering Project (EWB Cloud Labs, Project 01)

**Name:** Aman Kumar  |  **Batch:** YOUR BATCH  |  **Region:** ap-south-1 (Mumbai)  |  **Date:** 22-10-2026

## Objective
Connect two VPCs with VPC peering so that App-A (VPC A) and App-B (VPC B) can talk over **private IPv4 addresses only**. Both app servers have no public IP. One bastion host with a public IP in VPC A is the only entry point, and SSH to the private servers uses ProxyJump through it.

## Architecture
```
Laptop --SSH 22--> Internet Gateway (ewb-igw-a) --> Bastion (public subnet, VPC A)
                                                      |-- ProxyJump --> App-A (private subnet, VPC A)
                                                      '-- ProxyJump --> App-B (private subnet, VPC B)
App-A <------------- VPC peering (pcx) -------------> App-B      (private IPs only, VPC B has no IGW)
```

## IP plan and inventory
| Resource | ID | CIDR / address |
|---|---|---|
| ewb-vpc-a | vpc-0c5f736c9fc30ad6f | 10.10.0.0/16 |
| ewb-a-public | subnet-0fdc794f65aa19cba | 10.10.1.0/24 |
| ewb-a-private | subnet-02bf637ddb0e27d36 | 10.10.2.0/24 |
| ewb-vpc-b | vpc-0179eca8d4644d178 | 10.20.0.0/16 |
| ewb-b-private | subnet-0b32589cab30db11b | 10.20.1.0/24 |
| ewb-igw-a | igw-078d383a8a0d774d0 | attached to ewb-vpc-a |
| ewb-bastion | i-0900dc99b7ca8db08 | 10.10.1.10 + public IPv4 |
| ewb-app-a | i-00a4b627084dbaa6e | 10.10.2.10 (no public IP) |
| ewb-app-b | i-022316462bf65ab90 | 10.20.1.10 (no public IP) |
| ewb-a-b-peer | pcx-016ad3738d94d964e | Active |

## Route tables
| Table | Destination | Target |
|---|---|---|
| ewb-rt-a-public (rtb-086386f931fae0357) | 10.10.0.0/16, 0.0.0.0/0, 10.20.0.0/16 | local, igw, pcx |
| ewb-rt-a-private (rtb-044aead01aaaa28ff) | 10.10.0.0/16, 10.20.0.0/16 | local, pcx |
| ewb-rt-b-private (rtb-01e22e430e3341e37) | 10.20.0.0/16, 10.10.0.0/16 | local, pcx |

## Security group inbound rules
| Security group | Rule | Source |
|---|---|---|
| ewb-sg-bastion | SSH 22 | my public IPv4 /32 |
| ewb-sg-app-a | SSH 22, ICMP Echo request | 10.10.1.10/32, 10.20.1.10/32 |
| ewb-sg-app-b | SSH 22, ICMP Echo request | 10.10.1.10/32, 10.10.2.10/32 |

## Forward path and return path
**Forward:** App-A (10.10.2.10) uses the route `10.20.0.0/16 -> pcx` in ewb-rt-a-private, crosses the peering connection and reaches App-B, where ewb-sg-app-b allows the Echo request.
**Return:** App-B replies using `10.10.0.0/16 -> pcx` in ewb-rt-b-private. Without this return route the reply is lost.

## Test results
| Test | Result | Evidence |
|---|---|---|
| Initial failed ping (App-A to App-B) | 100% loss, cause: subnet not associated with ewb-rt-a-private | screenshots/07d-app-a-ping-app-b-failed-before-fix.jpg |
| Fix: explicit subnet association | saved | screenshots/05d-a-private-association-fixed.jpg |
| No public IP on App-A and App-B | confirmed | screenshots/02a-app-a-no-public-ip.jpg, screenshots/02b-app-b-no-public-ip.jpg |

## What I learned
- Peering is only the link. Routes on both sides and security group rules decide if traffic flows.
- A subnet without an explicit association uses the main route table, so check subnet associations first when ping fails.
- With ProxyJump the target server sees the bastion's private IP as the SSH source.
- Peering is non-transitive and does not give VPC B internet access.

## Cleanup
All lab resources were deleted after the evidence was captured. See `screenshots/12a` to `12e` (instances terminated, subnets deleted, route tables, peering deleted, only default VPC left).

## Files
- `AmanKumar_EWB_VPC_Peering.pdf` - full report with labelled screenshots
- `screenshots/` - evidence
- `ssh-config-example.txt` - SSH config with the real bastion IP replaced by a placeholder

> The private key (`ewb-lab.pem`) is intentionally not included in this repository.
