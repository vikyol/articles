## AWS KMS Governance

The goal of KMS key governance is to ensure that KMS keys are used appropriately and securely to protect sensitive data. KMS key governance includes several essential components, such as:

1. Key creation and rotation policies: Establish policies for creating and rotating KMS keys, such as requirements for key strength and frequency of rotation.
2. Key usage policies: Defining policies for how KMS keys should be used, such as which AWS services are allowed to access the keys, which IAM roles or users can use the keys, and under what circumstances keys should be used.
3. Key retention and deletion policies: Defining policies for how long KMS keys should be retained, how they should be securely destroyed at the end of their lifecycle, and how to handle any key material backups.
4. Key management and monitoring: Establishing processes for managing and monitoring KMS keys, such as logging all key usage, conducting periodic audits of key usage, and enforcing key access controls.


### Separation of Duties

Separation of duties is a security principle that involves assigning different roles and responsibilities to different individuals within an organization. This is done to prevent conflicts of interest and reduce the risk of fraud, errors, and other security breaches. It is used to ensure that one individual does not have all the necessary permissions to be able to complete a malicious action.

With regard to KMS, separation of duties involves assigning different roles and responsibilities to different individuals or teams within an organization. This is to prevent unauthorized access to encryption keys and ensure the security of sensitive data. For example, an organization may implement the following separation of duties controls for KMS:

1. Key Administrators: Individuals responsible for creating and managing encryption keys. Key administrators may have full access to KMS, including the ability to create and manage keys, and to grant permissions to other users or roles.

2. Key Owners: Individuals responsible for managing keys and granting permissions to other users or roles. For example, a team can be the key owner and assign specific rights to their keys.

3. Key Users: Individuals or applications that use encryption keys to encrypt or decrypt sensitive data. Key users may have limited access to KMS, restricted only to the keys and operations that they require to perform their specific job functions.

