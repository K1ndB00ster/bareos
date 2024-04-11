# bareos
# bareos client installer/remoover
# This script not install bareos server, it's for quick setup clients on server only!
# Source code for python

import os

def main():
    print("Привіт! Зараз я допоможу тобі з Бареосом, обери бажану дію: ")
    print("1. Створити налаштування для нового клієнта на сервері")
    print("2. Видалити існуючого клієнта")
    choice = input("Що робити будемо, обери варійант: ")

    if choice == '1':
        #Створюємо змінні:
        clientname = input("Введіть ім'я клієнту: ")
        address = input('Введіть адресу клієнту: ')
        port = input('Вкажіть порт для бекапів (за замовчанням 9101-9103, оберіть один з них та вкажіть): ')
        key = input('Введіть ключ клієнту: ')
        folder = input('Вкажіть точний маршрут діректорії яку потрібно резервувати: ')
        timefull = input('Вкажіть бажаний час для повного бекапу у вигляді 13:00 : ')
        timeinc = input('Вкажіть бажаний час для інкрементального бекапу у вигляді 14:00 : ')

        #Задаємо маршрути до створення файлів
        client = open(f'/etc/bareos/bareos-dir.d/client/{clientname}.conf', 'w+')
        fileset = open(f'/etc/bareos/bareos-dir.d/fileset/{clientname}.conf', 'w+')
        job = open(f'/etc/bareos/bareos-dir.d/job/{clientname}.conf', 'w+')
        jobdefs = open(f'/etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf', 'w+')
        poolfull = open(f'/etc/bareos/bareos-dir.d/pool/{clientname}.conf', 'w+')
        poolinc = open(f'/etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf', 'w+')
        schedule = open(f'/etc/bareos/bareos-dir.d/schedule/{clientname}.conf', 'w+')

        #Створюємо необхідні для бекапів файли
        #Налаштування клієнту:
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

        #Налаштування директорії бекапів:
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

        #Робота для клієнту:
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

        #Робота за замовчанням для створеного клієнту:
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

        #Пул повних бекапів:
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

        #Пул інкрементальних бекапів:
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

        #Файл планування завдань, або планувальник:
        schedule.write(f'''Schedule {{
        Name = "{clientname}"
        Run = Level = Full on 1 at {timefull}
        Run = Level = Incremental mon-sun at {timeinc}
        }}
        ''')

        # Додаємо права та власника створеним файлам:
        os.system(f'chmod 640 /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')
        os.system(f'chown bareos:bareos /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')

        #Перезавантажуємо Бареос:
        os.system('systemctl restart bareos-dir bareos-sd bareos-fd')
    elif choice == '2':
        clientname = input("Введіть Ім'я клієнту, якого бажаєте видалити: ")
        os.system(f'rm -rd /etc/bareos/bareos-dir.d/client/{clientname}.conf /etc/bareos/bareos-dir.d/fileset/{clientname}.conf /etc/bareos/bareos-dir.d/job/{clientname}.conf /etc/bareos/bareos-dir.d/jobdefs/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}.conf /etc/bareos/bareos-dir.d/pool/{clientname}-inc.conf /etc/bareos/bareos-dir.d/schedule/{clientname}.conf')
        os.system('systemctl restart bareos-dir bareos-sd bareos-fd')
    else:
        print("Неправильний вибір. Програма завершує роботу.")

def action1():
    print("Ви обрали дію 1. Клієнта створено.")

def action2():
    print("Ви обрали дію 2. Клієнта видалено.")

if __name__ == "__main__":
    main()
