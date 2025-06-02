Okay, here are 20 use-case-based interview questions related to Velero on EKS, along with concise answers, formatted in Markdown.

---

# Velero on EKS: Use-Case Based Interview Questions

## Category: Basic Backup & Restore

1.  **Use Case:** Your development team has accidentally deleted a critical Kubernetes Deployment and its associated Persistent Volume Claim (PVC) in a development EKS cluster. They need to recover it quickly.
    * **Question:** As a DevOps engineer, how would you use Velero to restore this specific application without affecting other applications in the namespace?
    * **Answer:** I would identify the last successful Velero backup that contains the deleted resources. Then, I'd use `velero restore create <restore-name> --from-backup <backup-name> --include-resources deployment,persistentvolumeclaims --include-namespaces <namespace> --selector app=<app-label>`. I'd ensure the PVC is also included in the restore if it was deleted.

2.  **Use Case:** You need to establish a daily backup routine for all Kubernetes resources and associated persistent volumes in your production EKS cluster.
    * **Question:** Describe how you would set up Velero to achieve this, including the necessary AWS resources.
    * **Answer:** I'd install Velero on the EKS cluster, configuring it with an S3 bucket for backup storage and an IAM role with appropriate permissions (S3 access, EC2 snapshot permissions) via IRSA (IAM Roles for Service Accounts). I would then create a Velero schedule using `velero schedule create <schedule-name> --schedule="0 0 * * *" --include-cluster-resources=true --snapshot-volumes=true`.

3.  **Use Case:** Your company's compliance policy dictates that all backups must be retained for 30 days and then automatically deleted.
    * **Question:** How would you configure Velero to meet this retention policy for new backups?
    * **Answer:** When creating a Velero schedule or on-demand backup, I would specify the `--ttl` (Time To Live) flag, for example, `--ttl 720h` (for 30 days) in `velero schedule create` or `velero backup create` commands. This ensures backups expire and are cleaned up.

4.  **Use Case:** A critical EKS cluster in `us-east-1` has experienced a region-wide outage, and you need to bring up your applications in a new EKS cluster in `us-west-2` from the last known good backup.
    * **Question:** Outline the high-level steps for performing a cross-region disaster recovery using Velero.
    * **Answer:** I'd first set up a new EKS cluster in `us-west-2`. Then, install Velero on this new cluster, pointing it to the same S3 bucket where backups from `us-east-1` are stored. Finally, I would create a restore operation, potentially using `--restore-volume-data-from-source-cluster` if the original cluster's PV data needs explicit migration, and ensure I map any necessary namespaces if they differ.

## Category: Stateful Applications & Consistency

5.  **Use Case:** You are backing up an EKS cluster that hosts a MongoDB StatefulSet. You want to ensure the MongoDB backup is application-consistent, meaning no data is corrupted or incomplete during the backup.
    * **Question:** How would you modify your MongoDB deployment to achieve application-consistent backups with Velero?
    * **Answer:** I would add `pre.hook.backup.velero.io` and `post.hook.backup.velero.io` annotations to the MongoDB StatefulSet's pod template. The pre-hook would execute `mongo --eval "db.fsyncLock()"` to lock the database for writes, and the post-hook would execute `mongo --eval "db.fsyncUnlock()"` to release the lock, ensuring data is flushed to disk before the volume snapshot.

6.  **Use Case:** You have a custom application that writes data to a Persistent Volume, but it doesn't have a built-in mechanism for quiescing. You still need an application-consistent backup.
    * **Question:** How can you ensure data consistency for this application during a Velero backup?
    * **Answer:** I would use Velero's `exec` hooks with a custom script. The pre-hook script would perform an `fsfreeze` on the relevant mount point within the application container's filesystem. The post-hook script would then `fsunfreeze` the mount point. This ensures all pending I/O operations are completed before the snapshot.

