# eeeeeee



import socket

role = input("Host or client? ").lower()

if role == "host":
    port = int(input("Port: "))
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(("0.0.0.0", port))
    server.listen(1)

    print("Waiting for client...")
    conn, addr = server.accept()
    print("Connected to", addr)

    conn.sendall("Hello from host\n".encode())
    message = conn.recv(1024).decode()
    print("Client says:", message)

    conn.close()
    server.close()

elif role == "client":
    ip = input("Host IP: ")
    port = int(input("Port: "))

    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client.connect((ip, port))

    message = client.recv(1024).decode()
    print("Host says:", message)

    client.sendall("Hello from client\n".encode())
    client.close()
