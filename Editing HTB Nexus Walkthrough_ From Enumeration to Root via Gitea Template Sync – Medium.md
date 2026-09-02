## HTB Nexus Walkthrough: From Enumeration to Root via Gitea Template Sync

Target: Nexus (HTB)

Vulnerability 1: CVE-2026–38526 (Krayin CRM RCE via TinyMCE Upload)

Vulnerability 2: Gitea Template Sync Privilege Escalation


Difficulty: Likely Hard/Medium (due to complex local attack path)
Vulnerability: Gitea Template Sync vulnerability / Local Privilege Escalation via Template Repository Git Hooks.

<img width="627" height="723" alt="image" src="https://github.com/user-attachments/assets/9c5045e6-5f54-45f0-a43f-eb3e1e82cc28" />

## Step 1: Initial Exploitation of Krayin CRM

Our initial attack vector is a Remote Code Execution (RCE) vulnerability in Krayin CRM. We are provided with a Python exploit script that takes advantage of an insecure file upload endpoint.

The Exploit Mechanism:
As seen in the source code, the script performs the following actions:


1-Visits /admin/login to scrape a CSRF token.

2-Authenticates to the admin panel using the provided credentials.

3-Sends a POST request to /admin/tinymce/upload containing a PHP webshell disguised with a GIF header

( GIF89a ).

4-Retrieves the location of the uploaded file from the server's JSON response.

5-Uses curl to pass commands to the webshell via the cmd GET parameter, effectively giving us unauthenticated RCE on the backend.

We execute the exploit against http://billing.nexus.htb/ using the discovered initial credentials for
user j.matthew@nexus.htb .

<img width="800" height="586" alt="image" src="https://github.com/user-attachments/assets/9667c84a-04af-4f49-b811-0213300e2718" />

Show the cat exploit.py command output, showing the Python script used.

. Caption: Fig 1. The Python exploit code for CVE-2026-38526 used to gain initial RCE on the Krayin CRM.

## Step 2: Enumeration and Database Leak

Once we have RCE, we immediately look for high-value files. We use the exploit’s -c command argument to read Krayin CRM's .env configuration file.

<img width="744" height="840" alt="image" src="https://github.com/user-attachments/assets/531fbcee-b485-4b6d-b1ac-ad35865636dd" />

Show the python3 exploit.py run along with the cat /var/www/krayin/.env command output.

Caption: Fig 2. Leveraging the RCE to read the application’s .env file, revealing sensitive credentials and internal API keys.

Show the exploit.py run along with the cat /var/www/krayin/.env command output.

Caption: Fig 1. Retrieving the .env file reveals internal credentials (DB password, API keys, and the password

for j.matthew@nexus.htb ).

## Step 2: Pivot SSH Access

The .env file we just leaked contains the password N27xh!!2ucY04 for the user j.matthew@nexus.htb . However, enumerating the system further reveals that the machine has a user named jones . We test these credentials and successfully SSH into the system as jones .

<img width="714" height="789" alt="image" src="https://github.com/user-attachments/assets/d2dffd43-34be-4c14-afc3-a4eca7dfe6e8" />

Show the successful ssh jones@10.129.1.5 command and the resulting shell.

Caption: Fig 3. Using the leaked credentials to pivot from RCE to an SSH shell as the jones user.

## Show the successful ssh jones@10.129.1.5 command and the resulting shell.

Caption: Fig 2. Obtaining initial shell access to the target machine as user jones .


## Step 3: Enumerating Local Services & The Attack Path

<img width="736" height="820" alt="image" src="https://github.com/user-attachments/assets/b83e0eb8-0284-4a9e-b749-6e04e66c5c16" />

<img width="721" height="841" alt="image" src="https://github.com/user-attachments/assets/3691b7ed-f687-415e-aaab-784a30a7c50f" />

Upon inspecting the local environment, you observe that the /var/log/template-sync.log log file is rapidly being written to with entries like Template sync starting . This indicates that a cron job or systemd service is running to sync Git templates. You also notice git.nexus.htb resolves locally to 127.0.0.1:3000 . This local service is likely Gitea.

Show the cat /var/log/template-sync.log command demonstrating the automated sync behavior.

Caption: Fig 3. Analyzing the template-sync.log reveals a critical background service synchronizing Git templates, a perfect path for a pivot to root.

## Step 4: Writing the Exploit Script

Because of the template sync mechanism, creating a repository on the local Gitea instance with the template: true flag will automatically cause the root-owned service to pull and execute your repository. The exploit works by adding a Git Hook (pre-receive or post-receive) to this template that writes your SSH public key

to /root/.ssh/authorized_keys .

You create an exploit script (similar to the provided exploit.py or exploit.sh ) that performs the following:

1-Generates an SSH key pair.
2-Authenticates to the local Gitea instance ( 127.0.0.1:3000 ) using jones 's credentials.
3-Requests an API Token via curl .
4-Creates a new repository named rce with "template": true .
5-Pushes the malicious code (containing the SSH key) to the repo.

<img width="717" height="835" alt="image" src="https://github.com/user-attachments/assets/ae009e5a-74d6-471f-aff4-471a4d7f256c" />

Show the local crafting of the exploit.py file and the errors encountered while debugging the syntax before getting it right.

Caption: Fig 4 & 5. Writing and debugging the Python/Bash exploit that leverages the Gitea API to create a malicious template repository.

## Step 5: Triggering the Template Sync Exploit

You run the final working exploit. The script sets export GITEA="http://127.0.0.1:3000" to bypass DNS issues, hits the local API, and creates the repository. Once pushed, the script waits (e.g., 65 seconds in the screenshots) for the background template-sync service to detect the new template and deploy it (running the hook as root, which injects your SSH public key into the root user's authorized_keys file).



Show the successful git push output and the "Exploit successfully bypassed Git checks and deployed!" message.

Caption: Fig 6. The exploit successfully pushes the malicious template repo and waits for the background service to execute the payload with root privileges.

## Step 6: Root Access & Capture the Flag

After the 65-second wait, you initiate an SSH connection to root@127.0.0.1 using the private key you generated earlier. You land as the root user. The final objective is met by reading the root.txt flag.

<img width="730" height="838" alt="image" src="https://github.com/user-attachments/assets/f1aec284-3753-4410-aca5-64714dfcb497" />

Show the ssh root@127.0.0.1 command leading to a root shell, followed by ls and cat root.txt .

Caption: Fig 7. Successfully obtaining a root shell.
