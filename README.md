# Xiaomi-BLE-Protocol-Reverse-Engineering

Reverse Engineering the Bluetooth Communication Protocol of Xiaomi IoT Devices

## Bluetooth Services and Characteristics
The Xiaomi IoT devices communicate using various Bluetooth services and characteristics. Here is a detailed breakdown:

**Services**
Service 0x1801: `00001801-0000-1000-8000-00805f9b34fb`
Service 0x1800: `00001800-0000-1000-8000-00805f9b34fb`
Service 0xfe95: `0000fe95-0000-1000-8000-00805f9b34fb`
**Characteristics and Descriptors**
Characteristic 0x2a05: `00002a05-0000-1000-8000-00805f9b34fb` (Info)
Descriptor 0x2902: `00002902-0000-1000-8000-00805f9b34fb`
Characteristic 0x2a00: `00002a00-0000-1000-8000-00805f9b34fb` (Read)
Characteristic 0x2a01: `00002a01-0000-1000-8000-00805f9b34fb` (Read)
Characteristic 0x2aa6: `00002aa6-0000-1000-8000-00805f9b34fb` (Read)
Characteristic 0x0004: `00000004-0000-1000-8000-00805f9b34fb` (Read)
Characteristic 0x0005: `00000005-0000-1000-8000-00805f9b34fb` (Notify, Read)
Characteristic 0x0010: `00000010-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)
Characteristic 0x0016: `00000016-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)
Characteristic 0x0017: `00000017-0000-1000-8000-00805f9b34fb` (Notify, Write with reply)
Characteristic 0x0018: `00000018-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)
Characteristic 0x001a: `0000001a-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)
Characteristic 0x001b: `0000001b-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)
Characteristic 0x001c: `0000001c-0000-1000-8000-00805f9b34fb` (Notify, Write with no reply)

## Key Exchange and Session Key Generation for Xiaomi IoT Devices

The process of secure communication with Xiaomi IoT devices involves a series of steps to establish a shared session key and ensure encrypted data exchange. Below, I will elaborate on each step involved in the key exchange and session key generation process.

#### Step 1: Generating the Key Pair

The initial step is to generate a key pair using the elliptic curve cryptography method known as EC-secp256r1 (also called NIST P-256). This is a commonly used elliptic curve that provides robust security.

- **Generate a Key Pair**: Create a pair of keys: a public key and a private key. For the purposes of this explanation, we will refer to these keys collectively as the "Client Key Pair." The public key will be used to initiate secure communication with the device, while the private key remains confidential and is used in subsequent cryptographic operations.

#### Step 2: Sending the Public Key to the Device

Next, you need to send the public key from the client key pair to the Xiaomi device.

- **Prepare the Public Key**: The public key generated from the elliptic curve cryptography process typically starts with a prefix byte `0x04`. Before sending it to the device, this prefix should be removed. The resulting 64-byte public key can then be sent to the device.
- **Use Characteristic 0016**: Transmit the prepared 64-byte public key to the device using the Bluetooth characteristic identified as `0016`.

#### Step 3: Receiving the Device’s Public Key

The Xiaomi device will respond by sending its own public key back to the client.

- **Reconstruct the Device’s Public Key**: The public key received from the device will be 64 bytes long. To reconstruct the full public key, prepend the prefix byte `0x04` to this 64-byte key. This reconstructed key, now 65 bytes long, will be referred to as the "Device Public Key."

#### Step 4: Creating the Shared Secret Key

With both the client's private key and the device's public key available, the next step is to generate a shared secret key.

- **Generate the Shared Secret Key**: Using the Elliptic Curve Diffie-Hellman (ECDH) algorithm, combine the client’s private key (32 bytes) with the device’s public key (65 bytes) to produce a 32-byte shared secret key. This shared secret key will be referred to as the "Shared Secret."

#### Step 5: Combining with the Token Value

To further secure the communication, a token value obtained from an online source is combined with the shared secret.

- **Obtain the Token Value**: The token value can be 16, 24, or 32 bytes in length and is necessary for the authentication process. This token value will be referred to as the "Authentication Token."
- **Combine the Keys**: Concatenate the shared secret key with the authentication token to form a new composite key. This combined key will be referred to as the "Composite Key."

#### Step 6: Generating the Session Key

The composite key is used to generate a session key using the HMAC-based Extract-and-Expand Key Derivation Function (HKDF) with HMAC-SHA256.

- **Specify the Salt and Info**: For the HKDF process, use the string "miot-mesh-login-salt" as the salt and "miot-mesh-login-info" as the info.
- **Derive the Session Key**: The HKDF process produces a 64-byte session key, which will be referred to as the "Session Key."

#### Step 7: Breaking Down the Session Key

The session key is divided into several parts, each serving a different purpose:

1. **Decryption Key**: The first 16 bytes of the session key are used as the decryption key. This will be referred to as the "Decryption Key."
2. **Encryption Key**: The next 16 bytes of the session key are used as the encryption key. This will be referred to as the "Encryption Key."
3. **Decryption Nonce**: The subsequent 4 bytes are used as the decryption nonce. This will be referred to as the "Decryption Nonce."
4. **Encryption Nonce**: The next 4 bytes are used as the encryption nonce. This will be referred to as the "Encryption Nonce."

### Authentication Process

Once the session key has been established, the authentication process can begin. This process involves several steps to ensure that both the client and the device can trust each other.

1. **Calculate the CRC32 Hash**: Compute the CRC32 hash of the device’s public key.
2. **Encrypt the Device Public Key**: Encrypt the device’s public key using AES-CCM-128. The encryption key for this process is the latter 32 bytes of the session key, and the nonce used is a fixed value, `1112131415161718191a1b`.
3. **Send the Encrypted Key**: Transmit the encrypted device public key through the Bluetooth characteristic identified as `0016`.
4. **Verify Authentication**: If the device responds positively to this transmission, the authentication process is considered successful.

### Encrypting Messages to the Device

After successful authentication, messages sent to the device must be encrypted to ensure confidentiality.

1. **Encrypt the Message**: Use the AES-CCM-128 algorithm with the encryption key derived from the session key.
2. **Form the Nonce**: The nonce for this encryption process is a 12-byte value consisting of the encryption nonce (4 bytes) followed by `0000` (4 bytes) and a sequential message counter (4 bytes, starting from 0).
3. **Transmit the Encrypted Message**: Send the encrypted message using the Bluetooth characteristic identified as `001a`.

### Decrypting Messages from the Device

Messages received from the device are also encrypted and must be decrypted.

1. **Decrypt the Message**: Use the AES-CCM-128 algorithm with the decryption key derived from the session key.
2. **Form the Nonce**: The nonce for this decryption process is a 12-byte value consisting of the decryption nonce (4 bytes) followed by `0000` (4 bytes) and a sequential message counter (4 bytes).
3. **Receive and Decrypt the Message**: Receive the encrypted message via the Bluetooth characteristic identified as `0016` and decrypt it accordingly.
