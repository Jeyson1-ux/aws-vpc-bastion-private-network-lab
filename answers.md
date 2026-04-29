# 📘 Concept Check Answers

## 1. In Task 2.1 you opened SSH from 0.0.0.0/0. In a real company, what would you restrict it to instead, and how?

In a real production environment, SSH access should never be open to `0.0.0.0/0` because that allows any IP address on the internet to attempt a connection. This increases the attack surface and exposes the instance to brute force attacks.

Instead, I would restrict SSH access in one of the following ways:

* Limit access to my own IP address using a rule like:

  ```
  MyIP/32
  ```
* Restrict access to a corporate VPN IP range.
* Remove SSH entirely and use AWS Systems Manager (SSM) Session Manager instead.

The most secure option is to avoid SSH completely and rely on SSM, which removes the need for open ports and key management.

---

## 2. You copied a private key onto the bastion in Task 3.2. Name two reasons this is a bad practice and one alternative.

Copying private keys onto servers is bad practice for several reasons:

1. If the bastion host is compromised, the attacker gains access to the private key and can move laterally to other systems.
2. Private keys stored on instances can be accidentally exposed through logs, backups, or misconfigurations.

A better alternative is:

* Use AWS SSM Session Manager, which eliminates the need for SSH keys entirely.
* Or use SSH agent forwarding instead of copying keys.

---

## 3. When would you choose a Gateway Endpoint vs an Interface Endpoint vs just using the NAT Gateway?

* **Gateway Endpoint**:
  Used for S3 and DynamoDB. It is free and allows private access without using the internet or NAT Gateway.

* **Interface Endpoint**:
  Used for services like SSM, EC2 API, and others. It creates an ENI inside the VPC and allows private communication with AWS services.

* **NAT Gateway**:
  Used when instances need general internet access (e.g., downloading packages or accessing external APIs).

In this project, I used:

* NAT Gateway for general internet access
* S3 Gateway Endpoint to reduce NAT usage and cost

---

## 4. VPC Peering vs Transit Gateway — when does Transit Gateway start to make sense?

VPC Peering is useful for connecting a small number of VPCs directly. However, it becomes difficult to manage as the number of VPCs increases because it requires a full mesh of connections.

Transit Gateway becomes useful when:

* There are many VPCs
* Centralized routing is needed
* You want a hub-and-spoke architecture

Transit Gateway simplifies network management and scales better for large environments.
