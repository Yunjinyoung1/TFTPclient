# TFTPclient
네트워크 기말과제
이 과제는 네트워크 프로그래밍 기말과제 입니다.

import socket
import struct
import sys

# TFTP opcode
OPCODE_RRQ = 1
OPCODE_WRQ = 2
OPCODE_DATA = 3
OPCODE_ACK = 4
OPCODE_ERROR = 5

# TFTP modes
MODE_OCTET = b'octet'

# TFTP error codes
ERROR_FILE_NOT_FOUND = 1
ERROR_ACCESS_VIOLATION = 2
ERROR_DISK_FULL = 3
# Add more error codes as needed

def create_rrq_packet(filename):
    # Create RRQ (Read Request) packet
    return struct.pack('!H', OPCODE_RRQ) + filename.encode() + b'\0' + MODE_OCTET + b'\0'

def create_wrq_packet(filename):
    # Create WRQ (Write Request) packet
    return struct.pack('!H', OPCODE_WRQ) + filename.encode() + b'\0' + MODE_OCTET + b'\0'

def create_ack_packet(block_number):
    # Create ACK (Acknowledgment) packet
    return struct.pack('!HH', OPCODE_ACK, block_number)

def parse_data_packet(data):
    # Parse DATA packet
    opcode, block_number = struct.unpack('!HH', data[:4])
    return opcode, block_number, data[4:]

def send_file(filename, server_ip, server_port, mode, client_port=0):
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    client_socket.bind(('', client_port))
    client_socket.settimeout(5)  # Set timeout to 5 seconds

    try:
        if mode == 'get':
            # Send RRQ packet
            rrq_packet = create_rrq_packet(filename)
            client_socket.sendto(rrq_packet, (server_ip, server_port))

            block_number = 1
            received_data = b''

            while True:
                # Receive DATA packet
                data, server_address = client_socket.recvfrom(516)  # Maximum TFTP packet size
                opcode, received_block_number, payload = parse_data_packet(data)

                if opcode == OPCODE_DATA and received_block_number == block_number:
                    received_data += payload

                    # Send ACK packet
                    ack_packet = create_ack_packet(block_number)
                    client_socket.sendto(ack_packet, server_address)

                    if len(payload) < 512:
                        # Last block received
                        break
                    block_number += 1
                else:
                    # Invalid packet received, handle error or break the loop
                    break

            # Write received file content to a local file
            with open(filename, 'wb') as file:
                file.write(received_data)

        elif mode == 'put':
            # Check if the file exists
            if not os.path.exists(filename):
                print(f"Error: File '{filename}' does not exist.")
                return

            # Send WRQ packet
            wrq_packet = create_wrq_packet(filename)
            client_socket.sendto(wrq_packet, (server_ip, server_port))

            block_number = 1
            with open(filename, 'rb') as file:
                while True:
                    data_block = file.read(512)
                    if not data_block:
                        break

                    # Prepare DATA packet
                    data_packet = struct.pack('!HH', OPCODE_DATA, block_number) + data_block

                    # Send DATA packet
                    client_socket.sendto(data_packet, (server_ip, server_port))

                    # Wait for ACK packet
                    try:
                        ack, server_address = client_socket.recvfrom(4)
                        opcode, received_block = struct.unpack('!HH', ack)
                        if opcode != OPCODE_ACK or received_block != block_number:
                            # Invalid ACK received, handle error or break the loop
                            break
                    except socket.timeout:
                        # Handle timeout (no ACK received)
                        print(f"Timeout: No acknowledgment received for block {block_number}.")
                        break

                    block_number += 1

    except socket.error as e:
        print(f"Socket error: {e}")
    finally:
        client_socket.close()

def main():
    if len(sys.argv) < 4 or len(sys.argv) > 6:
        print("Usage: mytftp host_address [get|put] filename [-p port_number]")
        return

    server_ip = sys.argv[1]
    mode = sys.argv[2]
    filename = sys.argv[3]
    server_port = 69  # Default TFTP port

    if '-p' in sys.argv:
        port_index = sys.argv.index('-p')
        try:
            server_port = int(sys.argv[port_index + 1])
        except (ValueError, IndexError):
            print("Invalid port number.")
            return

    send_file(filename, server_ip, server_port, mode)

if __name__ == "__main__":
    main()
