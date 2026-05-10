# AWS VPC Peering — Cross-Region Private Connectivity

## Architecture Overview

```
Your Laptop
    │  SSH (public IP)
    ▼
Internet Gateway (IGW)
    │
    ▼  VPC-A  │  Mumbai (ap-south-1)  │  10.100.0.0/16
┌─────────────────────────────────────┐
│  Public Subnet 10.100.0.0/24        │
│  ┌────────────────────────────────┐ │
│  │  EC2-A  (Bastion host)         │ │
│  │  Public IP + Private IP        │ │
│  └────────────────────────────────┘ │
│            │ SSH                    │
│  Private Subnet 10.100.11.0/24      │
│  ┌────────────────────────────────┐ │
│  │  EC2-B                         │ │
│  │  Private IP only               │ │
│  └────────────────────────────────┘ │
└──────────────────┬──────────────────┘
                   │  VPC Peering (pcx-xxxxx)
┌──────────────────▼──────────────────┐
│  VPC-B  │  N.Virginia (us-east-1)  │  10.200.0.0/16
│  Private Subnet 10.200.11.0/24      │
│  ┌────────────────────────────────┐ │
│  │  EC2-C                         │ │
│  │  Private IP only               │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**Goal:** From your laptop, reach EC2-C (private subnet, N.Virginia) by hopping through EC2-A → EC2-B → VPC Peering → EC2-C.

---

## Step 1 — Create VPCs

### VPC-A (Mumbai)
1. Go to **AWS Console → VPC → Your VPCs → Create VPC**
2. Region: `ap-south-1` (Mumbai)
3. Name: `VPC-A`
4. IPv4 CIDR: `10.100.0.0/16`
5. Create Internet Gateway → Name: `VPC-A-IGW`
6. Attach IGW to VPC-A (Actions → Attach to VPC)

### VPC-B (N.Virginia)
1. Switch region to `us-east-1` (N.Virginia)
2. Go to **VPC → Create VPC**
3. Name: `VPC-B`
4. IPv4 CIDR: `10.200.0.0/16`
5. **No internet gateway needed** for VPC-B

> **Why non-overlapping CIDRs?** VPC peering requires that both VPCs have completely non-overlapping IP ranges. 10.100.x.x and 10.200.x.x never overlap.

---

## Step 2 — Create Subnets & Route Tables

### VPC-A (Mumbai) — 2 subnets

**Public Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-A-Public-Subnet` |
| AZ | `ap-south-1a` |
| CIDR | `10.100.0.0/24` |

Create route table `VPC-A-Public-RT`:
- Route: `0.0.0.0/0` → `IGW` (the internet gateway)
- Associate with `VPC-A-Public-Subnet`

**Private Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-A-Private-Subnet` |
| AZ | `ap-south-1a` |
| CIDR | `10.100.11.0/24` |

Create route table `VPC-A-Private-RT`:
- Default route only: `10.100.0.0/16` → `local`
- Associate with `VPC-A-Private-Subnet`

### VPC-B (N.Virginia) — 1 subnet

**Private Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-B-Private-Subnet` |
| AZ | `us-east-1a` |
| CIDR | `10.200.11.0/24` |

Create route table `VPC-B-Private-RT`:
- Default route only: `10.200.0.0/16` → `local`
- Associate with `VPC-B-Private-Subnet`

---

## Step 3 — Launch EC2 Instances

### EC2-A (VPC-A, Public Subnet — Bastion Host)

1. Launch in **Mumbai region**, `VPC-A-Public-Subnet`
2. **Enable** Auto-assign Public IP
3. Security Group: `EC2-A-SG`
   - Inbound: SSH (22) from `MyIP` or `0.0.0.0/0`
   - Outbound: All traffic
4. Key pair: `my-key.pem` (same key will be used for all 3 instances)

### EC2-B (VPC-A, Private Subnet)

1. Launch in **Mumbai region**, `VPC-A-Private-Subnet`
2. **Disable** Auto-assign Public IP
3. Security Group: `EC2-B-SG`
   - Inbound: SSH (22) from `10.100.0.0/16` (VPC-A CIDR)
   - Outbound: All traffic

### EC2-C (VPC-B, Private Subnet)