![aws-kms-roles](https://user-images.githubusercontent.com/944576/221543742-8c703b5b-410d-4e15-94e5-c1e5da3dd5d8.png)

In the context of AWS, it is generally not recommended to manage keys using the AWS root user as it has full access to all resources in the account. Instead, it is advised to create a separate IAM role to perform key administration. This approach follows the principle of least privilege, which ensures that users and roles are only granted the minimum permissions necessary to perform their tasks. AWS key administrators may have full access to the KMS API, kms:*, which lets them c§eate and manage encryption keys, but it is advised not to assign full privileges to any role. For example, the following actions limit an IAM principal to only create keys and update its policy but prevent them from using the key for encryption/decryption.

```
"Action": [
  kms:Create*  
  kms:PutKeyPolicy
]
```

The key owners should only have permission to update KMS policies. The key owners' policy should contain the following rights:

The policy for the key user role should only contain actions pertaining to cryptographic operations:
```
"Action": [
  kms:PutKeyPolicy
]
```

It is worth mentioning that the above policy action lets key owners elevate their KMS privileges by updating key policies. It is therefore imperative to protect critical keys using service control policies at the AWS organization level.

### KMS Key Policy
A KMS key policy is attached to a specific KMS key and controls access to that key, regardless of the user or application attempting to access it. The policy specifies the permissions that are granted to IAM users, roles, and AWS services. In addition, it also grants permissions to external entities identified by their AWS account IDs or public keys. It is usually better to provide permissions in key policies that affect one KMS key, rather than in an IAM policy that can apply to many KMS keys.

Every customer master key (CMK) must have exactly one key policy. The statements in the key policy document determine who has permission to access the CMK and how they can use it. This default key policy gives the AWS account (root user) that owns the CMK full access and enables assigning permissions utilizing IAM policies. However, it is possible to disable IAM policies but keep root user access by using the following policy condition:

```
"Condition": {
    "StringEquals": {
        "aws:PrincipalType": "Account"
    }
}
```

This condition makes it possible to centralize key permission management within KMS key policies.

The sample policy below contains three policy statements:

1. Set the root user's rights and add a condition to disable IAM policies.
2. Set key administrators' rights.
3. Set key users' rights.

```
{
    "Version": "2012-10-17",
    "Id": "Key-policy",
    "Statement": [
        {
            "Sid": "Account root user has full access and IAM policies disabled",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:root"
            },
            "Action": "kms:*",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalType": "Account"
                }
            }
        },
        {
            "Sid": "Key administrators can manage the lifecycle of this key",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::123456789012:role/keyadmin"
                ]
            },
            "Action": [
                "kms:Create*",
                "kms:Describe*",
                "kms:Enable*",
                "kms:List*",
                "kms:Put*",
                "kms:Update*",
                "kms:Revoke*",
                "kms:Disable*",
                "kms:Get*",
                "kms:Delete*",
                "kms:TagResource",
                "kms:UntagResource",
                "kms:ScheduleKeyDeletion",
                "kms:CancelKeyDeletion"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Allow use of the key",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::123456789012:role/keyuser",
                ]
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        }
}
```

## Hardening KMS API Access

To access KMS securely from within your VPC, you can create a VPC endpoint for the KMS service using AWS PrivateLink. This eliminates the need for public internet connectivity to KMS and ensures that all traffic between your VPC and KMS stays within the AWS network.


![aws-kms-hardening](https://user-images.githubusercontent.com/944576/221542700-5515dd17-97eb-4185-ad2f-191a5681f6e5.png)

To secure a VPC endpoint in AWS, you can implement several measures, including:

- To control access to the VPC endpoint, you can use VPC security groups to allow or deny inbound and outbound traffic to the endpoint based on IP addresses, protocols, and ports. You can also specify which instances or subnets can access the endpoint by associating the security group with the VPC endpoint.
- You can create IAM policies to control which AWS accounts and roles have access to your VPC endpoints. IAM policies enable you to specify who can perform actions on the VPC endpoint and which actions they can perform.

It is also possible to restrict access to the KMS API using policy conditions. You can add a condition to your KMS key policy to deny encryption and decryption requests unless the request comes from the specified endpoint. The following example demonstrates this functionality:

```
{
  "Sid": "DenyIfRequestNotUsingVpcEndpoint" 
  "Effect": "Deny",
  "Principal": "*",
  "Action": [
      "kms:Encrypt",
      "kms:Decrypt"
  ],
  "Condition": {
      "StringNotEquals": {
          "aws:sourceVpce": "vpce-1234abcdf5678c90a"
       }
  }
}
```

### VPC Endpoint Policy
VPC endpoint policies are resource policies that are specific to VPC endpoints. They allow you to control which VPCs and subnets can access the VPC endpoint and which AWS services the endpoint can reach. VPC endpoint policies are used to grant permissions to resources that are accessed through the VPC endpoint and are managed independently of IAM policies.

By attaching a VPC endpoint policy, it is possible to limit the allowed API operations that applications can invoke through VPC endpoints.

E.g. The following policy allows only encrypt and decrypt operations on the key specified by the resource ID.
```
{
   "Statement":[
      {
         "Sid": "AllowedKmsActions",
         "Principal": "*",
         "Effect": "Allow",
         "Action": [ 
             "kms:Decrypt",
             "kms:Encrypt"
          ],
         "Resource": "arn:aws:kms:eu-west-1:140545465132:key/1234abcd-12ab-34cd-56ef-1234567890ab"
      }
   ]
}
```

In the diagram above, the VPC endpoint is protected by a security group, which allows traffic only from the Kubernetes nodes. As the key policy denies encryption and decryption requests that don't originate from the VPC endpoint, it is safe to say that only the applications running on the nodes can use this key for encrypting and decrypting data.


### Service Control Policies
Service Control Policies (SCPs) are a powerful security mechanism that can be used to restrict access to AWS services and resources, including KMS keys, at the AWS Organization level.

SCPs can be used to restrict access to KMS keys based on specific criteria, such as the IAM user or role attempting to access the key or the AWS service being used. 
SCPs can also be used to control who can create and delete KMS keys within an AWS account. By limiting these actions to only authorized users or roles, you can reduce the risk of accidental or malicious deletion of keys.

The following SCP limits key deletions to only a single role.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "OnlyKMSGuardianCanDeleteThisKeySecurKey",
            "Effect": "Allow",
            "Action": [
                "kms:ScheduleKeyDeletion",
                "kms:Delete*"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:123456789012:key/securekey"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalArn": [
                        "arn:aws:iam::123456789012:role/kmsguardian"
                    ]
                }
            }
        }
    ]
}
```
