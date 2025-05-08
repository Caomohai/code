# code
1
import socket


def format_request(request_line):
    parts = request_line.split()
    if len(parts) == 2:
        command = parts[0]
        key = parts[1]
        if command in ['READ', 'GET']:
            size = f"{7 + len(key):03d}"
            return f"{size}{'R' if command == 'READ' else 'G'} {key}"
    elif len(parts) == 3:
        command = parts[0]
        key = parts[1]
        value = parts[2]
        if command == 'PUT':
            size = 7 + len(key) + len(value)
            if size <= 999:
                size_str = f"{size:03d}"
                return f"{size_str}P {key} {value}"
    return None


def send_requests(server_host, server_port, request_file_path):
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
            client_socket.connect((server_host, server_port))
            with open(request_file_path, 'r') as request_file:
                for line in request_file:
                    line = line.strip()
                    request = format_request(line)
                    if request:
                        client_socket.sendall(request.encode())
                        response = client_socket.recv(1024).decode()
                        print(f"{line}: {response}")
    except Exception as e:
        print(f"Error sending requests: {e}")


if __name__ == "__main__":
    send_requests('localhost', 12345, 'requests.txt')
    



    import socket
import threading
import time
from collections import defaultdict

# 元组空间使用字典存储
tuple_space = {}
# 统计信息
client_count = 0
operation_count = 0
read_count = 0
get_count = 0
put_count = 0
error_count = 0


def process_request(request):
    global operation_count, read_count, get_count, put_count, error_count
    operation_count += 1
    command = request[3]
    key = request[5:]
    if command == 'P':
        value = request[request.index(' ', 5) + 1:]
    if command == 'R':
        read_count += 1
        if key in tuple_space:
            return f"OK ({key}, {tuple_space[key]}) read"
        else:
            error_count += 1
            return f"ERR {key} does not exist"
    elif command == 'G':
        get_count += 1
        if key in tuple_space:
            removed_value = tuple_space.pop(key)
            return f"OK ({key}, {removed_value}) removed"
        else:
            error_count += 1
            return f"ERR {key} does not exist"
    elif command == 'P':
        put_count += 1
        if key in tuple_space:
            error_count += 1
            return f"ERR {key} already exists"
        else:
            tuple_space[key] = value
            return f"OK ({key}, {value}) added"
    else:
        error_count += 1
        return "ERR invalid command"


def handle_client(client_socket):
    global client_count
    client_count += 1
    try:
        while True:
            request = client_socket.recv(1024).decode()
            if not request:
                break
            response = process_request(request)
            client_socket.sendall(response.encode())
    except Exception as e:
        print(f"Error handling client: {e}")
    finally:
        client_socket.close()


def monitor_tuple_space():
    while True:
        time.sleep(10)
        tuple_count = len(tuple_space)
        total_tuple_size = 0
        total_key_size = 0
        total_value_size = 0
        for key, value in tuple_space.items():
            total_tuple_size += len(key) + len(value)
            total_key_size += len(key)
            total_value_size += len(value)
        average_tuple_size = total_tuple_size / tuple_count if tuple_count > 0 else 0
        average_key_size = total_key_size / tuple_count if tuple_count > 0 else 0
        average_value_size = total_value_size / tuple_count if tuple_count > 0 else 0

        print("Tuple Space Summary:")
        print(f"Number of tuples: {tuple_count}")
        print(f"Average tuple size: {average_tuple_size}")
        print(f"Average key size: {average_key_size}")
        print(f"Average value size: {average_value_size}")
        print(f"Total number of clients: {client_count}")
        print(f"Total number of operations: {operation_count}")
        print(f"Total READs: {read_count}")
        print(f"Total GETs: {get_count}")
        print(f"Total PUTs: {put_count}")
        print(f"Total errors: {error_count}")


def start_server(port):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind(('localhost', port))
    server_socket.listen(5)
    print(f"Server started on port {port}")
    monitor_thread = threading.Thread(target=monitor_tuple_space)
    monitor_thread.daemon = True
    monitor_thread.start()
    while True:
        client_socket, client_address = server_socket.accept()
        client_handler = threading.Thread(target=handle_client, args=(client_socket,))
        client_handler.start()


if __name__ == "__main__":
    start_server(12345)
    