# Lab #2,22110078, Nguyen Tien Toan, INSE330380E_01FIE
# Task 1: Transfer files between computers  
**Question 1**: 
Conduct transfering a single plaintext file between 2 computers, 
Using openssl to implementing measures manually to ensure file integerity and authenticity at sending side, 
then veryfing at receiving side. 

**Answer 1**:
## 1. Build the containers by docker-compose
We will use this `docker-compose.yml` which is provided in security_labs:
![image](https://github.com/user-attachments/assets/48bd7839-a2ad-4b30-adfb-2862e53425db)

We will use `docker-compose up --build` to build the containers
![image](https://github.com/user-attachments/assets/ce0227cf-d017-470a-a671-271001535b57)

And here we get the containers
![image](https://github.com/user-attachments/assets/2aa6d847-4a02-433a-a034-47c5d0ec96a5)

## 2. Create a plaintext file 
We will let the container `alice-10.9.0.5` (Alice) be the sender and `bob-10.9.0.6` (Bob) be the receiver

We will use this code to access to the `alice-10.9.0.5` terminal:

```
docker exec -it alice-10.9.0.5 /bin/bash
```

![image](https://github.com/user-attachments/assets/49601466-1f61-48a0-9dcd-f0bbf9dd4c24)

Then we will use this code to create `file.txt` in `/home` of `alice-10.9.0.5`

```
echo "This is a test file for transfer." > /home/file.txt
```
And then check the exist of it by `cat`

![image](https://github.com/user-attachments/assets/a6427cec-1333-498c-aa6e-82d03702c13e)

So here we can see that we have successfully create a plaintext file

## 3. Encrypting the plaintext file
We will use the `aes-128-ecb` to encrypt the `file.txt`

And we will encrypt it by the key which already known by `alice` and `bob`, and because we use `aes-128-ecb` so the key need to be 16 bytes

The key: `00112233445566778899aabbccddeeff`

So to encrypt the `file.txt`, alice will use this code

```
openssl aes-128-ecb -e -in /home/file.txt -out file-encrypt.enc -K 00112233445566778899aabbccddeeff
```
With:
- `-e`: Encrypt the file
- `-in /home/file.txt:` The plaintext file to encrypt
- `-out /home/file-encrypt.enc`: The output encrypted file
- `-K 00112233445566778899aabbccddeeff`: The 128-bit key in hexadecimal

![image](https://github.com/user-attachments/assets/f1645b11-a2ba-46ac-886c-d3109ece3118)

And here, we can check `file-encrypt.enc` by `cat` too

![image](https://github.com/user-attachments/assets/d6f6729b-4731-4e6c-a097-a715bd43fccf)

## 4. Generate HMAC for Integrity Check
To ensure the integrity and authenticity of the file, we will create an HMAC (using a shared secret key). 

The shared secret key is already known by Alice and Bob

The HMAC will allow the receiver to verify if the file has been tampered with during transmission

So on both container, we need to create a `secret.key`

On Alice container:
```
echo -n "shared_secret_key" > /home/secret.key
openssl dgst -sha256 -hmac "$(cat /home/secret.key)" -out /home/file.hmac /home/file-encrypt.enc
```
With:
- `-sha256`: Using SHA-256 for HMAC
- `-hmac "$(cat /home/secret.key)"`: Use the shared secret key to create the HMAC
- `-out /home/file.hmac`: The HMAC file for the encrypted file

![image](https://github.com/user-attachments/assets/af281df1-e87e-4e41-bc66-fb0a8d13b8bd)

On Bob container:
```
echo -n "shared_secret_key" > /home/secret.key
```

![image](https://github.com/user-attachments/assets/41ad4dec-a5a6-4ab6-9b94-ed57133df714)

## 5. Install and run the SSH service on Bob container
For Alice transfer to Bob, we will have to install and start the SSH on Bob container

We will use this code to access to `bob-10.9.0.6` terminal
```
docker exec -it bob-10.9.0.6 /bin/bash
```
![image](https://github.com/user-attachments/assets/954a9d40-e1be-42a9-a93b-4e6e8b7c53e1)


To install OpenSSH server
```
apt install -y openssh-server
```
![image](https://github.com/user-attachments/assets/cdf8999e-e41d-4be9-a0f1-88a6a7305c52)

Start the SSH server
```
service ssh start
```

![image](https://github.com/user-attachments/assets/462c86f9-8124-4ebd-b13e-9d86a6f07f55)


Then we check if SSH is running
```
netstat -tuln | grep :22
```
![image](https://github.com/user-attachments/assets/50faf899-b8e4-4cc4-9e42-87a313e1d197)

We can see that the SSH service is listening on port 22

Finally, we set up the Bob user:
```
useradd -m -s /bin/bash bob
passwd bob
```

![image](https://github.com/user-attachments/assets/2435a29c-6ae6-4fda-b604-6f3b6acaa400)


## 6. Transfer file using SCP
Now, in Alice container, we will transfer file to Bob
```
scp /home/file-encrypt.enc /home/file.hmac bob@10.9.0.6:/home
```

![image](https://github.com/user-attachments/assets/cbcbd7ca-41ff-415a-a259-e0bd2f64a2c5)

## 7. Check result in Bob container
On Bob container, we can see that the file is already transfered

![image](https://github.com/user-attachments/assets/4a3eaa79-4e25-4570-a25b-a67de9b634d4)

We can decrypt the encrypted file by `aes-128-ecb` with the key which Bob has already known: `00112233445566778899aabbccddeeff` 

```
openssl aes-128-ecb -d -in /home/file-encrypt.enc -out /home/decrypted_file.txt -K 00112233445566778899aabbccddeeff
```
![image](https://github.com/user-attachments/assets/a99e2b38-b797-4412-b9a4-8c3baf6df75e)

And we can see that the `decrypted_file.txt` is similar to `file.txt` which is created by Alice

We will recompute the HMAC for the received file using the same shared secret key that Alice used
This ensures that the file has not been altered

```
openssl dgst -sha256 -hmac "$(cat /home/secret.key)" -out /home/recomputed.hmac /home/file-encrypt.enc
```

![image](https://github.com/user-attachments/assets/478dffeb-b5ed-4867-99b8-3070d42a5667)

Then we will compare the HMACs. If the HMACs match, the file is intact and authentic

If the HMACs do not match, it indicates the file was tampered with

```
cmp /home/recomputed.hmac /home/file.hmac
```

![image](https://github.com/user-attachments/assets/cb27feea-7b27-487b-af49-f60d6953af40)

So here we can see that the HMACs match which mean the file's integrity and authenticity are verified

# Task 2: Transfering encrypted file and decrypt it with hybrid encryption. 
**Question 1**:
Conduct transfering a file (deliberately choosen by you) between 2 computers. 
The file is symmetrically encrypted/decrypted by exchanging secret key which is encrypted using RSA. 
All steps are made manually with openssl at the terminal of each computer.

**Answer 1**:

![image](https://github.com/user-attachments/assets/8e5c054c-2c36-4c95-a766-6ceec210a6ee)

![image](https://github.com/user-attachments/assets/3b5f53f4-1278-4218-b392-03268ac894df)

Like these pictures, we will generate key pairs on Bob and then send Bob's public key to Alice. 

Next Alice will create a secret key (session key) to encrypt the file, then Alice use Bob's public key to encrypt the secret key. After that Alice will send the encrypted file and encrypted secret key to Bob

And Bob will use his private key to decrypt the encrypted secret key which send by Alice. Finally, Bob will use the secret key (after decrypt) to decrypt the encrypted file

## 1.Generate RSA Key Pairs on Bob container
First, we will generate the RSA private key by  this code
```
openssl genpkey -algorithm RSA -out bob_private_key.pem -aes256
```
With:
- `openssl genpkey`: Generates a private key
- `-algorithm RSA`: Specifies the RSA encryption algorithm
- `-out bob_private_key.pem`: Saves the generated private key to a file named bob_private_key.pem
- `-aes256`: Encrypts the private key file using AES-256. A passphrase will be requested to secure it

And the passphrase here: `1234`

![image](https://github.com/user-attachments/assets/9522d368-d564-4c2d-95e6-b3edd93b367e)

Then we extract the public key from the private key
```
openssl rsa -pubout -in bob_private_key.pem -out bob_public_key.pem
```

With:
- `openssl rsa`: Handles RSA keys
- `-pubout`: Extracts the public key from the private key

We can use `cat` to see the public key

![image](https://github.com/user-attachments/assets/f4928cf4-50b1-46c9-a267-d9ea80c0d5a1)

## 2. Install and run the SSH service on Alice container
The same for task 1, in order for Bob to send his public key to Alice, it is necessary to install and start SSH service on Alice container

To install OpenSSH server
```
apt install -y openssh-server
```
![image](https://github.com/user-attachments/assets/434ae6e3-eb3e-46b7-8314-f01bcef26d70)

Start the SSH server
```
service ssh start
```
![image](https://github.com/user-attachments/assets/598e6251-ad94-4fca-a31a-bcc6cc24d06a)

Then we check if SSH is running
```
netstat -tuln | grep :22
```
![image](https://github.com/user-attachments/assets/ebc085db-2a63-4330-87c0-e56343908e31)

We can see that the SSH service is listening on port 22

Finally, we set up the Alice user:
```
useradd -m -s /bin/bash alice
passwd alice
```
![image](https://github.com/user-attachments/assets/e5043ac4-cdce-4c87-8ebe-369fe589f7c0)

## 3. Transfer Bob's public key to Alice
We can use `scp` to transfer it
```
scp bob_public_key.pem alice@10.9.0.5:/home
```
![image](https://github.com/user-attachments/assets/a885c082-81ed-4e5e-a181-954435cb77f5)

## 4. Create a secret key (on Alice container)
We will generate a secret key for encrypting the `file.txt` (which was created in task 1)
```
openssl rand -base64 16 > secret.key
```
With:
- `rand`: Generates random data
- `-base64`: Encodes the output in Base64 for easier handling
- `16`: Specifies the number of random bytes to generate

Because later we will use AES-128-ECB to encrypt `file.txt` so here `secret.key` will have 16 bytes

![image](https://github.com/user-attachments/assets/fe83dd3e-d6bd-4575-8fb5-88b55f853c2d)

And of course we can use `cat` to see `secret.key`

## 5. Encrypt the file by secret key (on Alice container)
We will use this code to encrypt `file.txt`
```
openssl enc -aes-128-ecb -in file.txt -out encrypted_file.enc -pass file:secret.key
```
With:
- `-pass file:secret.key`: Reads the encryption key from the file secret.key.

![image](https://github.com/user-attachments/assets/2d54bc39-e791-4764-98f5-d05b89a2d499)

We can use `ls` to check that `encrypted_file.enc` is generated

## 6. Encrypt the secret key by Bob's public key (on Alice container)
First, we need to check if there are Bob's public key by `ls`

![image](https://github.com/user-attachments/assets/a079bf81-a9ec-4127-ad50-6977ce028480)

After that, we encrypt `secret.key` by Bob's public key
```
openssl rsautl -encrypt -inkey bob_public_key.pem -pubin -in secret.key -out secret_key_encrypted.bin
```
With:
- `openssl rsautl`: A tool for RSA-based encryption and decryption
- `-encrypt`: Indicates that the operation is encryption
- `-inkey bob_public_key.pem`: Specifies Bob’s public key file as the encryption key
- `-pubin`: Tells OpenSSL that the input key is a public key (not private)
- `-in secret.key`: Specifies the secret key file
- `-out secret_key_encrypted.bin`: Saves the encrypted file in binary format

Check that the encrypted file, `secret_key_encrypted.bin`, was created successfully
```
ls -l secret_key_encrypted.bin
```
![image](https://github.com/user-attachments/assets/27965345-e0a1-40d8-887a-165aa3b8148c)

## 7. Transfer the encrypted file and encrypted secret key to Bob
Like the public key, we can use `scp` on Alice container to transfer the encrypted file and the encrypted secret key to Bob container
```
scp secret_key_encrypted.bin encrypted_file.enc bob@10.9.0.6:/home
```
![image](https://github.com/user-attachments/assets/531da21d-89ea-47e9-ab66-5af08fb29031)

## 8. Decrypt the encrypted secret key
On Bob container, we will use the private key to decrypt the encrypted secret key
```
openssl rsautl -decrypt -inkey bob_private_key.pem -in /home/secret_key_encrypted.bin -out /home/secret.key
```

With:
- `-decrypt`: Indicates that the operation is decryption.
- `-inkey bob_private_key.pem`: Specifies Bob’s private key file for decryption.

It will require us to enter the pass pharse which is `1234`

![image](https://github.com/user-attachments/assets/fe30d253-47a4-4347-8a59-2f0e0efdce83)

Then we can use `ls home` to check for the `secret.key` 

![image](https://github.com/user-attachments/assets/fc5f5098-0ca0-419a-8073-8b14f78ad249)

## 9. Decrypt the encrypted file
Then we will use AES-128-ECB with `secret.key` to decrypt the encrypted file
```
openssl enc -aes-128-ecb -d -in encrypted_file.enc -out decrypted_file.txt -pass file:secret.key
```

![image](https://github.com/user-attachments/assets/684429a1-9312-4aa1-9941-f2012034b6af)

And here the decrypted file is similar to `file.txt` => Success.

# Task 3: Firewall configuration
**Question 1**:
From VMs of previous tasks, install iptables and configure one of the 2 VMs as a web and ssh server. Demonstrate your ability to block/unblock http, icmp, ssh requests from the other host.

**Answer 1**:
We will set up one VM (Bob) as a web and SSH server and demonstrate how to use iptables to block/unblock HTTP, ICMP, and SSH traffic from the other VM (Alice).

## 1. Set Up Bob as a Web and SSH Server
Access to Bob container
```
docker exec --privileged -it bob-10.9.0.6 /bin/bash
```
![image](https://github.com/user-attachments/assets/5498195e-e240-4745-b7d5-89c9d3770240)

We have to use `--privileged` because without the `--privileged` flag, the container doesn't have permission to modify iptables


Then we will install `apache2` (Web Server)
```
apt install apache2
```
![image](https://github.com/user-attachments/assets/fe68af9f-ec29-43ca-a23e-ce2486ccc38f)

About the SSH Server, we have installed it from previous tasks

Check for their status
```
service apache2 status
service ssh status
```
![image](https://github.com/user-attachments/assets/8a8a0753-663e-4b8d-b216-403f1b2f1135)

Test HTTP service: From Alice, use `curl` to test the web server
```
docker exec -it alice-10.9.0.5 curl http://10.9.0.6
```
With:
- `curl http://10.9.0.6`: Sends an HTTP GET request to Bob's IP address. If the web server is running, it will return the default Apache page

![image](https://github.com/user-attachments/assets/f9762a61-1577-4b8b-9c1f-1b258ca01333)

Test SSH access: From Alice, try SSH into Bob:
```
docker exec -it alice-10.9.0.5 ssh bob@10.9.0.6
```
With:
- `ssh bob@10.9.0.6`: Attempts to establish an SSH connection to Bob. If successful, we will be logged into Bob

![image](https://github.com/user-attachments/assets/2e5a20c0-0606-438f-a6ea-0ebe2a279add)

## 2. Install iptables on Bob
Install `iptables` inside Bob
```
apt-get install iptables -y
```
![image](https://github.com/user-attachments/assets/24998ab6-1675-4687-9b8a-c858bedd32aa)

Verify installation
```
iptables --version
```
![image](https://github.com/user-attachments/assets/ec0cd9d5-eb8e-4bde-9782-f0f4649025ce)

## 3. Block and Unblock Traffic Using iptables
We will demonstrate blocking and unblocking HTTP, ICMP, and SSH traffic
### 3.1 Block/Unblock HTTP (Port 80)
#### 1. Block HTTP traffic
Run the following command on Bob to block incoming HTTP traffic from Alice
```
iptables -A INPUT -p tcp --dport 80 -s 10.9.0.5 -j REJECT
```
With:
- `iptables -A INPUT`: Adds a rule to the INPUT chain, which processes incoming traffic
- `-p tcp`: Specifies the protocol (tcp) for the rule
- `--dport 80`: Specifies that the rule applies to traffic directed at port 80 (HTTP)
- `-s 10.9.0.5`: Limits the rule to traffic originating from Alice’s IP address
- `-j REJECT`: Specifies the action to take; REJECT drops the packet and sends an error message back to the source

![image](https://github.com/user-attachments/assets/4d794ab0-aac4-4de3-b1b4-03ca8d21bd6c)

To test HTTP block, from Alice, try to access the web server
```
docker exec -it alice-10.9.0.5 curl http://10.9.0.6
```
![image](https://github.com/user-attachments/assets/d6f2c02d-ed46-49d5-a8ea-bfd01a644e58)

We can see here, it fails because HTTP traffic is blocked

#### 2. Unblock HTTP traffic
To remove the rule and allow HTTP traffic again, run
```
iptables -D INPUT -p tcp --dport 80 -s 10.9.0.5 -j REJECT
```
With:
- `iptables -D INPUT`: Deletes the specified rule from the INPUT chain, restoring access

![image](https://github.com/user-attachments/assets/32cc57c4-fefd-43c0-a0c3-f5cada57bf29)

Test HTTP unblock
```
docker exec -it alice-10.9.0.5 curl http://10.9.0.6
```
![image](https://github.com/user-attachments/assets/8fc741e3-f255-4a5a-b92d-2c7cb90d2b27)

It works because we have unblocked the HTTP traffic

### 3.2 Block/Unblock SSH (Port 22)
#### 1. Block SSH traffic
Run the following command on Bob to block incoming SSH traffic from Alice
```
iptables -A INPUT -p tcp --dport 22 -s 10.9.0.5 -j REJECT
```
This rule works similarly to the HTTP block but targets port 22 (used by SSH)

![image](https://github.com/user-attachments/assets/db5f9968-a1a7-4215-82ea-77143e98ab2a)

Test SSH block, from Alice, try SSH into Bob
```
docker exec -it alice-10.9.0.5 ssh bob@10.9.0.6
```
![image](https://github.com/user-attachments/assets/e92b5c74-0ad6-407f-b66e-543d07300eef)

It fails because SSH traffic is blocked

#### 2. Unblock SSH traffic
To remove the rule and allow SSH traffic again, run
```
iptables -D INPUT -p tcp --dport 22 -s 10.9.0.5 -j REJECT
```
Delete the rule blocking SSH, allowing Alice to reconnect.

![image](https://github.com/user-attachments/assets/1727762c-348c-42d0-9440-0fcb10a9a1d3)

Test SSH unblock
```
docker exec -it alice-10.9.0.5 ssh bob@10.9.0.6
```
![image](https://github.com/user-attachments/assets/e730b5de-a107-482f-9e10-1cb96e7143d5)

It works because we have unblocked the SSH traffic

### 3.3 Block/Unblock ICMP (Ping)
#### 1. Block ICMP traffic
Run the following command on Bob to block ICMP traffic (ping) from Alice
```
iptables -A INPUT -p icmp -s 10.9.0.5 -j REJECT
```
With: 
- `-p icmp`: Specifies the protocol as ICMP (used for ping).
This blocks ping requests from Alice to Bob.

![image](https://github.com/user-attachments/assets/93112242-3b4c-4a04-b7bc-61aac71ec340)

Test ICMP block, from Alice, try pinging Bob
```
docker exec -it alice-10.9.0.5 ping -c 3 10.9.0.6
```
![image](https://github.com/user-attachments/assets/c908da5c-c831-446a-8868-c90a9876b949)

The ping fails because we have blocked ICMP traffic from Alice

#### 2. Unblock ICMP traffic
To remove the rule and allow ICMP traffic again, run
```
iptables -D INPUT -p icmp -s 10.9.0.5 -j REJECT
```
Deletes the ICMP rule, allowing ping requests

![image](https://github.com/user-attachments/assets/d75f1347-451a-478d-907a-1c6a6d63de48)

Test ICMP unblock
```
docker exec -it alice-10.9.0.5 ping -c 3 10.9.0.6
```
![image](https://github.com/user-attachments/assets/f59a71c4-eb9f-4dca-a536-c9ddc1f4a967)

It works because we have unblocked the ICMP traffic
