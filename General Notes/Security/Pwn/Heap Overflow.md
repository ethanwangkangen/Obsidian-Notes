Allocating and Freeing dynamic memory:
```c
char* buff;
buf = malloc(50);
...
scanf("%s", buff); // Possible to overwrite!
free(buf);
```

2 Linked lists are stored
- Allocated memory list
	- Keeps track of allocated memory chunks
- Free list
	- Keeps track of unallocated memory chunks


Overflow possible
- eg. scanf above, might overwrite into the next chunk
	- Might overwrite data pointers that point to the next node in the linked list


Data oriented attacks
- Don't need to write to any control data
- Unlike ...
	- Don't need to write attack payload in memory
	- Don't need to have attack payload be execcutable
	- Don't need to divert control flow to payload
- Just manipulate non control data
- Don't even need to corrupt anything to be dangerous!
	- Leak something, eg. heartbleed attack in OpenSSL
		- Client: server, give me this 500 word message if you are server. 'bird'
		- Server: bird, alice wants to change password .....

```c
//set root privilege
seteuid(0);

//set normal user privilege
seteuid(pw->pw_uid); //Idea: corrupt pw_uid to 0 to remain as root
//execute user's command
```



