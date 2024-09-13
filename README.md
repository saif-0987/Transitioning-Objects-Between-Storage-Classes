## Transitioning-of-an-object-from-one-storage-class-to_another
### Objectives:
We can move objects from one storage class to another storage class

### Prerequisites:
- Obviously a running Ceph Cluster.
- RGW services
- S3 user should be created
- Root level access

### Procedure:
1. Create a new data pool.
```
root@ceph-host-01:~# ceph osd pool create test.hot.data
```
2. Add a new storage class.
```
root@client-01:/# radosgw-admin zonegroup placement add  --rgw-zonegroup default --placement-id default-placement --stor
age-class cold.test
[
    {
        "key": "default-placement",
        "val": {
            "name": "default-placement",
            "tags": [],
            "storage_classes": [
                "STANDARD",
                "hot.test"
            ]
        }
    }
]
```
3. Provide the zone placement information for the new storage class.

```
root@client-01:/# radosgw-admin zone placement add --rgw-zone default --placement-id default-placement --storage-class hot.test --data-pool test.hot.data
{
           "key": "default-placement",
           "val": {
               "index_pool": "test_zone.rgw.buckets.index",
               "storage_classes": {
                   "STANDARD": {
                       "data_pool": "test.hot.data"
                   },
                   "hot.test": {
                       "data_pool": "test.hot.data",
                  }
               },
               "data_extra_pool": "",
               "index_type": 0
           }
```
4. Enable the rgw application on the data pool.
```
root@client-01:/# ceph osd pool application enable test.hot.data rgw
enabled application 'rgw' on pool 'test.hot.data'
```

5. Restart all the rgw daemons.
```
root@client-01:/# ceph orch restart rgw.client
Scheduled to restart rgw.client.test-1.dtqdxk on host 'test-1'
```

6. Create a bucket.
```
 aws s3api create-bucket --bucket my-test-bucket --create-bucket-configuration LocationConstraint=default:default-placement --endpoint-url http://myhost-01.com
```
7. Add the object.
```
aws s3api put-object --bucket my-test-bucket --key 10mb_file --endpoint-url http://myhost-01.com
```
8. Create a second data pool.

```
root@client-01:/# ceph osd pool create test.cold.data
```

9. Add a new storage class.
```
root@client-01:/# radosgw-admin zonegroup placement add  --rgw-zonegroup default --placement-id default-placement --stor
age-class cold.test
[
    {
        "key": "default-placement",
        "val": {
            "name": "default-placement",
            "tags": [],
            "storage_classes": [
                "STANDARD",
                "cold.test",
                "hot.test"
            ]
        }
    }
]
```

10. Provide the zone placement information for the new storage class.
```
root@client-01:/# radosgw-admin zone placement add --rgw-zone default --placement-id default-placement --storage-class c
old.test --data-pool test.cold.data
```

11. Enable rgw application on the data pool.
```
root@client-01:/# ceph osd pool application enable test.cold.data rgw
enabled application 'rgw' on pool 'test.cold.data'
```
12. Restart all the rgw daemons.
```
root@client-01:/# ceph orch restart rgw.client
Scheduled to restart rgw.client.test-1.dtqdxk on host 'test-1'
```
13. View the zone group configuration.
```
root@client-01:/# radosgw-admin zonegroup get
{
    "id": "fa0ab50a-c185-45bf-9b0f-881b139b3264",
    "name": "default",
    "api_name": "default",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "886f47bb-2071-4f12-9fd8-df455f63ac81",
    "zones": [
        {
            "id": "886f47bb-2071-4f12-9fd8-df455f63ac81",
            "name": "default",
            "endpoints": [],
            "log_meta": "false",
            "log_data": "false",
            "bucket_index_max_shards": 11,
            "read_only": "false",
            "tier_type": "",
            "sync_from_all": "true",
            "sync_from": [],
            "redirect_zone": ""
        }
    ],
    "placement_targets": [
        {
            "name": "default-placement",
            "tags": [],
            "storage_classes": [
                "STANDARD",
                "cold.test",
                "hot.test"
            ]
        }
    ],
    "default_placement": "default-placement",
    "realm_id": "",
    "sync_policy": {
        "groups": []
    }
}
```
14. View the zone configuration.
```
root@client-01:/# radosgw-admin zone get
{
    "id": "886f47bb-2071-4f12-9fd8-df455f63ac81",
    "name": "default",
    "domain_root": "default.rgw.meta:root",
    "control_pool": "default.rgw.control",
    "gc_pool": "default.rgw.log:gc",
    "lc_pool": "default.rgw.log:lc",
    "log_pool": "default.rgw.log",
    "intent_log_pool": "default.rgw.log:intent",
    "usage_log_pool": "default.rgw.log:usage",
    "roles_pool": "default.rgw.meta:roles",
    "reshard_pool": "default.rgw.log:reshard",
    "user_keys_pool": "default.rgw.meta:users.keys",
    "user_email_pool": "default.rgw.meta:users.email",
    "user_swift_pool": "default.rgw.meta:users.swift",
    "user_uid_pool": "default.rgw.meta:users.uid",
    "otp_pool": "default.rgw.otp",
    "system_key": {
        "access_key": "",
        "secret_key": ""
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "default.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "default.rgw.buckets.data"
                    },
                    "cold.test": {
                        "data_pool": "test.cold.data"
                    },
                    "hot.test": {
                        "data_pool": "test.hot.data"
                    }
                },
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "realm_id": "",
    "notif_pool": "default.rgw.log:notif"
}
```

15. Create a bucket.
```
 root@client-01:/# aws s3api create-bucket --bucket my-test-bucket --create-bucket-configuration LocationConstraint=default:default-placement --endpoint-url http://myhost-01.com

```
16. List the objects prior to transition.
```
root@client-01:/# radosgw-admin bucket list --bucket my-test-bucket
[
    {
        "name": "10mb_file",
        "instance": "",
        "ver": {
            "pool": 7,
            "epoch": 4
        },
        "locator": "",
        "exists": "true",
        "meta": {
            "category": 1,
            "size": 10485760,
            "mtime": "2024-09-13T10:19:55.235492Z",
            "etag": "f1c9645dbc14efddc7d8a322685f26eb",
            "storage_class": "STANDARD",
            "owner": "test1",
            "owner_display_name": "test1",
            "content_type": "application/octet-stream",
            "accounted_size": 10485760,
            "user_data": "",
            "appendable": "false"
        },
        "tag": "886f47bb-2071-4f12-9fd8-df455f63ac81.44676.8467969203769252872",
        "flags": 0,
        "pending_map": [],
        "versioned_epoch": 0
    }
]
```

17. Create a JSON file for lifecycle configuration.

18.  Set the lifecycle configuration on the bucket.
```
root@client-01:~# aws s3api get-bucket-lifecycle-configuration --bucket my-test-bucket

An error occurred (InvalidAccessKeyId) when calling the GetBucketLifecycleConfiguration operation: The AWS Access Key Id you provided does not exist in our records.

```

19. 
