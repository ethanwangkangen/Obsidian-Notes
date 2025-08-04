**Netcat**
- nc {ip} {port}
Once connection esatblished, can send requests

GET / HTTP/1.1
Host: example.com


# **Python Socket programming**

##  TCP 

**TCP Server**
```python
# server.py
import socket

HOST = '127.0.0.1'  # localhost
PORT = 65432        # non-privileged port

# Create socket
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind((HOST, PORT))   # Bind to address and port
    s.listen()             # Start listening for connections
    
    conn, addr = s.accept()  # Accept client connection
    with conn:
        while True:
            data = conn.recv(1024)  # Receive data
            if not data:
                break
            conn.sendall(data)  # Echo back
```

AF_INET: IPv4
SOCK_STREAM: for TCP
SOCK_DGRAM: for UDP

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM / socket.SOCK_DGRAM) as s
```
- socket object as context manager
- this is same as doing
```python
	s = socket.socket(....)
	try: 
		...
	finally: 
		s.close() 
```

s is listening socket
- `.bind( (IP, PORT) ) # binds to IP and port on local machine`
- `.listen()`
- `.accept() # blocks until receives incoming connection, then returns conn, addr`

conn is connection socket
- `.recv(bufsize) # receives data up to bufsize bytes` 
- `.send(data) # sends data (bytes) to the connected client`
- `.sendall(data) # same as above, but blocks until everything sent`


**TCP Client**
```python
# client.py
import socket

HOST = '127.0.0.1'  # Server's IP
PORT = 65432        # Server's port

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as c:
    c.connect((HOST, PORT))  # Connect to server
    c.sendall(b"Hello, server!")  # Send message
    data = c.recv(1024)  # Receive response

print(f"Received from server: {data.decode()}")
```

c is client socket
- `.connect() # connects client socket to the server's address` 
- `.recv(bufsize) # receives data up to bufsize bytes` 
- `.send(data) # sends data (bytes) to the connected client`
- `.sendall(data) # same as above, but blocks until everything sent`
##  UDP

**UDP Server**
```python
import socket

HOST = '127.0.0.1'  # Listen on localhost
PORT = 65432

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    s.bind((HOST, PORT))
    print(f"UDP server listening on {HOST}:{PORT}")
    
    while True:
        data, addr = s.recvfrom(1024)  # Receive datagram
        s.sendto(data, addr)  # Echo back
```

s is listening socket
- `.bind() # same as above`
- `.recvfrom(bufsize) # returns (data, addr), addr = (id, port)`
- `.sendto(data, addr) 

```python
# udp_client.py
import socket

SERVER_IP = '127.0.0.1'
SERVER_PORT = 12345

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    message = "Hello, UDP server!"
    s.sendto(message.encode(), (SERVER_IP, SERVER_PORT))  # Send to server

    data, server = s.recvfrom(1024)  # Receive response
    print("Received from server:", data.decode())
```

c is client socket
- `recvfrom(bufsize) # see above` 
- `.sendto(data, addr) ` 

For UDP
- No prior connection needed.
- Server and client socket use the same functions
- **Packet metadata contains sender's addres!**

Server binds()
- This tells OS which IP and port to listen on. That port is reserved for receiving incoming UDP packets.
- OS delivers incoming UDP packets to that socket

Client doesn't need to explitly bind()
- When UDP socket sends data using sendto(), the OS automatically assigns a free local port number (ephemeral port) for the client socket (if not bound alr)
- Assigns local IP address to
- Then attaches the ephemeral port and IP as the source address on the outgoing packet


