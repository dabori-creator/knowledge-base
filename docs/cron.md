

# Adding a Script to Cron

## 1. Cron Basics

Cron configuration file:

```bash
nano /etc/crontab
```

*There is an alternative way to create a cron job using the `crontab -e` command, but in this case, the job is created for a specific user and the Vim editor opens instead of nano.*

Each job is formatted as follows:

```
minute(0-59) hour(0-23) day(1-31) month(1-12) day_of_week(0-7) user /full/path/to/script or task
```

## 2. How to Use Operators

Operators allow you to specify multiple values in a field. There are four operators:

```
Asterisk (*): This operator sets all possible values for the field. For example, an asterisk in the "Hour" field is equivalent to every hour, and an asterisk in the month field is equivalent to every month, etc.

Comma (,): This operator specifies a list of values, for example: "1,5,10,15,20,25".

Dash (-): This operator specifies a range of values, for example: "5-15" days, which is equivalent to the set "5,6,7,8,9 … 13,14,15" when using the "Comma" operator.

Slash (/): This operator specifies a step value, for example: "0-23/" can be used in the hour field to indicate running the command every hour. Steps are also allowed after an asterisk, so if you need to run something every two hours, simply use "*/2".
```

## 3. Predefined Macros

There are several special Cron schedule macros used to define common intervals. You can use them instead of specifying the date in five columns.

```
@yearly (or @annually) - run the job once a year at midnight (12:00) on January 1st. Equivalent to 0 0 1 1 *.

@monthly - run the job once a month at midnight on the first day of the month. Equivalent to 0 0 1 * *.

@weekly - run the job once a week at midnight on Sunday. Equivalent to 0 0 * * 0.

@daily - run the job once a day at midnight. Equivalent to 0 0 * * *.

@hourly - run the job once an hour at the beginning of the hour. Equivalent to 0 * * * *.

@reboot - run the specified job at system startup (boot time).
```

## 4. Cron Job Examples

Below are some examples of cron jobs that show how to schedule a task for different time periods.

Run myscript every 5 minutes:

```
*/5 * * * * /root/script.sh
```

Run myscript every day at 1:00 AM:

```
0 1 * * * /root/script.sh
```

Run script every month on the 1st at 3:15 AM:

```
15 3 1 * * /root/script.sh
```

Run a command at 3:00 PM every day from Monday to Friday:

```
0 15 * * 1-5 command
```

Run a script every 5 minutes and redirect standard output to /dev/null, only standard error will be sent to the specified email address:

```
MAILTO=email@example.com
*/5 * * * * /path/to/script.sh > /dev/null
```

Run two commands every Monday at 3:00 PM (use the && operator between commands):

```
0 15 * * Mon command1 && command2
```

Run a PHP script every 2 minutes and write the result to a file:

```
*/2 * * * * /usr/bin/php /path/to/script.php >> /var/log/script.log
```

Run a script every day, every hour, from 8:00 AM to 4:00 PM:

```
00 08-16 * * * /path/to/script.sh
```

Run a script on the first Monday of every month at 7:00 AM:

```
0 7 1-7 * 1 /path/to/script.sh
```

Run a script at 9:15 PM on the 1st and 15th of every month:

```
15 9 1,15 * * /path/to/script.sh
```

Run main.py every hour:

```
@hourly python3 main.py
```