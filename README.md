## Transitioning Objects Between Storage Classes
### Objectives:
The goal is to move objects from one storage class to another within a Ceph cluster.

### Prerequisites:
- A running Ceph Cluster.
- RGW services
- An S3 user created.
- Root-level access to the Ceph cluster.

### Procedure:
1. Create a new data pool.
```
root@ceph-host-01:# ceph osd pool create test.hot.data
```

2. Add a new storage class.
```
root@ceph-host-01:# radosgw-admin zonegroup placement add  --rgw-zonegroup default --placement-id default-placement --storage-class hot.test
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

3. Provide zone placement information for the new storage class.
```
root@ceph-host-01:# radosgw-admin zone placement add --rgw-zone default --placement-id default-placement --storage-class hot.test --data-pool test.hot.data
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
root@ceph-host-01:# ceph osd pool application enable test.hot.data rgw
enabled application 'rgw' on pool 'test.hot.data'
```

5. Restart all the rgw daemons.
```
root@ceph-host-01:# ceph orch restart rgw.client
Scheduled to restart rgw.client.test-1.dtqdxk on host 'test-1'
```

6. Create a bucket.
```
root@ceph-host-01:# aws s3api create-bucket --bucket my-test-bucket --create-bucket-configuration LocationConstraint=default:default-placement --endpoint-url http://192.168.9.12:8001
```

7. Add an object to the bucket.
```
root@ceph-host-01:# aws s3api put-object --bucket my-test-bucket --key 10mb_file --endpoint-url http://192.168.9.12:8001
```

8. Create a second data pool.
```
root@ceph-host-01:# ceph osd pool create test.cold.data
```

9. Add a new storage class.
```
root@ceph-host-01:# radosgw-admin zonegroup placement add  --rgw-zonegroup default --placement-id default-placement --storage-class cold.test
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
root@ceph-host-01:# radosgw-admin zone placement add --rgw-zone default --placement-id default-placement --storage-class cold.test --data-pool test.cold.data
```

11. Enable rgw application on the data pool.
```
root@ceph-host-01:# ceph osd pool application enable test.cold.data rgw
enabled application 'rgw' on pool 'test.cold.data'
```

12. Restart all the rgw daemons.
```
root@ceph-host-01:# ceph orch restart rgw.client
Scheduled to restart rgw.client.test-1.dtqdxk on host 'test-1'
```

13. View the zone group configuration.
```
root@ceph-host-01:# radosgw-admin zonegroup get
{
    "id": "9c1e3d4f-8a7b-4d5c-9e3f-2b4a6c8d0e1f",
    "name": "default",
    "api_name": "default",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "c1a2e3f4-5678-9abc-def0-1234567890ab",
    "zones": [
        {
            "id": "c1a2e3f4-5678-9abc-def0-1234567890ab",
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
root@ceph-host-01:# radosgw-admin zone get
{
    "id": "9c1e3d4f-8a7b-4d5c-9e3f-2b4a6c8d0e1f",
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
 root@ceph-host-01:#  aws s3api create-bucket --bucket my-test-bucket --create-bucket-configuration LocationConstraint=default:default-placement --endpoint-url http://192.168.9.12:8001

```

16. List the objects prior to transition.
```
root@ceph-host-01:#  radosgw-admin bucket list --bucket my-test-bucket
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

17. Create a JSON file `lifecycle.json` for lifecycle configuration.

19. Verify lifeycle configuration is not yet applied 
```
root@ceph-host-01:# aws s3api get-bucket-lifecycle-configuration --bucket my-test-bucket

An error occurred (InvalidAccessKeyId) when calling the GetBucketLifecycleConfiguration operation: The AWS Access Key Id you provided does not exist in our records.

```

19. Set the lifecycle configuration for the bucket.
```
root@client-01:~# aws s3api put-bucket-lifecycle-configuration --bucket my-test-bucket --lifecycle-configuration file://lifecycle.json  --endpoint-url http://192.168.9.12:8001
```

20. Retrieve the lifecycle configuration on the bucket.
```
root@ceph-host-01:# aws s3api get-bucket-lifecycle-configuration --bucket my-test-bucket --endpoint-url http://192.168.9.12:8001
{
    "Rules": [
        {
            "Expiration": {
                "Days": 365
            },
            "ID": "double transition and expiration",
            "Prefix": "",
            "Status": "Enabled",
            "Transitions": [
                {
                    "Days": 20,
                    "StorageClass": "cold.test"
                },
                {
                    "Days": 5,
                    "StorageClass": "hot.test"
                }
            ]
        }
    ]
}
```

21. List the objects in the bucket and verify that the storage class is still STANDARD
```
root@ceph-host-01:# radosgw-admin bucket list --bucket my-test-bucket
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

22. Also check the object metadata to confirm the storage class is STANDARD
```
root@ceph-host-01:# aws s3api head-object --bucket my-test-bucket   --key 10mb_file  --endpoint-url http://192.168.9.12:8001
{
    "AcceptRanges": "bytes",
    "Expiration": "expiry-date=\"Sun, 14 Sep 2025 00:00:00 GMT\", rule-id=\"double transition and expiration\"",
    "LastModified": "Fri, 13 Sep 2024 10:19:55 GMT",
    "ContentLength": 10485760,
    "ETag": "\"f1c9645dbc14efddc7d8a322685f26eb\"",
    "ContentType": "application/octet-stream",
    "Metadata": {
        "s3cmd-attrs": "atime:1721976105/ctime:1721976085/gid:0/gname:root/md5:f1c9645dbc14efddc7d8a322685f26eb/mode:33188/mtime:1721976085/uid:0/uname:root"
    },
    "StorageClass": "STANDARD"
}
```

23. Verify that lifecycle configuration has not been scheduled yet
```
root@ceph-host-01:# radosgw-admin lc list
[
    {
        "bucket": ":my-test-bucket:886f47bb-2071-4f12-9fd8-df455f63ac81.44565.1",
        "started": "Thu, 01 Jan 1970 00:00:00 GMT",
        "status": "UNINITIAL"
    }
]
```

24. Set the debug interval to see the lifecycle changes immediately
```
root@ceph-host-01:# ceph config set global rgw_lc_debug_interval 10
```

25. Restart the RGW service to apply new configuration.
```
root@ceph-host-01:# ceph orch restart rgw.client
Scheduled to restart rgw.client.test-1.dtqdxk on host 'test-1'
```

26. Check the status of the lifecycle configuration and ensure it has changed to COMPLETE
```
root@ceph-host-01:# radosgw-admin lc list
[
    {
        "bucket": ":my-test-bucket:886f47bb-2071-4f12-9fd8-df455f63ac81.44565.1",
        "started": "Fri, 13 Sep 2024 10:26:29 GMT",
        "status": "COMPLETE"
    }
]
```

27. Confirm the object has transitioned from STANDARD to hot.test, and finally to cold.test storage class
```
root@ceph-host-01:# aws s3api head-object --bucket my-test-bucket   --key 10mb_file  --endpoint-url http://192.168.9.12:8001
{
    "AcceptRanges": "bytes",
    "Expiration": "expiry-date=\"Sun, 14 Sep 2025 00:00:00 GMT\", rule-id=\"double transition and expiration\"",
    "LastModified": "Fri, 13 Sep 2024 10:19:55 GMT",
    "ContentLength": 10485760,
    "ETag": "\"f1c9645dbc14efddc7d8a322685f26eb\"",
    "ContentType": "application/octet-stream",
    "Metadata": {
        "s3cmd-attrs": "atime:1721976105/ctime:1721976085/gid:0/gname:root/md5:f1c9645dbc14efddc7d8a322685f26eb/mode:33188/mtime:1721976085/uid:0/uname:root"
    },
    "StorageClass": "cold.test"
}
```

28. Verify the storage class transition by listing the bucket contents
```
root@ceph-host-01:# radosgw-admin bucket list --bucket my-test-bucket
[
    {
        "name": "10mb_file",
        "instance": "",
        "ver": {
            "pool": 7,
            "epoch": 5
        },
        "locator": "",
        "exists": "true",
        "meta": {
            "category": 1,
            "size": 10485760,
            "mtime": "2024-09-13T10:19:55.235492Z",
            "etag": "f1c9645dbc14efddc7d8a322685f26eb",
            "storage_class": "cold.test",
            "owner": "test1",
            "owner_display_name": "test1",
            "content_type": "application/octet-stream",
            "accounted_size": 10485760,
            "user_data": "",
            "appendable": "false"
        },
        "tag": "_Datn3CIUvNDyc2Ma27jCAeSS6ZRhS1I",
        "flags": 0,
        "pending_map": [],
        "versioned_epoch": 0
    }
]
```

