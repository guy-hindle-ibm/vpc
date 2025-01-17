---

copyright:
  years: 2022
lastupdated: "2022-12-03"

keywords:

subcollection: vpc

---

{{site.data.keyword.attribute-definition-list}}

# Cross-account encryption for file storage resources
{: #vpc-byok-cross-acct-key-file}

{{site.data.keyword.filestorage_vpc_short}} supports cross-account customer-managed encryption. With this feature, users can access a customer root key (CRK) from another account and then use that CRK to encrypt a new file share.
{: shortdesc}

{{site.data.keyword.filestorage_vpc_full}} is available for customers with special approval to preview this service in the Frankfurt, London, Dallas, Toronto, Washington, Sao Paulo, Sydney, Osaka, and Tokyo regions. Contact your IBM Sales representative if you are interested in getting access.
{: preview}

## About cross account key access and use
{: #byok-cross-acct-about}

Customer root keys in one account can be made available to users in another account to use when they create an encrypted file share. The user in the account that's creating the file share can be the same user as the owner of the root key in the other account. The privileged user from the account that owns the root key [invites the user](/docs/account?topic=account-iamuserinv) of the second account where the file share is to be created and sets the IAM delegated policy to the root keys.

The user who creates the encrypted file share then specifies the CRN of the root key from the account in the [API request](#byok-cross-key-acct-create-API). The user specifies the CRN of the root key in the `encryption_key` property. The user who creates the file share with cross-account encryption also uses the IAM token from their own account.

## Restrictions
{: #byok-cross-key-acct-restrictions}

Creating an encrypted file share in the account that owns the root key and using a root key from a sub-account is not allowed and results in an error.

## Before you begin
{: #byok-cross-key-acct-prereqs}

From {{site.data.keyword.iamshort}} (IAM), authorize access between Cloud File Storage (source service) and {{site.data.keyword.keymanagementserviceshort}} (target service). Specify reader access for the role. For more information, see [Prerequisites for setting up customer-managed encryption](/docs/vpc?topic=vpc-vpc-encryption-planning&interface=api#byok-encryption-prereqs).

You must use the API to see the encryption keys that are used between accounts, and to set up the IAM policy. You can use the UI to access your accounts and see the [IAM authorization](/docs/account?topic=account-serviceauth#serviceauth). However, the UI does not show the encryption keys that are used across accounts. As account user where the root keys are, you can use the [{{site.data.keyword.keymanagementserviceshort}}](/docs/key-protect?topic=key-protect-provision) UI to verify the resources that are associated with the encryption keys.

## Create an IAM policy to establish authorization between accounts
{: #byok-cross-acct-iam}

Before you can create a file share that is encrypted with a root key from another account, you must create an IAM policy for authorizing the second account to use the root key of the owner account.

For encrypted file shares, make a `POST /policies` call to the IAM API to create a policy.

In the following example:

* Account A, which owns the key, grants `AuthorizationDelegator` and `Reader` access to Account B, where the encrypted file share is to be created.

* The `resources` section specifies information about Account A that owns the root key, including the key management service (KMS). In this example, `kms` indicates a {{site.data.keyword.keymanagementserviceshort}} instance. If it were a {{site.data.keyword.hscrypto}} instance, you would specify `hs-crypto`.

* The `subjects` section specifies Account B, where the file share is to be created.  Account B creates the policy and grants reader access for `share` to the KMS. In this example, `kms` indicates a {{site.data.keyword.keymanagementserviceshort}} instance.

```curl
curl -X "POST" "https://iam.cloud.ibm.com/v1/policies" \
     -H "Authorization: <Auth Token>" \
     -H 'Content-Type: application/json' \
     -d $'{
        "roles": [
            {
            "role_id": "crn:v1:bluemix:public:iam::::role:AuthorizationDelegator"
            },
            {
            "role_id": "crn:v1:bluemix:public:iam::::serviceRole:Reader"
            }
        ],
        "resources": [
            {
            "attributes": [
                {
                "name": "Account-A",
                "operator": "stringEquals",
                "value": "<CRK-Account-A-ID>"
                },
                {
                "name": "Key-Protect",
                "operator": "stringEquals",
                "value": "kms" // For HPCS use hs-crypto
                }
              ]
            }
        ],
        "type": "authorization",
        "description": "Reader and Delegator role for KeyProtect Service instance",
        "subjects": [
            {
            "attributes": [
                {
                "name": "serviceName",
                "value": "is"
                },
                {
                "name": "resourceType",
                "value": "share"
                },
                {
                "name": "Account-B",
                "value": "<Account-B-ID>"
                }
              ]
            }
        ]
     }'
```
{: codeblock}

## Create a cross-account encrypted file share with the API
{: #byok-cross-key-acct-create-API}

With the VPC API, [create a file share](/docs/vpc?topic=vpc-file-storage-vpc-encryption&interface=api#fs-byok-api) in one account and specify root key from another account that owns the key. The API calls for doing this are the same as when you create an encrypted file share with a root key from the same account. Make a `POST /shares` call and specify the CRN of the root key from the account that owns the root key in the `encryption_key` property.

For example:

 ```curl
   curl -X POST \
   "$vpc_api_endpoint/v1/shares?version=2022-08-09&generation=2" \
   -H "Authorization: $iam_token" \
   -d '{
       "encryption_key": {
          "crn":"crn:v1:staging:public:kms:us-south:a/df0564dd126042ebb03e0224728ce939:4957299d-0ba0-487f-a1a0-c724a729b8b4:key:0cb88b98-9261-4d07-8329-8f594b6641b5"
        },
        "iops": 1000,
        "name": "my-encrypted-share",
        "profile": {
          "name": "tier-5iops"
        },
        "resource_group": {
           "id": "678523bcbe2b4eada913d32640909956"
         },
        "size": 200,
        "zone": {
          "name": "us-south-2"
        }
      }'
   ```
   {: codeblock}

In the response, the CRN of the encryption key is from the account that owns the root key, while the file share is created in a second account.

```json
{
  "created_at": "2022-08-09T23:28:45Z",
  "crn": "crn:v1:staging:public:is:us-south-1:a/b5c782a8f47a2d1527257e3465f21568::share:r134-fe7219eb-c9a9-4aab-8636-9a57141f0cee",
  "encryption": "user_managed",
  "encryption_key": {
    "crn": "crn:v1:staging:public:kms:us-south:a/df0564dd126042ebb03e0224728ce939:4957299d-0ba0-487f-a1a0-c724a729b8b4:key:0cb88b98-9261-4d07-8329-8f594b6641b5"
  },
  "href": "https://us-south-stage01.iaasdev.cloud.ibm.com/v1/shares/r134-fe7219eb-c9a9-4aab-8636-9a57141f0cee",
  "id": "r134-fe7219eb-c9a9-4aab-8636-9a57141f0cee",
  "initial_owner": {
    "gid": 0,
    "uid": 0
  },
  "iops": 1000,
  "lifecycle_state": "stable",
  "name": "bluitel-test-us-south-1-er5s",
  "profile": {
    "href": "https://us-south-stage01.iaasdev.cloud.ibm.com/v1/share/profiles/custom-iops",
    "name": "custom-iops",
    "resource_type": "share_profile"
  },
  "replication_role": "none",
  "replication_status": "none",
  "replication_status_reasons": [],
  "resource_group": {
    "crn": "crn:v1:staging:public:resource-controller::a/b5c782a8f47a2d1527257e3465f21568::resource-group:5400b7d6ac9f0a183f70abbe8b8d54c6",
    "href": "https://resource-controller.cloud.ibm.com/v2/resource_groups/5400b7d6ac9f0a183f70abbe8b8d54c6",
    "id": "5400b7d6ac9f0a183f70abbe8b8d54c6",
    "name": "Default"
  },
  "resource_type": "share",
  "size": 200,
  "targets": [],
  "user_tags": [
    "description:user-tags-test",
    "tag1:value1",
    "tag2:value2",
    "tag3:value with spaces"
  ],
  "zone": {
    "href": "https://us-south-stage01.iaasdev.cloud.ibm.com/v1/regions/us-south/zones/us-south-1",
    "name": "us-south-1"
  }
}
```
{: screeen}

## Next steps
{: #next-steps-multiten-shares}

Learn more about [data encryption](/docs/vpc?topic=vpc-vpc-encryption-about) for VPC resources.
