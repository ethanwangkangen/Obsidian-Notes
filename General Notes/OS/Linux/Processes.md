**ps**
- each process has PIC
- to see oprocesses run by others users and those that don't run from a session, `ps aux`

**kill**
- kill process

# How processes start
- systemd is first process
	- pid = 0
	- all other processes are child processes of systemd

# Starting processes on boot
- systemctl \[option] \[service]
- options:
	- start
	- stop
	- enable 
	- disable

# Foreground and background
- & command to run something in background
	- or ctrl+z
- fg to bring it back to foreground

# Cron
- time based job scheduler
- runs in background as daemon
- `crontab` file is where cron jobs are listed

`/* * * * * command_to_run`
`│ │ │ │ │`
`│ │ │ │ └─ Day of the week (0–7) (Sun=0 or 7)`
`│ │ │ └─── Month (1–12)`
`│ │ └───── Day of the month (1–31)`
`│ └─────── Hour (0–23)`
`└───────── Minute (0–59)`

eg. run myscript.sh everyday at 2.30am
`30 2 * * * /home/user/myscript.sh`

View ctrontabl:
- crontabl -l
Edit crontab:
- crontab -e
Remove crontab: