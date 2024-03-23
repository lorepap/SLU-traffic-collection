# SLU-traffic-collection
A group project for the class CS4930-51: Advanced Computer Networks. The project consists in the design, implementation and deployment of a network collection server to collect SLU Campus traffic.

## 1. System Architecture
![Screenshot 2024-03-23 at 5 42 32 PM](https://github.com/lorepap/SLU-traffic-collection/assets/56161227/0a0d1c49-af8f-4499-9cdd-8c6794d680c5)

## 2. Measurement Machine Logic
Using [PMACCT](http://www.pmacct.net/), a multi-purpose passive network monitory tool, we aimed to measure SLU's network traffic by configuring the following configuration files:

1. [nfacctd.conf](./nfacctd.conf)
2. [pmacctd.conf](./pmacctd.conf)
3. [sfacctd.conf](./sfacctd.conf)

After several attempts to get the netflow command, _nfacctd_, that collects NETFLOW metrics to work, we ended up using _pmacctd_ with the configuration file _pmacctd.conf_. It simply captures packets at the specified network interface instead of relying on NETFLOW protocol.

### 2.1 Log management
According to the latest version of _pmacctd.conf_, a csv file is written everyday to `/home/admin/traffic`. The configuration determines the frequency of logs being read from the network interface and written to the log file. Per the latest version, this is every 1 hour.

`pmacct_bg.py` is a cron job script that runs every Sunday at midnight. It transfers the logs in `/home/admin/traffic` from the measurement machine to hopper at `/lab/nrg/slunet/logs/`

All errors and reports are stored locally and an email is sent to receivers specified in the code.

After a log file is successfully transferred to hopper, it is removed from the measurement machine

### 2.2 Cron Jobs
In addition to the cron job that runs everyday to transfer network logs from the measurement machine to hopper, we have an addition job that initializes the _pmacct_ job anytime the measurement machine is restarted. This can be found by running the following commands:

```sh
$ crontab -e
```

After picking your favorite text editor (nano or vim or etc), the follow is revealed:

```txt
59 23 * * SUN python3 /home/pokorie/tool/pmacct_bg.py

@reboot sh /home/pokorie/tool/pm_start.sh
```

## 3. Hopper Logic

On hopper we have several python scripts for analysis. They include the following:

1. [log_reader.py](./log_reader.py): A python program that allows you to read a log file and print out the values as a JSON object on the terminal
2. [db_logger.py](./db_logger.py): A python program that examines new log files in the `/lab/nrg/slunet/logs`, reads its content and stores them in the _msql_ database. After each log file is processed, we compress it to a tar file and remove the original _csv_ file.
3. [unzip.py](./unzip.py): A python program that unzips a tar file and removes the tar file from the directory, `/lab/nrg/slunet/logs`

### 3.1 mySQL database

On hopper we have a _mysql_ database that allows us to store the values of the log files. [schema.sql](./schema.sql) specifies the schema of the database. The connection to the database is as follows:

```python
    app.config['MYSQL_DATABASE_HOST'] = 'db1.mcs.slu.edu'
    app.config['MYSQL_DATABASE_USER'] = <send an email request>
    app.config['MYSQL_DATABASE_PASSWORD'] = <send an email request>
    app.config['MYSQL_DATABASE_DB'] = 'slunet'
```

## 4. Installation
Run the following pip commands to install packages. Remove the "--user" flag if you are an admin user

```bash
$ pip3 install mysql-connector-python-rf --user
$ pip3 install sendgrid --user
$ pip3 install scp --user
$ pip3 install flask-mysql --user
$ pip3 install python-decouple --user
```

Create a `.env` file in the root folder of where you cloned this repo and enter the following contents
```txt
HOPPER_USER=<yourusername>
HOPPER_PASSWORD=<yourpassword>
MYSQL_DATABASE_HOST=db1.mcs.slu.edu
MYSQL_DATABASE_USER=<send an email request>
MYSQL_DATABASE_PASSWORD=<send an email request>
MYSQL_DATABASE_DB=slunet
SENDGRID_API_KEY=<send an email request>
```

## 5. Relevant Literature

1. [Collecting telemetry data with pmacct](http://www.pmacct.net/Lucente_collecting_netflow_with_pmacct_v1.3.pdf)
2. [Quick Start to PMACCT commands](https://github.com/pmacct/pmacct/blob/master/QUICKSTART)
3. [PMACCT Configuration Keys](https://github.com/pmacct/pmacct/blob/master/CONFIG-KEYS)



##  6. Errors Encountered
```txt
'aggregate' directive cannot include both src_host/dst_host and
src_as/dst_as. This is not supported as both HOST and AS primitives are
"multiplexed" in the same field. You can still fire two more plugins and
keep AS and HOST stats segregated each other.
```
This probably means that your configuration file is using conflicting configuration keys. Please check [PMACCT Configuration Keys](https://github.com/pmacct/pmacct/blob/master/CONFIG-KEYS) for the appropriate configuration keys for each program

