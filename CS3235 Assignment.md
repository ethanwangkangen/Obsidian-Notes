
# mixed_codebase.rs

This is the starting point.

pub struct EnhancedStudentDatabase
- `rust database Box<UserDatabase>
- `c wrapped database DatabaseExtensions`


*Main function*
Signing up
- enqueue users 
	- push UserInfo structs with information into self.pending_requests

Logging in
- Take the password, cut off up to MAX_PASSWORD_LENGTH-1
- call *login_user()* on username and password
- if user not found in Rust database,
	- assume it is in C database
	- checks user_references, if found a match for username and password
		- set inactivity count to 0 and is_active to 1
		- call *create_session_for_c_ptr()* from c_extensions to get a session_token
			- in turn calls *create_user_session* from C 
	- get the user from c_extensions, *get_user_in_c_backend()*
	- push a new user into user_references
	- get the password from c_extentions, *get_user_password()*
	- if password correct,
		- return c_extensions, *login_user()*
- if user found in Rust database,
	- get the user from Rust database's *find_user_by_username()*
	- if password correct,
		- call *create_session()* from c_extensions to get a session token
		- call *update_user_session_token()* 
			- updates the session token for Rust database
		- call *activate_user()*
			- which calls *user_login()* on the Rust database

End of day
-  call *increase_day()* on Enhanced Database
	- call *sync_database*()
		- for entries in pending_requests, call *add_user_with_sync*
			- if overloaded, 
				- c_extensions, *sync_user_to_c_backend()
				- push id into c allocated_users
			- create user and add to rust database
	- increment own day counter
	- update day counter in C
	- validate active user sessions
	- updat rust database with *update_database_daily()
	- c_extensions *increment_day()*

Questions
- what is user_references
- create_session_for_c_ptr??
- what is session token and manager


# database_enhanced.c

Global states
- SessionManager_t* global_session_manager
- UserDatabase_t* global_db
- int * global_day_counter

Session manager
- Stores SessionInfo pointers
- SessionInfo
	- username, user id, session token, session idle time, is_active
- 