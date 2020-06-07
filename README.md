
# SysBench Testing of a MariaDB Database


## TL;DR

Clone this project, set your configuration information in the `.sysbench.cfg` file and run the `sysbench.sh` script. To be able to run mulitple tests over multiple thread numbers I have automated this with a simple bash script [Automating tests and results](README.md#Automating-tests-and-results)

## What is SysBench?

SysBench is an open-source tool for running a multitude of tests on your system. It can test Memory, Disk, CPU and the performance of your database.

Sysbench is like a swissarmy knife of benchmarking tools, it covers all sorts of tests , but we are going to use it for testing the database.

## Why SysBench?

There are various tools for testing your database, people create their own, or they may use `mysqlslap`.  `mysqlslap` is not a fair test on the server, it only runs one type of query on the database at a time whereas `sysbench` provides a repeatable set of tests mimicking real-life traffic.  

It is written using LUA. LUA was a scripting language used for computer games. You can write your own tests, but we are going to use one of the shipped tests.

## The Tests?

Locate the SysBench files on your server, on my Mac they are in `/usr/local/Cellar/sysbench/1.0.20/share/sysbench/`, you will see that there are some tests delivered by the install. if you `ls` the directory you should see something similar to this:

```
bulk_insert.lua           oltp_delete.lua           oltp_point_select.lua     oltp_read_write.lua       oltp_update_non_index.lua select_random_points.lua  tests
oltp_common.lua           oltp_insert.lua           oltp_read_only.lua        oltp_update_index.lua     oltp_write_only.lua       select_random_ranges.lua
```

There are a selection of Online Transactional Processing (OLTP) scripts, ready for you to run against your database.

## Installation

To install SysBench you can use yum `yum install sysbench` or on a Mac `brew install sysbench`.

Please beaware that the current version of SysBench installs mysql-client 8.0.19 to your machine.

These tests were carried out using `sysbench 1.0.20`, you can check the version with `sysbench --version`.

Ensure you have enough threads available, I had to set this on my Mac:

```
launchctl limit maxfiles
sudo launchctl limit maxfiles 65536 200000
```

## Using SysBench

The first part of using SysBench is the initialisation, this creates a database on the server. Once the server is initialised you can carry out the tests.

### Initialisation

You must create a database instance for the testing to take place. To do this run `mysqladmin create sysbenchtest`.

You can check the database is available with `mariadb -e "SHOW DATABASES"`

```bash
➜  ~ mariadb -e "SHOW DATABASES"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sysbenchtest       |
+--------------------+
➜  ~
```

### To prepare the system execute this command:

```
sysbench oltp_read_write --db-driver=mysql --table-size=$lv_table_size --mysql-user=$lv_username --mysql-password=$lv_password --mysql-db=$lv_database --mysql-host=$lv_host --mysql-port=$lv_port --threads=12 prepare
```
This will create a table in the `sysbenchtest` database. You can check this with a `SHOW CREATE TABLE` command like this:

```
➜  ~ mariadb sysbenchtest -e "SHOW CREATE TABLE sbtest1"
+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                                                                                                                                                                                                        |
+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| sbtest1 | CREATE TABLE `sbtest1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT 0,
  `c` char(120) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '',
  `pad` char(60) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=10001 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci |
+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
➜  ~
```

From here you can see that there is a new table created. If you do a `SELECT COUNT(*)` from the new table, you can see there are 10,000 records inserted into the database:

```
➜  ~ mariadb sysbenchtest -e "SELECT COUNT(*) FROM sysbenchtest.sbtest1"
+----------+
| COUNT(*) |
+----------+
|    10000 |
+----------+
➜  ~
```

### Running a test

To ensure that everything works you can carry out a basic test. We will rerun the same command, but this time with the `run` command and not `prepare`.

```
sysbench oltp_read_write --db-driver=mysql --table-size=$lv_table_size --mysql-user=$lv_username --mysql-password=$lv_password --mysql-db=$lv_database --mysql-host=$lv_host --mysql-port=$lv_port  --threads=12 run
```

When the test finishes running you will be presented with a result set:

```
Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            350
        write:                           100
        other:                           50
        total:                           500
    transactions:                        25     (2.44 per sec.)
    queries:                             500    (48.88 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.2260s
    total number of events:              25

Latency (ms):
         min:                                  402.52
         avg:                                  408.89
         max:                                  415.59
         95th percentile:                      411.96
         sum:                                10222.25

Threads fairness:
    events (avg/stddev):           25.0000/0.00
    execution time (avg/stddev):   10.2222/0.00
```

### Cleanup

Once you have run a test and recorded the results, you will need to cleanup the system, ready for preparing future tests. To cleanup you csn run the same sysbench command again, but this time with a `cleanup` parameter and not a `run`:

```
➜  ~ sysbench oltp_read_write --db-driver=mysql --mysql-user=$lv_username --mysql-password=$lv_password --mysql-db=$lv_database --mysql-host=$lv_host --mysql-port=$lv_port cleanup
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Dropping table 'sbtest1'...
➜  ~
```

### The results

When you run `sysbench` various statistics are returned to you on the screen. If you are comparing configurations you will generally be interested in the transactions and how many the system could perform per second.

Latency should also be very important to you, the lower the latency the quicker the activity took place. If there is an unstable or slow network Latency could be much higher.


# Automating tests and results

To be able to fully test an installation, and spot errors, we should automate the testing with various variables, however to compare results before and after modifications, or accross different systems the tests should be consistent.

Be sure you have SysBench installed and available on your system.

## 1: Git Clone this project:


```bash
git clone https://github.com/kesterriley/mariadb-sysbench.git
cd ~/mariadb-sysbench
```

## 2: Set the configuration

Fill your details in to the .sysbench.cfg and ensure your file is only Read and Writable by you `chmod 600 .sysbench.cfg`

## 3: Run the Script

Make the script executable for the user `chmod 700 sysbenchtest.sh` and then

```bash
./sysbenchtest.sh
```

## 4: View the results

The script will generate for you a `./results` folder and the results for each test ran will be placed here, it will also generate a CSV file for you.

## Contributors

Thanks to the following people who have contributed to this project:

[@kesterriley](https://github.com/kesterriley)

## Contact

If you want to contact me you can reach me at kesterriley@hotmail.com.

## License
<!--- If you're not sure which open license to use see https://choosealicense.com/--->

This project uses the following license: [MIT](https://github.com/kesterriley/my-helm-charts/blob/master/LICENSE).
