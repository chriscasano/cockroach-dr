# Disaster Recovery Examples

## Run Differentials

This tutorial shows how you can recover data from a either a fat finger or malicious data attack.  This tutorial can be utilized if the data issue is found within the Cockroach GC window for a particular zone configuration which by default is 25 hours.  An additional tutorial will show how to recover using a backup.

### Setup Movr with a bad actor record

For this example, we'll create a MOVR workload and show how to recover from a malicious attack from a user called devil.  On the MOVR rides table, let's activate an audit log so we can see all of the SQL transactions that incur on the table.  While the workload is running, we'll inject the rogue user called devil and have that user create a malicious transaction which updates all of Tyler Dalton's transactions to be $1000.

```
cockroach workload init movr --drop
cockroach sql --insecure -e "alter table movr.rides EXPERIMENTAL_AUDIT set READ WRITE;"
cockroach workload run movr --max-rate 10 --duration 10s
cockroach sql --insecure -e "create user devil; grant select, update on movr.* to devil;"
cockroach sql --insecure -u devil -d movr -e "update rides set revenue = 1000 where rider_id = (select id from users where name = 'Tyler Dalton');"
cockroach workload run movr --max-rate 10 --duration 10s
```

### Identify Customer Complaint

"One day, Tyler calls us and says...'What are all of these $1000 charges?''"

```
cockroach sql --insecure -d movr -e \
"select r.* \
from users u inner join rides r \
on u.id = r.rider_id \
where u.name = 'Tyler Dalton' \
and r.revenue = 1000;"
```

### Identify Transactions in Audit Log

"Let's go to the audit logging directory and do a search for transactions containing $1000"

```
cat cockroach-sql-audit.log | grep 1000 | grep Tyler
```

`I200829 18:30:13.261462 10900 sql/exec_log.go:188  [n1,client=127.0.0.1:52979,hostnossl,user=devil] 5 exec "$ cockroach sql" {"rides"[55]:READWRITE, "rides"[55]:READ} "UPDATE rides SET revenue = 1000 WHERE rider_id = (SELECT id FROM users WHERE name = 'Tyler Dalton')" {} 12.192 13 OK 0
I200829 18:30:35.327135 11917 sql/exec_log.go:188  [n1,client=127.0.0.1:53043,hostnossl,user=root] 9 exec "$ cockroach sql" {"rides"[55]:READ} "SELECT r.* FROM users AS u INNER JOIN rides AS r ON u.id = r.rider_id WHERE (u.name = 'Tyler Dalton') AND (r.revenue = 1000)" {} 4.635 13 OK 0
`

### Show privs for the user devil and remove them ASAP.

```
cockroach sql --insecure -d movr -e "show grants for devil;"
cockroach sql --insecure -d movr -e "revoke all on movr.* from devil; drop user devil;"
```

### Get the correct ride values before devil applied the malicious update:

Get ride balances before the rogue update was applied.  Using the first two columns of the audit log file, construct the timestamp for an as of system time query.  Below is an example:

From audit log: "I200829 18:30:13.261462 ...."  which can be represented like this for "2020-08-29 18:30:13.261462".  Let's make this 1 second earlier so we can see the values before the change was made.  "2020-08-29 18:30:12"

```
cockroach sql --insecure -d movr -e \
"select r.id, r.revenue \
from users u inner join rides r \
on u.id = r.rider_id as of system time '<insert timestamp from the audit log>' \
where u.name = 'Tyler Dalton';"
```

example: "select r.id, r.revenue from users u inner join rides r on u.id = r.rider_id as of system time '2020-08-29 18:30:12' where u.name = 'Tyler Dalton';"

### Restore the correct data values

"The following query returns the correct value of the rides records before the malicious transaction was applied.  Using the output from the query below, you can apply each of these updates to restore the rides table back to it's original state."

```
cockroach sql --insecure -d movr -e \
"select 'update movr.rides set revenue = ' || r.revenue::STRING || ' where id = ' || '''' || r.id::STRING || '''' || ';' \
from users u inner join rides r \
on u.id = r.rider_id as of system time '<insert timestamp from the audit log>' \
where u.name = 'Tyler Dalton';"
```

For each of the values that are returned up above, you can apply those values into a cockroach sql shell to update to ride transactions to the previous value.

###  Cleanup

cockroach sql --insecure -e "set sql_safe_updates = false; drop database movr;"
