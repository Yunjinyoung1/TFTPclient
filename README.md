# TFTPclient
네트워크 기말과제
이 과제는 네트워크 프로그래밍 기말과제 입니다.

import socket
import struct

OPCODE_RRQ = 1
OPCODE_DATA = 3
OPCODE_ACK = 4
OPCODE_ERROR = 5

MODE_OCTET = b'octet'

ERROR_UNDEFINED = 0
ERROR_FILE_NOT_FOUND = 1

def create_rrq_packet(filename):
    # Create RRQ (Read Request) packet
    return struct.pack('!H', OPCODE_RRQ) + filename.encode() + b'\0' + MODE_OCTET + b'\0'

def create_ack_packet(block_number):
    # Create ACK (Acknowledgment) packet
    return struct.pack('!HH', OPCODE_ACK, block_number)

def parse_data_packet(data):
    # Parse DATA packet
    opcode, block_number = struct.unpack('!HH', data[:4])
    return opcode, block_number, data[4:]

def send_file(filename, server_ip, server_port):
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    client_socket.settimeout(5)  # Set timeout to 5 seconds

    try:
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

    except socket.error as e:
        print(f"Socket error: {e}")
    finally:
        client_socket.close()

if __name__ == "__main__":
    server_ip = '203.250.133.88'
    server_port = 69
    file_to_download = 'file_to_download.txt'  # Change this to the desired file name

    send_file(file_to_download, server_ip, server_port)
