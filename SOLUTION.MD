# Configure Multi-Tier Security Groups Lab - Solution

**Student Name:** Balint Lojt
**Date Completed:** 15/07/2026

---
**Note:** 
AWS doesn't allow security group names to start with "sg-" (that
prefix is reserved for auto-generated security group IDs). Renamed all four groups with the suffix pattern instead: bastion-sg, web-tier-sg, app-tier-sg, database-sg.

**Note:** 
The lab's original design has app and database tiers fully private
(no public IP), but this requires a NAT Gateway for outbound internet access
(needed to install packages like netcat/Node.js), which isn't covered until
next week's lab. Per instructor guidance, app and database instances were
given temporary public IPs for this lab only, to allow package installation.
Inbound access remains fully restricted by security groups regardless -
only bastion-sg can SSH to these instances, so this doesn't compromise the
actual access control being demonstrated.

**Note:**
During test phase(README.md step 4), when I had to document with screenshots,
I put all the separate terminals into 1 png file. 
On the top of each terminal session I echoed which test phase I'm doing 
at that very moment (from test 1~7). 
At my current learning stage this felt the easiest to see through for me and
for anyone who will check my work. 




# Environment Details

| Tier | Instance ID | Private IP | Public IP | Security Group |
|------|-------------|------------|-----------|----------------|
| Bastion | [i-xxxxxxxxxxxxx] | [10.x.x.x] | [x.x.x.x — redact if repo is public] | [sg-bastion] |
| Web | [i-xxxxxxxxxxxxx] | [10.x.x.x] | [x.x.x.x] | [sg-web-tier] |
| App | [i-xxxxxxxxxxxxx] | [10.x.x.x] | [none] | [sg-app-tier] |
| Database | [i-xxxxxxxxxxxxx] | [10.x.x.x] | [none] | [sg-database] |

- **Region / VPC:** [eu-west-1 / vpc-xxxxxxxx]
- **Key Pair:** [bootcamp-week2-key.pem]
- **My IP (SSH source):** [x.x.x.x/32]

---

# Step 1: Create Security Groups

## Screenshot 1 – Security Groups List

Save your screenshot as:

```
screenshots/01-security-groups-list.png
```

![Security Groups List](screenshots/01-security-groups-list.png)

---

- [X] `sg-bastion` created
- [x] `sg-web-tier` created
- [x] `sg-app-tier` created
- [x] `sg-database` created
- [x] Rules reference other **security groups**, not IP ranges (except port 22 on the bastion and 80/443 on the web tier)

### bastion-sg

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [22] | [143.179.136.227/32] | [SSH from my laptop] |
| Outbound | [22] | [web-tier-sg, app-tier-sg, database-sg] | [Hop to private tiers] |

### web-tier-sg

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [80] | [0.0.0.0/0] | [Public HTTP] |
| Inbound | [443] | [0.0.0.0/0] | [Public HTTPS] |
| Inbound | [22] | [bastion-sg] | [SSH via bastion only] |
| Outbound | [8080] | [app-tier-sg] | [Proxy to app tier] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

### app-tier-sg

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [8080] | [web-tier-sg] | [App traffic from web tier] |
| Inbound | [22] | [bastion-sg] | [SSH via bastion only] |
| Outbound | [3306] | [database-sg] | [MySQL queries] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

### database-sg

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [3306] | [app-tier-sg] | [MySQL from app tier only] |
| Inbound | [22] | [bastion-sg] | [SSH via bastion only] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

---

### Why reference a security group instead of a private IP or CIDR?

In AWS, servers reboot, scale up, and scale down constantly. Every time they do, they get brand new IP addresses. If I lock the database down to a specific IP, I’ll be stuck manually updating firewall rules every single time a server restarts or more added. 
If I refer it to the correct security group ID any time a scaling or restart happens it automatically connects to it no matter the IP.





# Step 2: Launch Instances and Connect Through the Bastion

## Screenshot 2 – Web Tier Security Group Rules

Save your screenshot as:

```
screenshots/02-web-tier-rules.png
```

![Web Tier Rules](screenshots/02-web-tier-rules.png)

---

- [X] 4 instances launched, one security group each
- [X] Connected to the bastion from my laptop
- [X] Reached web / app / db by **private IP** through the bastion

---

### Which connection method did you use?

- [X] ProxyJump
- [ ] Copy key to bastion

---

### My `~/.ssh/config` (or the commands I ran)

Host bastion
    HostName XX.XX.XX.XX (redacted)
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem

Host web
    HostName 172.31.25.29
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion

Host app
    HostName 172.31.17.142
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion

Host db
    HostName 172.31.30.68
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion



### Why must you target the private tiers by private IP instead of public IP?

Shrinking the attack surface of the instances. Any public IP is an extra target that anyone can try to break into.
Brutefocring in or overwhelm with DDoS. The defence in depth model builds on this that only what is a must to expose 
to the wide internet should be exposed. Example: load balancer or Bastion. Also setting up 1 way traffic only. 
If a server needs an update it should establish a one way channel through NAT gateway. It would allow to reach out
but does not establish the channel backwards. 




---
# Step 3: Simulate Application Traffic

## Screenshot 3 – Services Running

Save your screenshot as:

```
screenshots/03-services-running.png
```

![Services Running](screenshots/03-services-running.png)

---

