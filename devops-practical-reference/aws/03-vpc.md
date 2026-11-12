# AWS — VPC Public/Private Subnet Lab

Console workflow and SSH commands from Virtualization, Cloud, and AWS slides 54–62.

## 1. Create the VPC

In the AWS Console, open **VPC → Create VPC → VPC and more** and enter:

| Setting | Value |
|---|---|
| Name tag | `my-ditiss-lab` |
| IPv4 CIDR block | `172.20.0.0/16` |
| Availability Zones | `1` |
| Public subnets | `1` |
| Private subnets | `1` |
| Customize subnet CIDR blocks | Enabled |
| Public subnet CIDR | `172.20.0.0/17` |
| Private subnet CIDR | `172.20.128.0/17` |
| NAT Gateway | Zonal, in 1 AZ |
| VPC endpoints | None |
| DNS options | Enable both boxes |

Create the VPC.

> The problem statement on slide 55 says `172.20.0.0/16` for the public subnet, which overlaps with the VPC CIDR and is not a valid subnet allocation. The detailed setup on slide 58 gives the non-overlapping public `/17` and private `/17` ranges above; use those detailed values.

## 2. Launch the public EC2 instance

Open **EC2 → Launch instance** and configure:

| Setting | Value |
|---|---|
| AMI | Amazon Linux |
| Name | `My-EC2-public-instance` |
| Key pair | Reuse or create a lab key pair |
| VPC | `my-ditiss-lab` |
| Subnet | Public subnet |
| Auto-assign public IP | Enable |

Configure the security group for the lab. At minimum, allow SSH from your own public IP. Add any required service ports only from the intended source ranges. Launch the instance.

## 3. Launch the private EC2 instance

Launch a second Amazon Linux instance:

| Setting | Value |
|---|---|
| AMI | Amazon Linux |
| Name | `My-EC2-private-instance` |
| Key pair | Reuse the same key pair |
| VPC | `my-ditiss-lab` |
| Subnet | Private subnet |
| Auto-assign public IP | Disable |

Configure the private instance's security group to allow SSH from the public instance's security group and HTTP as required by the task. Launch the instance.

## 4. Connect to the public instance

On your Linux client, set the key permission:

```bash
chmod 400 <KEY_PAIR_FILE.pem>
```

Connect:

```bash
ssh -i <KEY_PAIR_FILE.pem> ec2-user@<PUBLIC_EC2_IP>
```

## 5. Copy the key to the public instance and connect to the private instance

The slide's Windows workflow converts a PuTTY `.ppk` into an OpenSSH `.pem`, then uses WinSCP to copy it to the public EC2 instance. On the public EC2 instance, run:

```bash
chmod 400 <KEY_PAIR_FILE.pem>
ssh -i <KEY_PAIR_FILE.pem> ec2-user@<PRIVATE_EC2_PRIVATE_IP>
```

Keep the private key readable only by the account that needs it and remove the copied key from the public instance after the lab.

## 6. Install Apache on the private EC2 instance

After SSH into the private instance:

```bash
sudo yum install httpd -y
sudo service httpd start
sudo chkconfig httpd on
curl localhost
```

Only if package installation fails because of DNS resolution, the slide shows:

```bash
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
```

From the public instance, test the private web server:

```bash
curl http://<PRIVATE_EC2_PRIVATE_IP>
```

## 7. Clean up the VPC lab

Delete all lab resources:

1. Stop or terminate both EC2 instances.
2. Delete the lab subnets.
3. Delete lab network interfaces that remain.
4. Delete the Internet Gateway.
5. Delete the NAT Gateway and release its Elastic IP.
6. Delete the VPC.

Do not leave a NAT Gateway or allocated Elastic IP running after the practical; they can incur charges.

Source reference: Virtualization, Cloud, and AWS slides 54–62.
