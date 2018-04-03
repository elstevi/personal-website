<!--
.. title: Backup Wordpress website offsite with Tarsnap
.. slug: backup-wordpress-website-offsite-with-tarsnap
.. date: 2018-03-27 22:55:24 UTC-07:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

I use <a href='https://www.tarsnap.com/'>tarsnap</a> because it sticks to the unix philosophy...it does one thing (backups) very well. Encryption is done on the client before the backup ever leaves, and it is as easy as using tar.

<h2>Make sure you are signed up with tarsnap</h2>
If you haven't already, <a href=''>go get an account at tarsnap</a>. You will need one to complete this tutorial. I put about $5 in my account, and have never refilled it.
<!-- TEASER_END -->
<h2>Install tarsnap</h2>
This can be done with ports or packages, I am going to pick packages because lazy.
```
root@stevendouglas:~ # pkg install -y tarsnap
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 1 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        tarsnap: 1.0.37_1

Number of packages to be installed: 1

224 KiB to be downloaded.
[stevendouglas.me] [1/1] Fetching tarsnap-1.0.37_1.txz: 100%  224 KiB 229.2kB/s    00:01
Checking integrity... done (0 conflicting)
[stevendouglas.me] [1/1] Installing tarsnap-1.0.37_1...
[stevendouglas.me] Extracting tarsnap-1.0.37_1: 100%
Message from tarsnap-1.0.37_1:
================================================================================

If you have never used tarsnap before, you will need to create an account
with the tarsnap service and deposit money into the account before you
can start using tarsnap; see
  https://www.tarsnap.com/gettingstarted.html
for details.

Once you have a tarsnap account you will need to create a key file using
the tarsnap-keygen utility before you start storing archives; this key
file MUST BE KEPT IN A SAFE LOCATION since you will not be able to read
your backups without it.

================================================================================

```

<h2>Configure the host for tarsnap use</h2>
Take this example command from <a href='https://www.tarsnap.com/gettingstarted.html'>tarsnap's legit documentation</a>:
```
sudo tarsnap-keygen --keyfile /root/tarsnap.key --user me@example.com --machine mybox
```
and retrofit it to suit your needs. The <code>--user</code> flag will need to match your tarsnap account username. The machine can really be whatever you want, I generally use the <code>hostname</code>.

This will register the machine it is run on with tarsnap. It will ask for your password.

Take the <code>/root/tarsnap.key</code> and save it somewhere. Without this key, you will be unable to restore from your backups! Also, anyone who acquires this key will be able to restore your backups!

For tarsnap keys, I generally keep two copies. I have one that gets put into my password manager, and one gets put in a safe deposit box. Seriously. They are cheap and a relatively secure place to keep things. I suggest getting a safe deposit box that is relatively accessible and geographically diverse from your home.

<h2>Write a backup script</h2>
This is the backup script I wrote for backing up this wordpress site:
```
#!/bin/sh
echo "Backup starting - `date`"
BACKUPS_TO_KEEP=60
BACKUP_NAME='stevendouglas.me-backup'
STAGING_DIR="/tmp/backups/${BACKUP_NAME}"
WORDPRESS_DIR='/usr/local/www/wordpress'
WP_CONFIG="${WORDPRESS_DIR}/wp-config.php"

MYSQL_HOST=`grep -r -i DB_HOST ${WP_CONFIG} | cut -d ' ' -f2 | cut -d \' -f2`
MYSQL_NAME=`grep -r -i DB_NAME ${WP_CONFIG} | cut -d ' ' -f2 | cut -d \' -f2`
MYSQL_PASS=`grep -r -i DB_PASSWORD ${WP_CONFIG} | cut -d ' ' -f2 | cut -d \' -f2`
MYSQL_USER=`grep -r -i DB_USER ${WP_CONFIG} | cut -d ' ' -f2 | cut -d \' -f2`

mkdir -p ${STAGING_DIR}
printf "Dumping database..."
mysqldump --single-transaction -u ${MYSQL_USER} -p${MYSQL_PASS} -h ${MYSQL_HOST} ${MYSQL_NAME} > ${STAGING_DIR}/db.sql
echo "done"

printf "Copying wordpress files..."
cp -R ${WORDPRESS_DIR} ${STAGING_DIR}
echo "done"

printf "Running tarsnap..."
tarsnap -c -f "${BACKUP_NAME}-$(date +%Y-%m-%d_%H-%M-%S)" "${STAGING_DIR}" 2> /dev/null
printf "$?."
echo "done"
rm -rf ${STAGING_DIR}

CURRENT_BACKUPS=`tarsnap --list-archives | grep ${BACKUP_NAME} | wc -l | tr -d ' '`
echo "We want to have ${BACKUPS_TO_KEEP} backups, and have ${CURRENT_BACKUPS}."
if [ "${CURRENT_BACKUPS}" -gt "${BACKUPS_TO_KEEP}" ]; then
        echo "trimming old backups"
        BACKUPS_TO_TRIM=`echo ${CURRENT_BACKUPS}-${BACKUPS_TO_KEEP} | bc`
        DELETE_BACKUPS=`tarsnap --list-archives | grep ${BACKUP_NAME} | sort -r | tail -n ${BACKUPS_TO_TRIM}`

        for BACKUP in $DELETE_BACKUPS; do
                printf "Deleting ${BACKUP}..."
                tarsnap -d -f ${BACKUP} 2> /dev/null
                echo "DONE"
        done
fi
echo "Backup finishing - `date`"
```

<h2>Running at an interval</h2>
This line in cron should do it:
```
0 2 * * * /root/backup.sh > /var/log/backup 2>&1
```
You are now backing up daily at 2AM.

<h2>Restoring</h2>
First find the backup you want to restore:
```
root@stevendouglas:~ # tarsnap --list-archives | sort
stevendouglas.me-backup-2017-06-25_19-02-24
stevendouglas.me-backup-2017-06-25_19-08-35
stevendouglas.me-backup-2017-06-25_19-16-08
```

And use this command to restore:
```
tarsnap -x -f stevendouglas.me-backup-2017-06-25_19-16-08
```

And our files are restored:
```
root@stevendouglas:~ # ls -lh tmp/backups/stevendouglas.me-backup/                                                                                                                                    
total 234
-rw-r--r--  1 root  wheel   678K Jun 25 19:16 db.sql
drwxr-xr-x  6 root  wheel    23B Jun 25 19:16 wordpress
```

<h2>Conclusion</h2>
You did backup your key...right? Tarsnap is a wonderfully simple service that does exactly what it says on the box. 
