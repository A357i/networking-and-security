# Computer Fundamentals

> Covers: Inside a Computer System, Computer Types, Client-Server Basics, Virtualisation Basics, Cloud Computing Fundamentals (Pre-Security Section 2)

---

## 1. Inside a Computer System
- **CPU**: executes instructions; clock speed & cores affect processing power.
- **RAM**: volatile, fast working memory — lost on power-off.
- **Storage**: HDD/SSD — persistent. SSDs are faster but write-wear differs from HDDs (relevant for forensic recovery too).
- **Motherboard**: connects all components; BIOS/UEFI boots the system before the OS loads.
- **I/O devices**: keyboard, monitor, NIC (network interface card) — the NIC is your entry point for all networking topics.

## 2. Computer Types
Desktop, laptop, server (built for uptime/throughput, not user comfort), embedded systems (fixed-purpose, e.g. routers/IoT — often under-patched and a common attack target), mobile devices.

## 3. Client-Server Basics
- **Client**: requests a service (your browser).
- **Server**: provides the service (web server, DB server).
- Model underlies almost everything you'll attack: web apps, APIs, DNS, SSH — a client initiates, a server responds. Recognizing this pattern is what makes protocol analysis approachable.

## 4. Virtualisation Basics
- **Hypervisor Type 1 (bare-metal)**: runs directly on hardware (ESXi, Hyper-V) — used in enterprise/cloud.
- **Hypervisor Type 2 (hosted)**: runs on top of an OS (VirtualBox, VMware Workstation) — what you likely use for your home lab.
- **Snapshots**: save VM state — essential for safe malware analysis or testing exploits without permanent damage.
- **Why it matters for you**: your entire lab (attack box + vulnerable targets) is virtualized; isolation between VMs is a security boundary worth understanding, not just a convenience.

## 5. Cloud Computing Fundamentals
- **IaaS** (Infrastructure as a Service): raw compute/storage/network (AWS EC2, Azure VMs) — you manage the OS up.
- **PaaS** (Platform as a Service): managed runtime (Heroku, Elastic Beanstalk) — you manage code, not infra.
- **SaaS** (Software as a Service): fully managed app (Gmail, Salesforce) — you manage data/config only.
- **Shared Responsibility Model**: cloud provider secures the infrastructure "of" the cloud; you secure what you put "in" it (data, IAM config, app-layer security). Misconfigured S3 buckets and IAM roles are a huge real-world attack surface because this line gets misunderstood.

---

## Open Questions / To Expand
- [ ] Add specifics on hypervisor escape risks once covered
