# Pipe ( | )
- Pipes the **stdout** of one command into the **stdin** of the next
	- eg. `ps aux | grep nginx | wc -l
		- ps aux → piped to grep nginx → piped to wc -l.

# File descriptors
- When `open()/socket()/pipe()`, kernel returns an **int** (0,1,2)
	- Process uses that int with syscalls like `read(fd,..)`, `write(fd,...`
- Standard FDs:
	- 0: `stdin` (input)
	- 1: `stdout` (output)
	- 2: `stderr` (errors)
- Duping
	- `dup2(oldfd, newfd)`
		- makes newfd refer to the same open-file object as oldfd
# Redirect ( > )
- Pipes the **stdout** of one command into a **file** (by default)
	- So by default, `>` is actually `1>
- Can specify which file descriptor to pipe out to the file
	- `cmd > out.txt` -> make FD 1 point to out.txt
	- `cmd 2> err.txt` -> make FD 2 point to err.txt
	- `cmd > all.txt 2>&1` -> point 2 at 1
		- &1: duplicate an existing file descriptor
		- 2>&1: make fd2 (stderr) a duplicate of fd1 (stdout)
			- **ie. stderr (2) goes where stdout (1) is going.
		- conceptually `dup2(1, 2)
		- this means **stdout and stderr OUTPUT to the same file**
- Can use >& (no space!) to duplicate fd (see above)
	- The &  just means **treat this as a file descriptor and not filename**

Sequence matters!
eg. 
- `cmd > all.txt 2>&1`
	- Initially, stdout goes to terminal, stderr goes to terminal
	- `cmd > all.txt` (by default, > takes 1>)
		- stdout of cmd goes to all.txt, stderr goes to terminal
	- `2>&1`
		- duplicate fd1 (current stdout) onto fd2 
		- stderr goes where stdout is going (all.txt, from above)
		- so both go to all.txt

versus
- `cmd 2>&1 > all.txt`
	- Initially, stdout goes to terminal, stderr goes to terminal
	- `2>&1`
		- duplicate fd1 (current stdout) onto fd2 
		- stderr goes where stdout is going (terminal, so no change)
		- so both go to terminal
	- `> all.txt`
		- equivalent to `1> all.txt`
		- so stdout goes to all.txt
		- stderr remains at terminal