- [X] Database: `nc -l 3306` loop running (`nmap-ncat` installed)
- [x] App: `server.js` listening on 8080 (`nodejs` installed)
- [x] Web: Nginx installed, proxying `/` to `http://172.31.17.142:8080`, enabled on boot

### Anything that did not start cleanly?

All worked as intended. 

# Step 4: Test Traffic Flow

## Screenshot 4 – Traffic Flow Test

Save your screenshot as:

```
screenshots/04-traffic-flow-test.png
```

![Traffic Flow Test](screenshots/04-traffic-flow-test.png)

---

**Reading results:** *connection refused* = the security group **allowed** the packet but nothing was listening. *Timeout / hang* = the security group **blocked** the packet.

Tick the box if the test did what the **Expected** column says.

| # | Test | Run from | Expected | Got it |
|---|------|----------|----------|--------|
| 1 | `curl 34.235.128.23 | Laptop | ✅ HTTP 200 | [worked as intended ] |
| 2 | `ssh bastion`, then `ssh WEB_PRIVATE_IP` | Laptop → bastion | ✅ Connects | [ worked as intended] |
| 3 | `ssh ec2-user@WEB_PUBLIC_IP` | Laptop | ❌ Timeout | [ worked as intended, got no response form terminal] |
| 4 | `nc -zv APP_PRIVATE_IP 8080` | Web | ✅ Succeeds | [worked as intended ] |
| 5 | `nc -zv DB_PRIVATE_IP 3306` | App | ✅ Succeeds | [worked as intended ] |
| 6 | `nc -zv DB_PRIVATE_IP 3306` | Web | ❌ Timeout | [worked as intended ] |
| 7 | `curl http://WEB_PUBLIC_IP` | Laptop | ✅ "Hello from App Tier" | [worked as intended] |

---

### Test 6 — How did it fail?

(Timeout is the correct answer. "Connection refused" means the web tier can reach the database and a rule is too permissive.)

- [X] Timed out / hung ✅ Correct
- [ ] Connection refused ⚠️ Rules too permissive

---

### Did every test pass on the first attempt?

- [X] Yes
- [ ] No

If no, what did you have to fix?

N/A

# Step 5: Architecture

## Screenshot 5 – Completed Architecture

Save your screenshot as:

```
screenshots/05-architecture-console.png
```

![Architecture Console](screenshots/05-architecture-console.png)

---

Confirm each communication path.

- [X] Internet → Web on port **80**
- [X] Web → App on port **8080**
- [X] App → Database on port **3306**
- [X] SSH access only through the bastion host
- [X] Web → Database blocked
- [X] Internet → App blocked
- [X] Internet → Database blocked

---

### Instances with no public IP
Database
App 

### Instances reachable from the Internet

Bastion
Web


# Bonus Challenges (Optional)

- [ ] **Challenge 1:** Remove the `443 → 0.0.0.0/0` egress rule from one tier. What broke?

```
_______________________________________________________________
```

- [ ] **Challenge 2:** Attach two security groups to one instance. How do the rules combine?

```
_______________________________________________________________
```

- [ ] **Challenge 3:** Tighten the bastion's inbound SSH to a single `/32` and re-test from another network.

```
_______________________________________________________________
```

- [ ] **Challenge 4:** Recreate the four security groups using the AWS CLI.

```
_______________________________________________________________
```

---

# Reflection

## 1. Why can the application tier reach the database, but the web tier cannot?

Because in the security groups I stated that webtier should not have permission to communicate with the database.
webtier is exposed to the internet and in case someone manages to get inside the webtier instance will have a direct route 
to the database.



## 2. Why do all private instances only accept SSH from the bastion?


Bastion works as the middle man. Bastion only accepts SSH connection specifically from my IP or whoever else is allowed.
Once Bastion confirmed that my SSH key is valid it will initiate a connection to the private instance. 
This way there are no exposed port toward the internet that can be targeted.


## 3. When a connection timed out instead of failing immediately, what was blocking it?

It's the Security Group (or NACL) dropping traffic without any feedback. Because Security Groups default to "deny all," any traffic that isn't explicitly allowed gets dropped.

Immediate failure is more of a software issue. EC2 instance not running or the listening port is different. 


## 4. Which security group rule would you never configure in a production environment, and why?

Inside the private instances the inbound traffic to be set global 0.0.0.0.0.
It would give another target surface to the wide internet and bots can brute force,
getting DDoS attacks. 
Violates the least priviliged principal among with breaking the defence in depth.



# Key Learnings

### Hardest part of this lab

Making sure that every security group is correctly setup (inbound,outbound rules).
Once the setup is done then to check if all of them works as intended
and also for me to actually imagine the connections without any connection drawings/ pictures for it. 

### One thing I will do differently in future AWS deployments

Before I setup any kind of security group I would write/ draw a skeleton to check if the groups are compliance ready 
and no miss configuration happens at the actual deployment. 

# Checklist

- [X] 4 security groups created with security group references
- [X] 4 EC2 instances launched
- [X] SSH works only through the bastion
- [X] Required services running on all tiers
- [X] All 7 connectivity tests completed
- [X] Blocked traffic confirmed with timeouts
- [X] End-to-end application test successful
- [X] Architecture documented
- [X] All required screenshots captured
- [X] Reflection questions completed
- [X] Changes committed to Git
- [X] Pull Request created


