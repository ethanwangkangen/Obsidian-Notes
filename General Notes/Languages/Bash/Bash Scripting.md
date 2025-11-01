
# Shebang
`#!/bin/bash`
- Where is bash located
- If python, would be `#!/usr/bin/env python`

# Arguments
- `$0` - Name of script
- `$1 to $9` -  Arguments


# Variables
- Assigning: NO SPCE
	- eg. `domain=$1`
- Using variables
	- `echo $domain
		- parameter expansion: replace `$domain with the value of the variable `domain
- Command substitution
	- `$(cmd)`
		- run `cmd`, then replace with its stdout (trailing newline trimmed)
	- eg. `netrange=$(whois $ip | grep "NetRange\|CIDR" | tee -a CIDR.txt`
		- Runs the whole pipeline inside the parenthese, replaces it with its stdout.
		- The result is assigned to netrange.
		- Without this, netrange set to whois, then the next command becomes whatever $ip expands to 

## Usage of $
See $ as an inline replacement operator:
- Need a variable’s text? → $var
- Need another command’s output inline? → $(...)
- Need a quick integer calc inline? → $((...))

## Usage of "" and ''
- Word splitting
	- After Bash expands variables/commands substitutions (with $), it splits the result into separate arguments.
		- eg. 
		- `name="Ethan Wang"
		- `echo $name` -> echo Ethan Wang (two words)
		- `echo "$name"` -> one word `<Ethan Wang>
	- Use "" to **prevent** the splitting into 2 words
- Globbing
	- After splitting, Bash expands **wildcards** to matching file names in current directory
	- eg. 
		- `echo *` -> echo data.csv report.txt 
		- `rm *.txt` -> expands to `rm report.txt`
	- Use "" to **prevent** globbing

## Wildcards (see Globbing above)
- `*` — any string (including empty)
- `?` — any single character
- `[abc] / [a-z]` — any one listed/ranged character
- `[^a] or [!a]` — any one not in the set
# Arrays
```c
domains=(www.inlanefreight.com ftp.inlanefreight.com vpn.inlanefreight.com www2.inlanefreight.com)

echo ${domains[0]}
```

Note: if use " ", then the separation by space will be ignored.

## Special variables

| Variable     | Description                                           |
| ------------ | ----------------------------------------------------- |
| `$#`         | Number of arguments passed to the script.             |
| `$@`         | Used to retrieve the list of command-line arguments.  |
| `$n` (eg. 1) | n'th argument passed to script                        |
| `$$`         | PID of the currently executing process.               |
| `$?`         | Exit status of the script. 0 is success, 1 is failure |

# Conditionals
```c
if [ $# -eq 0 ]
then 
	echo ...
	echo ...
elif [ ... ]
then
	...
fi
```
Note:
- spacing inside `[ ] (after and before the [ ])
- if then ->elif then OR else then -> fi
- can use ; instead of newlines
	- `if [ ... ]; then echo "hi"; fi`

# Comparisons

## Strings

| **Operator** | **Description**                             |
| ------------ | ------------------------------------------- |
| `==`         | is equal to                                 |
| `!=`         | is not equal to                             |
| `<`          | is less than in ASCII alphabetical order    |
| `>`          | is greater than in ASCII alphabetical order |
| `-z`         | if the string is empty (null)               |
| `-n`         | if the string is not null                   |

Note: when comparing with variables, need put it in double quotes, telling Bash that content of variable should be handled as string
- eg. `if [ "$1" == "Hello" ]`
- More precisely, quotes make it such that it doesn't break if it's empty or has spaces (turns into multiple arguments) or contains globs (`* ?  [ ]`) which can expand to filesnames

eg. `if [[ ! -z "$salt" ]]`
- if $salt variable is not empty

String comparisons (< , >) only work within double square brackets `[[ ]]`

String length:
- `${#variable}
- Also can count with 
	- `count=$(echo $var | wc -m)`
		- `wc -m`: number of characters
		- BUT THIS IS OFF BY ONE! Since it will count the `\n` by echo. Need to subtract one

Searching for string within a string:
- `[[ $haystack == *"$needle"* ]]`
	- * as wildcard

Printing last 20 characters of a tring
- `echo $var | tail -c 20`
	- piping: echo feeds its output to tail, then tail writes it's stdout to terminal.


## Integers

| **Operator** | **Description**             |
| ------------ | --------------------------- |
| `-eq`        | is equal to                 |
| `-ne`        | is not equal to             |
| `-lt`        | is less than                |
| `-le`        | is less than or equal to    |
| `-gt`        | is greater than             |
| `-ge`        | is greater than or equal to |
- be careful not to use things like < here!

# File operators

|**Operator**|**Description**|
|---|---|
|`-e`|if the file exist|
|`-f`|tests if it is a file|
|`-d`|tests if it is a directory|
|`-L`|tests if it is if a symbolic link|
|`-N`|checks if the file was modified after it was last read|
|`-O`|if the current user owns the file|
|`-G`|if the file’s group id matches the current user’s|
|`-s`|tests if the file has a size greater than 0|
|`-r`|tests if the file has read permission|
|`-w`|tests if the file has write permission|
|`-x`|tests if the file has execute permission|
eg. `if [ -e "$1" ]`

## Boolean/logical operators
- Can compare strings with `[[ condition ]]`
	- eg. `if [[ $# > 1 ]]`

|**Operator**|**Description**|
|---|---|
|`!`|logical negotation NOT|
|`&&`|logical AND|
|`\|`|logical OR|


# Arithmetic

## Arithmetic operations

| **Operator** | **Description**                         |
| ------------ | --------------------------------------- |
| `+`          | Addition                                |
| `-`          | Subtraction                             |
| `*`          | Multiplication                          |
| `/`          | Division                                |
| `%`          | Modulus                                 |
| `variable++` | Increase the value of the variable by 1 |
| `variable--` | Decrease the value of the variable by 1 |

`(( ))
- Evaluates a C-style arithmetic evaluation
- Returns exit status 0 (true) if result =/= 0 else 1 if result = 0 (false)
- Use as evaluation/test context
	- eg. `if (( i > 3 )); then echo "bigger'; fi`
		- if i=4, echoes "bigger"
	- eg. `(( i++ ))
		- If i do this without the brackets, bash will parse i++ as a command named i with arguments ++ instead of maths

`$(( ))`
- Essentially an inline calculator
- expansion to a string, use when want the value in output or to asign
- eg. `echo $((10+10))`
- eg. `sum=$((y+x))`

Variable length
- `echo ${#variable_name}


# Input Output
- `echo -e "Hello`
- `read -p "Selection your option: " opt`
	- `-p`: ensures input remains on the same line
	- `opt`: value stored in the variable opt

## Tee
- eg.
	- `netrange=$(whois $ip | grep "NetRange\|CIDR" | tee -a CIDR.txt)
		- Recall that this runs the entire command, then the `$()` sets the result of the stdout (of the final command) to the `netrange` variable
	- `tee
		- Writes **both** to standard output and to FILES


# Flow: Loops

## For loops
```c
for variable in 1 2 3 4
do
	echo $variable
done
```
- `for i in {1..40}` -> take note: 2 dots only
- `for variable in file1 file2 file3`
- `for ip in "10.10.10.170 10.10.10.174 10.10.10.175"`
- `for ip in $ipaddr`
	- eg. if $ipaddr outputs multiple arguments

Can do in a single line as well
- `for i in 1 2 3; do echo "hello"; done`

## While loops
```c
while [ <condition is true>  ]
do
	...
done
```
- Can also use `break` inside it, usually inside an `if-else` block

```bash
while <command; do ...; done
```

- Loops as long as the command before `do` succeeds (exit status 0)

## Until loops (just use while.)

```c
while [ <condition is false>  ]
do
	...
done
```

# Flow: Branches

## Case statements

```c
case <expression> in
	pattern_1 ) statements ;;
	pattern_2 ) statements ;;
	pattern_3 ) statements ;;
esac
```

- Replace `"1"` etc. with the values the var can take.
- `esac` is just case spelled backwards


# Functions

## Defining

```c
function name {
	...
}

# OR 

name() {
	...
}
```

## Calling 
- `name`
	- No need ()

## Parameter passing
- Same way we did it in scripts
- Within the function definition, refer to the arguments as `$1` `$2` etc.
```c
function print_pars {
	echo $1 $2 $3
}
```

- Then when calling the function, pass them in like this
	- `print_pars "$one" "$two" "$three"`

**IMPORTANT**
- ALL variables in bash are **global**
## Return values
- `$?` stores the return value of the most recently executed command or function/command substitution

eg.
```c

function given_args {
    if [ $# -lt 1 ]
    then
        echo -e "Number of arguments: $#"
        return 1
    else
        echo -e "Number of arguments: $#"
        return 0
    fi
}
```

So after calling this funcction, `$?` gets updated with either 0 or 1.
Can also pass the result of the function into a variable
- `content=$(given_args "argument")

For in-built processes, these are the return codes

|**Return Code**|**Description**|
|---|---|
|`1`|General errors|
|`2`|Misuse of shell builtins|
|`126`|Command invoked cannot execute|
|`127`|Command not found|
|`128`|Invalid argument to exit|
|`128+n`|Fatal error signal "`n`"|
|`130`|Script terminated by Control-C|
|`255\*`|Exit status out of range|
# Debugging
- `-v`
- `-x`
- eg. `bash -x -v CIDR.sh`