7.  **Use Case:** After restoring a stateful application using Velero, the application fails to start correctly, reporting data corruption. You suspect the backup wasn't truly consistent.
    * **Question:** What steps would you take to troubleshoot this consistency issue?
    * **Answer:** I would examine the Velero backup logs (`velero backup logs <backup-name>`) to see if the pre/post hooks executed successfully and if there were any errors or timeouts. I'd also check the application logs at the time of backup to see if the database or application quiesced correctly. I'd then test the hooks manually on a running pod to ensure they function as expected.

## Category: Granular Control & Filtering

8.  **Use Case:** You only need to back up resources within a specific namespace, `prod-web-app`, and exclude all other namespaces in your EKS cluster to save storage and improve backup speed.
    * **Question:** What Velero command would you use to achieve this?
    * **Answer:** I would use `velero backup create <backup-name> --include-namespaces prod-web-app --snapshot-volumes=true`.

9.  **Use Case:** Your cluster has many `ConfigMap` and `Secret` objects, but you only want to back up `ConfigMap`s and `Secret`s that have the label `backup-me=true`.
    * **Question:** How would you configure your Velero backup to include only these specific resources?
    * **Answer:** I would use label selectors in the backup command: `velero backup create <backup-name> --include-resources configmaps,secrets --selector backup-me=true --snapshot-volumes=true`.

10. **Use Case:** During a restore operation, you want to restore a specific Deployment and its associated Service, but not its Persistent Volume Claim (PVC) because you want to attach it to an existing PV.
    * **Question:** How would you execute such a granular restore using Velero?
    * **Answer:** I would use `velero restore create <restore-name> --from-backup <backup-name> --include-resources deployments,services --exclude-resources persistentvolumeclaims --include-namespaces <namespace>`.

## Category: Advanced Scenarios & Troubleshooting

11. **Use Case:** You've upgraded your EKS cluster to a newer Kubernetes version, and you want to test if your existing Velero backups are compatible with the new cluster version before committing to the upgrade.
    * **Question:** Describe your testing strategy for this scenario.
    * **Answer:** I would create a separate, temporary EKS cluster with the new Kubernetes version. Then, install Velero on this test cluster, pointing it to my existing S3 backup location. I'd perform a full restore of a representative application (especially stateful ones) from an old backup into this new cluster and thoroughly test the application's functionality.

12. **Use Case:** Your Velero backup logs show "pod volume backup failed" errors for several stateful applications.
    * **Question:** What are the common reasons for this error, and how would you begin troubleshooting it in an EKS context?
    * **Answer:** Common reasons include: Velero's `node-agent` (Restic/Kopia) not running or having permissions issues, insufficient permissions for Velero's IAM role to perform EBS snapshots, incorrect `backup.velero.io/backup-volumes` annotations, or `fsfreeze`/`fsyncLock` commands failing. I'd check `velero backup logs`, `kubectl logs -n velero deployment/velero` and `daemonset/node-agent`, review IAM policies, and verify application annotations.

13. **Use Case:** You've performed a Velero restore, and while all Kubernetes objects are recreated, your LoadBalancer Service in AWS does not receive traffic.
    * **Question:** What is a common reason for this issue with Velero and LoadBalancer Services, and how can you resolve it?
    * **Answer:** Kubernetes often re-assigns UIDs to restored Service objects. If the cloud provider's LoadBalancer is tied to the old UID, it won't associate with the new one. The solution usually involves deleting and recreating the affected LoadBalancer Services after the Velero restore is complete, allowing Kubernetes to provision a new one.

14. **Use Case:** You need to monitor the health and progress of your Velero backup and restore operations in real-time.
    * **Question:** How can you integrate Velero with your existing monitoring stack (e.g., Prometheus and Grafana) on EKS?
    * **Answer:** Velero exposes Prometheus metrics. I would configure Prometheus to scrape metrics from the Velero server pod. These metrics include backup/restore durations, success/failure counts, and more. I would then create Grafana dashboards to visualize these metrics and set up alerts for backup failures or long-running operations.

