1. Wanted to write a script that takes in a file of usernames, outputs a curl command for each

```bash
while IFS= read -r line
do
	curl http://faculty.academy.htb:50879/courses/linux-security.php7 -X POST -d "username=$line" -H 'Content-Type: application/x-www-form-urlencoded'
done < $1
```

`while <command>; do`
- Loops as long as the command returns success

`IFS= read -r line`
- `IFS=`: Internal Field Separator set to empty string to preserve whitespace issues
	- Basically, this being here **removes** leading and trailing whitespace characters
- `-r`: to not allow backslashes to escape characters
- in each iteration, `$line`  set to each line from the input to the command

`<$1`
- Duplicates the file onto **stdin (fd 0)** for the `while .. done` block
- Recap: [[Pipe, Redirect, File Descriptors]]
- why not Pipe instead?
	- this will spawn whilein a subshell, so variables set inside may not persist afterwards

Could also have done
```bash
exec 3< $1
while IFS= read -r -u 3 line; do
  # commands here can use stdin freely
done
exec 3<&-   # close fd 3
```

- `exec 3<$1`
	- `exec` with no program modifies current she''s file descriptors
	- `3<$1` opens the file specified by argument, attaches it to `fd 3`
	- `-u 3` tells read to read from fd 3 instead of the default stdin (fd 0).

`"username=$line"`
- Important to use double quotes `(" ")` instead of single quotes `(' ')` here!
	- To allow expansion.