1. Launch in **N.Virginia region**, `VPC-B-Private-Subnet`
2. **Disable** Auto-assign Public IP
3. Security Group: `EC2-C-SG`
   - Inbound: SSH (22) from `10.100.0.0/16` (VPC-A CIDR)
   - Inbound: ICMP - All (ping) from `10.100.0.0/16` (VPC-A CIDR)
   - Outbound: All traffic

> **Why allow from VPC-A CIDR on EC2-C?** After peering, traffic from EC2-B will arrive at EC2-C with the source IP from the 10.100.x.x range. EC2-C's security group must allow it.

---

## Step 4 — Test Connectivity (Should Fail)

### Copy your SSH key to EC2-A
```bash
# Option A: Copy key to EC2-A
scp -i my-key.pem my-key.pem ec2-user@<EC2-A-Public-IP>:~/.ssh/

# Option B: Use SSH agent forwarding (more secure)
ssh-add my-key.pem
ssh -A -i my-key.pem ec2-user@<EC2-A-Public-IP>
```

### SSH chain: Laptop → EC2-A → EC2-B → try ping EC2-C
```bash
# From your laptop, SSH to EC2-A
ssh -i my-key.pem ec2-user@<EC2-A-Public-IP>

# From EC2-A, SSH to EC2-B (using private IP)
ssh -i ~/.ssh/my-key.pem ec2-user@10.100.11.<x>

# From EC2-B, try to ping EC2-C
ping 10.200.11.<x>
# Expected result: ping hangs / no response — NO ROUTE TO HOST
```

**Why does this fail?**
- EC2-B is in VPC-A's private subnet
- EC2-C is in VPC-B's private subnet
- The two VPCs have no connectivity — they are completely isolated by default
- No route exists in `VPC-A-Private-RT` for the `10.200.0.0/16` range

---

## Step 5 — Create VPC Peering Connection

### 5a. Copy VPC-B's ID
1. Go to **N.Virginia → VPC → Your VPCs**
2. Find VPC-B and copy its VPC ID (e.g. `vpc-0abc123def456`)

### 5b. Create the peering request (from Mumbai)
1. Go to **Mumbai → VPC → Peering Connections → Create peering connection**
2. Fill in the form:

| Field | Value |
|-------|-------|
| Name | `VPC-A-VPC-B-Peering` |
| Local VPC to peer with | Select `VPC-A` |
| Account | My account |
| Region | Another Region |
| Select region | `us-east-1` (N.Virginia) |
| VPC ID (Accepter) | Paste `vpc-0abc123def456` |

3. Click **Create Peering Connection**
4. Note the peering connection ID: `pcx-xxxxx`

### 5c. Accept the peering request (from N.Virginia)
1. Switch to **N.Virginia region**
2. Go to **VPC → Peering Connections**
3. Find the pending peering request
4. **Actions → Accept Request → Accept**

### 5d. Verify peering status
- Status should now show **Active**

> **Important:** VPC peering alone is NOT enough. You still need to update route tables — the two VPCs know about the connection but don't yet know how to route traffic through it.

---

## Step 6 — Update Route Tables

### VPC-A-Private-RT (Mumbai)
1. Go to **Mumbai → VPC → Route Tables**
2. Select `VPC-A-Private-RT` → Routes → Edit routes → Add route:

| Destination | Target |
|-------------|--------|
| `10.200.0.0/16` | `pcx-xxxxx` (select Peering Connection) |

3. Save changes

### VPC-B-Private-RT (N.Virginia)
1. Go to **N.Virginia → VPC → Route Tables**
2. Select `VPC-B-Private-RT` → Routes → Edit routes → Add route:

| Destination | Target |
|-------------|--------|
| `10.100.0.0/16` | `pcx-xxxxx` (select Peering Connection) |

3. Save changes

> **Why both route tables?** Peering is non-transitive and bidirectional — each VPC must explicitly know how to reach the other. Without both routes, traffic can leave one VPC but the response has no return path.

---

## Step 7 — Verify Full Connectivity

```bash
# From EC2-B terminal, ping EC2-C
ping 10.200.11.<x>
# Expected: PING working! You see replies

# Optional: SSH to EC2-C directly from EC2-B
ssh -i ~/.ssh/my-key.pem ec2-user@10.200.11.<x>
# Expected: Logged in to EC2-C
```