15. **Use Case:** You want to implement a strategy where your EKS cluster can be easily replicated for testing or development purposes, including all application data and configurations.
    * **Question:** Beyond disaster recovery, how can Velero facilitate this "cluster cloning" use case?
    * **Answer:** Velero is ideal for this. I can take a full backup of the source cluster and then restore it to a new EKS cluster (potentially with different storage classes or namespaces if needed, using `restic`/`kopia` or `VolumeSnapshotLocation` remapping). This creates an identical environment for testing new features, upgrades, or debugging without impacting production.

16. **Use Case:** You have an application that uses a custom Kubernetes Custom Resource Definition (CRD).
    * **Question:** Will Velero automatically back up instances of this CRD? What, if any, considerations are there?
    * **Answer:** Yes, Velero will back up instances of the CRD by default, as it backs up all Kubernetes API objects. The main consideration is that the *CRD definition itself* (the schema) also needs to be present in the target cluster *before* restoring instances of that CRD. Velero backs up CRD definitions when `--include-cluster-resources` is used, so ensuring this flag is set for your cluster-wide backups is important.

17. **Use Case:** Your EKS cluster uses IRSA (IAM Roles for Service Accounts) for granular AWS access.
    * **Question:** How would you configure Velero to leverage IRSA for its AWS permissions instead of directly providing AWS access keys?
    * **Answer:** When installing Velero, I would ensure the Velero Service Account (usually `velero` in the `velero` namespace) is annotated with `eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<YOUR_VELERO_IAM_ROLE>`. This role would have the necessary S3 and EC2 permissions. I would then install Velero without providing explicit credentials via a secret (e.g., using `--no-secret`).

18. **Use Case:** You've noticed that your Velero backups are taking an excessively long time to complete, particularly for clusters with many small Persistent Volumes.
    * **Question:** What potential optimizations or alternative strategies would you consider to improve backup performance in this scenario?
    * **Answer:** I would investigate:
        * **Storage Class Optimization:** Ensure PVs are on fast storage types (e.g., GP3 EBS).
        * **Velero Uploader Type:** If using Restic/Kopia, consider if volume snapshots (CSI) are feasible for more direct block-level backups, which are often faster.
        * **Parallelism:** Check Velero's parallelism settings, though often this is handled by the underlying snapshot mechanism.
        * **Filtering:** Review if there are unnecessary namespaces or resources being backed up.
        * **Incremental Backups:** While Velero performs full backups, the underlying snapshotting (e.g., EBS) is incremental, so subsequent backups should be faster. Ensure the snapshotting is actually working.
        * **Network Bandwidth:** Verify network connectivity and bandwidth between the EKS nodes and S3.

19. **Use Case:** You need to restore a backup to a cluster where the original storage class name is different.
    * **Question:** How would you handle this remapping of storage classes during a Velero restore operation?
    * **Answer:** I would use a `Restore` custom resource with a `storageClassMappings` field. For example:
        ```yaml
        apiVersion: velero.io/v1
        kind: Restore
        metadata:
          name: restore-with-sc-remap
          namespace: velero
        spec:
          backupName: <your-backup-name>
          storageClassMappings:
            old-storage-class-name: new-storage-class-name
        ```
        This tells Velero to provision new PVCs using `new-storage-class-name` instead of `old-storage-class-name`.

20. **Use Case:** You have a strict security policy requiring that your S3 backup bucket is immutable and cannot be deleted or modified by anyone, including Velero, after a backup is written.
    * **Question:** How would you configure the S3 bucket and Velero to support this requirement while still allowing Velero to write new backups?
    * **Answer:** This requires careful planning. You would enable **S3 Object Lock** on the S3 bucket to make objects immutable. For Velero to write new backups to such a bucket, you would generally need to create a **versioned S3 bucket**. Velero would write new versions of objects, but previous versions (backups) would be immutable. The IAM policy for Velero would grant `s3:PutObjectVersion`, but not `s3:DeleteObjectVersion`. For pruning, Velero would essentially "mark" old versions for deletion, but the S3 lifecycle rules and Object Lock would enforce the immutability for the retention period. This is an advanced scenario requiring coordination between Velero's lifecycle management and S3's object lock capabilities.
