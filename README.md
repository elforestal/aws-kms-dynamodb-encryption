<img width="878" height="510" alt="image" src="https://github.com/user-attachments/assets/f1353711-026f-4582-9fde-ddb6a8618d3d" />

## Introducing Today's Project!

In this project, I will demonstrate how to protect sensitive data at rest in AWS by encrypting a DynamoDB table with a customer-managed key (CMK) in AWS Key Management Service (KMS). The goal is to enforce least-privilege access to encrypted data: 

I'll create and manage the encryption key, apply it to a DynamoDB table, then prove the control works by attempting to read the data as a separate IAM user who has full DynamoDB access but no permission to use the KMS key. That user is denied at the kms:Decrypt step, showing that data-layer access depends on key permissions, not just resource permissions. 

This is defense in depth — encryption at rest plus key-policy access control — which is a core requirement in secure cloud architecture and compliance frameworks.

### Tools and concepts

**Services used**
- 🔐 **AWS KMS** — created and managed a customer-managed encryption key (CMK)
- 🗄️ **Amazon DynamoDB** — encrypted at rest with the KMS key
- 👤 **AWS IAM** — managed users and permissions

**Key concepts**
- Encryption at rest
- Symmetric encryption
- Customer-managed keys vs. AWS-managed keys
- Key policies and how they control `kms:Decrypt` access
- Transparent data encryption
- **Defense in depth** — encryption and IAM as two independent layers, so access to a *resource* doesn't grant access to the *data* inside it

### Project reflection

2 Hours

This project reinforced how encryption and IAM operate as independent layers — access to a resource doesn't grant access to the data inside it. The most valuable part was the access-control test: denying the test user, reading the missing kms:Decrypt permission from the error, then granting exactly that permission through the key policy. That deny-diagnose-grant cycle mirrors how you'd troubleshoot and scope real least-privilege access in production.

---

## Encryption and KMS

Encryption is the process of converting readable data (plaintext) into an unreadable, scrambled format (ciphertext) using an algorithm, so that only someone with the correct key can reverse it back to its original form. Companies and developers do this to protect the confidentiality of sensitive data — customer records, credentials, financial and health information — so that even if an attacker gains access to the storage or intercepts the data, they can't read it without the key. It's also a common requirement for compliance frameworks like PCI DSS, HIPAA, and GDPR. Encryption keys are the secret values the algorithm uses to encrypt and decrypt data; whoever controls the key effectively controls access to the data, which is why protecting and scoping access to keys (as we're doing with KMS) is as important as the encryption itself.

AWS KMS (Key Management Service) is a managed service for creating, storing, and controlling the encryption keys used to protect data across AWS. Instead of handling raw key material myself, KMS acts as a secure, centralized vault: the keys are stored in hardware security modules (HSMs) and never leave KMS in plaintext, and I control access to them through key policies and IAM rather than by managing the keys directly. On top of secure storage, KMS integrates natively with other AWS services (like DynamoDB, S3, and EBS) so they can encrypt data at rest with a key I manage, and it logs every use of a key through CloudTrail — which gives the audit trail that compliance frameworks require. In short, KMS lets me enforce and prove who can access encrypted data, which is why it's central to secure cloud architecture.

Encryption keys are broadly categorized as symmetric or asymmetric. I set up a symmetric key because it uses a single key to both encrypt and decrypt the data, which is faster and more efficient for encrypting data at rest — exactly the use case here, where DynamoDB needs to continuously encrypt and decrypt table data behind the scenes. Asymmetric encryption uses a public/private key pair and is designed for scenarios where two parties need to exchange data securely without sharing a secret, like signing or encrypting data in transit. Since this is a single-account, at-rest encryption use case with no need to share a public key externally, a symmetric key is the right fit — it's also the AWS default for KMS because it matches the most common encryption workloads.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-kms_a2b3c4d5)

---

## Encrypting Data

DynamoDB is AWS's fully managed NoSQL database service, built for fast, consistent performance at any scale — but in this project what matters is that it's the resource I'm protecting: my KMS key will encrypt the data stored in the table at rest, so that access to the actual data depends on key permSecuring a DynamoDB table with a customer-managed AWS KMS key and testing least-privilege access control via key policies and IAM.issions, not just database permissions.

DynamoDB offers three encryption-at-rest options: an AWS-owned key (fully managed by AWS with no visibility or control), an AWS-managed key (managed by KMS, visible in your account but not directly controllable), and a customer-managed key (a CMK you create and control in KMS). I chose the customer-managed key because it's the only option that lets me define the key policy myself — controlling exactly which IAM principals can decrypt the data — which is what makes the access-control test in the next steps possible and reflects least-privilege in a real security setup.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-kms_q8r9s0t1)

---

## Data Visibility

A KMS key manages permissions through its key policy — a resource-based policy attached directly to the key that defines which IAM principals can use it and what actions they can perform, like kms:Encrypt, kms:Decrypt, or administrative operations. This is separate from a user's IAM permissions on other services: even a user with full DynamoDB access is blocked from the encrypted data unless the key policy (or a matching IAM grant) explicitly allows them to decrypt. That separation is what lets the key act as an independent, fine-grained access-control layer over the data itself.

The items are visible to me because, as the IAM Admin user, I have permission to use the KMS key that encrypts the table. The data is still encrypted at rest — DynamoDB just decrypts it automatically on my behalf because I'm an authorized key user, a feature called transparent data encryption. The encryption isn't blocking me; the key policy is granting me access. That distinction is the whole point of the next step, where a user with full DynamoDB access but no KMS key permission tries to read the same data and is denied at the decrypt stage.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-kms_c0d1e2f3)

---

## Denying Access

I configured a new IAM user with console access and attached the AmazonDynamoDBFullAccess policy, giving it full permissions over DynamoDB — but I deliberately granted no permissions to my KMS key. This separation is intentional: the user can fully interact with the database resource itself, but without kms:Decrypt on the key, it can't read the encrypted data inside it. That gap between resource access and data access is exactly what the next step tests.

When I tried to view the table's items as the test user, I got an access denied error citing a missing kms:Decrypt permission — instead of the data. Even with full DynamoDB access, the user couldn't read the encrypted contents because they weren't granted use of the KMS key, confirming the encryption and key policy were working exactly as intended.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-kms_w0x1y2z3)

---

## EXTRA: Granting Access

I edited the KMS key policy as the key administrator and added the test user as a key user, which grants the permissions needed to use the key for cryptographic operations like kms:Decrypt. Once that principal was included in the key policy, the user could decrypt and view the table's data — the same request that was denied before now succeeded, because access is controlled by the key policy itself.

I logged back in as the test user and reopened the DynamoDB table's items — the same action that returned a kms:Decrypt access denied error before. This time it succeeded and the data was visible, confirming that adding the user to the key policy granted the decrypt permission and that access is controlled at the key-policy layer.

Access controls like IAM policies and security groups govern access to the resource or service — who can reach the DynamoDB table and what actions they can take on it. But they don't protect the data itself; once someone is through that gate, they can read whatever they're allowed to. Encryption secures the data within the resource, so even if access controls are bypassed — a stolen disk, a leaked backup, intercepted traffic — the data stays unreadable without the decryption key. The two work as layers: only a user with both resource access and key permissions can actually see the data.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-kms_feffb2fb8)

---

---
