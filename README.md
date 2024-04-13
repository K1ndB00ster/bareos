<h1 style="background-color:rgba(12, 10, 174, 0.1);">Bareos client installer</h1>
<i>bareos client installer/remoover</i>
<p><strong>This script not install bareos server, it's for quick setup clients on server only!</strong></p>
<p><i>Python source code, this is my first project, so please don't judge too harshly</i></p>

----
<img src=https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54>
 <img src=https://img.shields.io/badge/cent%20os-002260?style=for-the-badge&logo=centos&logoColor=F0F0F0> <img src=https://img.shields.io/badge/-Rocky%20Linux-%2310B981?style=for-the-badge&logo=rockylinux&logoColor=white> <img src=https://img.shields.io/badge/Red%20Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white>

```python
import os

def main():
    print("Hello! Now I will help you with Bareos, choose the desired action: ")
    print("1. Create settings for a new client on the server")
    print("2. Delete an existing client")
    choice = input("What will we do, choose an option: ")

    if choice == '1':
        #Create variables:
        clientname = input("Enter client name: ")
        address = input('Enter ip address client: ')
        port = input('Specify the port for backups (by default 9101-9103, choose one of them and specify): ')
        key = input('Enter client key: ')
        folder = input('Specify the exact path of the directory to be backed up: ')
        timefull = input('Specify the desired time for a full backup in the form of 13:00: ')
        timeinc = input('Specify the desired time for the incremental backup as 14:00: ')

        #Set routes to create files
        client = open(f'/etc/bareos/bareos-dir.d/client/{clientname}.conf', 'w+')
        fileset = open(f'/etc/bareos/bareos-dir.d/fileset/{clientname}.conf', 'w+')
        job = open(f'/etc/bareos/bareos-dir.d/job/{clientname}.conf', 'w+')
        jobdefs = open(f'/etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf', 'w+')
        poolfull = open(f'/etc/bareos/bareos-dir.d/pool/{clientname}.conf', 'w+')
        poolinc = open(f'/etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf', 'w+')
        schedule = open(f'/etc/bareos/bareos-dir.d/schedule/{clientname}.conf', 'w+')

        #Create the necessary files for backups
        #Client settings:
        client.write(f'''Client {{
        Name = "{clientname}"
        Address = "{address}"
        Port = {port}
        Password = "{key}"
        File Retention = 6 months
        Job Retention = 6 months
        Auto Prune = yes
        }}
        ''')

        #Setting up the backup directory:
        fileset.write(f'''FileSet {{
        Name = "{clientname}"
        Enable VSS = yes
        Include {{
            Options {{
            compression = GZIP
            Signature = MD5
            Drive Type = fixed
            IgnoreCase = yes
            }}
            File = "{folder}"
        }}
        }}

        ''')

        #Job for the client:
        job.write(f'''Job {{
        Name = "{clientname} Backup"
        Description = "Backup {clientname} (after the nightly save)"
        JobDefs = "{clientname}"
        Level = Full
        FileSet="{clientname}"
        Schedule = "{clientname}"

        # This creates an ASCII copy of the catalog
        # Arguments to make_catalog_backup are:
        #  make_catalog_backup <catalog-name>
        RunBeforeJob = "/usr/lib/bareos/scripts/make_catalog_backup MyCatalog"

        # This deletes the copy of the catalog
        RunAfterJob  = "/usr/lib/bareos/scripts/delete_catalog_backup MyCatalog"

        # This sends the bootstrap via mail for disaster recovery.
        # Should be sent to another system, please change recipient accordingly
        Write Bootstrap = "|/usr/bin/bsmtp -h localhost -f \\\"\(Bareos\) \\\" -s \\\"Bootstrap for Job %j\\\" root"
        Priority = 11                   # run after main backup
        }}
        ''')

        #Job by default for the created client:
        jobdefs.write(f'''JobDefs {{
        Name = "{clientname}"
        Type = Backup
        Level = Incremental
        Client = {clientname}
        FileSet = "{clientname}"                     # selftest fileset
        #Schedule = "{clientname}"
        Storage = File
        Messages = Standard
        Pool = {clientname}
        Priority = 11
        Write Bootstrap = "/var/lib/bareos/%c.bsr"
        Full Backup Pool = {clientname}                  # write Full Backups into "Full" Pool
        #Differential Backup Pool = Differential  # write Diff Backups into "Differential" Pool
        Incremental Backup Pool = {clientname}-inc    # write Incr Backups into "Incremental" Pool
        }}
        ''')

        #A pool of full backups:
        poolfull.write(f'''Pool {{
        Name = {clientname}
        Pool Type = Backup
        Recycle = yes                       # Bareos can automatically recycle Volumes
        AutoPrune = yes                     # Prune expired volumes
        Volume Retention = 6 months         # How long should the Full Backups be kept?
        #Maximum Volume Bytes = 50G         # Limit Volume size to something reasonable
        Maximum Volume Jobs = 4
        Maximum Volumes = 7                 # Limit number of Volumes in Pool
        Label Format = "{clientname}-full-"              # Volumes will be labeled "Full-<volume-id>"
        }}

        ''')

        #Pool of incremental backups:
        poolinc.write(f'''Pool {{
        Name = {clientname}-inc
        Pool Type = Backup
        Recycle = yes                       # Bareos can automatically recycle Volumes
        AutoPrune = yes                     # Prune expired volumes
        Volume Retention = 6 months          # How long should the Incremental Backups be kept?
        #Maximum Volume Bytes = 1G          # Limit Volume size to something reasonable
        Maximum Volume Jobs = 23
        Maximum Volumes = 7                 # Limit number of Volumes in Pool
        Label Format = "{clientname}-inc-"       # Volumes will be labeled "Incremental-<volume-id>"
        }}

        ''')

        #Task scheduling file, or scheduler:
        schedule.write(f'''Schedule {{
        Name = "{clientname}"
        Run = Level = Full on 1 at {timefull}
        Run = Level = Incremental mon-sun at {timeinc}
        }}
        ''')

        #Add the rights and owner to the created files:
        os.system(f'chmod 640 /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')
        os.system(f'chown bareos:bareos /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')

        #Reload Bareos:
        os.system('systemctl restart bareos-dir bareos-sd bareos-fd')
    elif choice == '2':
        clientname = input("Enter the Name of the client you want to delete: ")
        os.system(f'rm -rd /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')
        os.system('systemctl restart bareos-dir bareos-sd bareos-fd')
    else:
        print("Wrong choice. The program terminates.")

def action1():
    print("You have chosen action 1. The client has been created.")

def action2():
    print('You have chosen action 2. The client has been deleted.')

if __name__ == "__main__":
    main()
```