### Complete traffic path (end-to-end)

```
Your Laptop
  │  SSH (port 22, public internet)
  ▼
Internet Gateway (VPC-A)
  │
  ▼
EC2-A  10.100.0.x  (public subnet)
  │  SSH (private network, VPC-A)
  ▼
EC2-B  10.100.11.x  (private subnet, VPC-A)
  │  ICMP/SSH via VPC-A-Private-RT → pcx-xxxxx
  ▼
VPC Peering Connection
  │  via VPC-B-Private-RT → local
  ▼
EC2-C  10.200.11.x  (private subnet, VPC-B)
```

---

## Security Group Summary

| Instance | SG Name | Rule | Source | Reason |
|----------|---------|------|--------|--------|
| EC2-A | EC2-A-SG | SSH (22) inbound | `0.0.0.0/0` or My IP | Accessible from internet |
| EC2-B | EC2-B-SG | SSH (22) inbound | `10.100.0.0/16` | Only from VPC-A |
| EC2-C | EC2-C-SG | SSH (22) inbound | `10.100.0.0/16` | Only from VPC-A (via peering) |
| EC2-C | EC2-C-SG | ICMP All inbound | `10.100.0.0/16` | Allow ping from VPC-A |

---

## Route Table Summary

### VPC-A-Public-RT
| Destination | Target | Note |
|-------------|--------|------|
| `10.100.0.0/16` | local | Within VPC-A |
| `0.0.0.0/0` | `igw-xxxxx` | Internet access |

### VPC-A-Private-RT (after peering)
| Destination | Target | Note |
|-------------|--------|------|
| `10.100.0.0/16` | local | Within VPC-A |
| `10.200.0.0/16` | `pcx-xxxxx` | Route to VPC-B via peering |

### VPC-B-Private-RT (after peering)
| Destination | Target | Note |
|-------------|--------|------|
| `10.200.0.0/16` | local | Within VPC-B |
| `10.100.0.0/16` | `pcx-xxxxx` | Route to VPC-A via peering |

---

## Common Pitfalls & Troubleshooting

**Ping still not working after peering?**
- Check security group on EC2-C allows ICMP from `10.100.0.0/16`
- Confirm BOTH route tables are updated (not just one)
- Verify the peering connection status is **Active** (not pending)
- Make sure you're using EC2-C's private IP, not a public IP

**SSH to EC2-B fails from EC2-A?**
- Ensure your `.pem` key is on EC2-A or use SSH agent forwarding (`-A` flag)
- Confirm EC2-B's security group allows SSH from `10.100.0.0/16` (not just a specific IP)

**VPC peering request not visible in N.Virginia?**
- Make sure you switched to the correct region in the AWS console
- Check under VPC → Peering Connections — filter by "Pending Acceptance"

**Cannot create peering — CIDR overlap error?**
- VPC-A (10.100.0.0/16) and VPC-B (10.200.0.0/16) must not overlap
- If you accidentally used the same CIDR (e.g., 10.0.0.0/16 for both), you must delete and recreate one VPC with a different CIDR

---

## Clean-Up

### If continuing to the next exercise
1. Terminate EC2-C in VPC-B
2. Delete VPC peering connection
3. Delete VPC-B and its subnet/route table

### If stopping completely
1. Terminate all three EC2 instances (EC2-A, EC2-B, EC2-C)
2. Delete VPC peering connection
3. Delete VPC-B (N.Virginia)
4. Delete VPC-A and its subnets, route tables, and IGW (Mumbai)

> **Cost note:** Stopped EC2 instances still incur EBS storage charges. VPC peering has a per-GB data transfer cost for cross-region traffic. VPCs and subnets themselves have no ongoing cost.

---

## Key Concepts Recap

| Concept | What it means |
|---------|---------------|
| **VPC Peering** | Private network link between two VPCs (same or different account/region) |
| **Non-transitive** | Peering A↔B and B↔C does NOT give A↔C connectivity |
| **Route table required** | Peering alone does nothing — you must add routes in both VPCs |
| **Security groups** | Must allow traffic from the peer VPC's CIDR, not just the local CIDR |
| **No overlapping CIDRs** | Peered VPCs must have entirely different IP ranges |
| **Bastion / Jump host** | EC2-A acts as a gateway into the private network from the internet |
