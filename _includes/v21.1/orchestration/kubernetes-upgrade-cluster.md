It is strongly recommended that you regularly upgrade your CockroachDB version in order to pick up bug fixes, performance improvements, and new features. The [CockroachDB upgrade documentation](upgrade-cockroach-version.html) describes how to perform a "rolling upgrade" of a CockroachDB cluster by stopping and restarting nodes one at a time. This is to ensure that the cluster remains available during the upgrade.

The corresponding process on Kubernetes is a [staged update](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#staging-an-update), in which the Docker image is updated in the CockroachDB StatefulSet and then applied to the pods one at a time.

<section class="filter-content" markdown="1" data-scope="operator">
1. Verify that you can upgrade.

    To upgrade to a new version, you must first be on a production release of the previous version. The release does not need to be the latest production release of the previous version, but it must be a production release rather than a testing release (alpha/beta).

    Therefore, if you are upgrading from v20.1 to v20.2, or from a testing release (alpha/beta) of v20.2 to v21.1:

    1. First [upgrade to a production release of v20.2](../v20.2/orchestrate-cockroachdb-with-kubernetes.html#upgrade-the-cluster). Be sure to complete all the steps.

    1. Then return to this page and perform a second upgrade to v21.1.

    1. If you are upgrading from any production release of v20.2, or from any earlier v21.1 release, you do not have to go through intermediate releases; continue to step 2.

1. Verify the overall health of your cluster using the [DB Console](ui-overview.html). On the **Overview**:
    - Under **Node Status**, make sure all nodes that should be live are listed as such. If any nodes are unexpectedly listed as suspect or dead, identify why the nodes are offline and either restart them or [decommission](#remove-nodes) them before beginning your upgrade. If there are dead and non-decommissioned nodes in your cluster, it will not be possible to finalize the upgrade (either automatically or manually).
    - Under **Replication Status**, make sure there are 0 under-replicated and unavailable ranges. Otherwise, performing a rolling upgrade increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability. Therefore, it's important to identify and resolve the cause of range under-replication and/or unavailability before beginning your upgrade.
    - In the **Node List**:
        - Make sure all nodes are on the same version. If any nodes are behind, upgrade them to the cluster's current version first, and then start this process over.
        - Make sure capacity and memory usage are reasonable for each node. Nodes must be able to tolerate some increase in case the new version uses more resources for your workload. Also go to **Metrics > Dashboard: Hardware** and make sure CPU percent is reasonable across the cluster. If there's not enough headroom on any of these metrics, consider [adding nodes](#add-nodes) to your cluster before beginning your upgrade.

1. Review the [backward-incompatible changes in v21.1](../releases/v21.1.0.html#backward-incompatible-changes) and [deprecated features](../releases/v21.1.0.html#deprecations). If any affect your deployment, make the necessary changes before starting the rolling upgrade to v21.1.

1. Decide how the upgrade will be finalized.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    By default, after all nodes are running the new version, the upgrade process will be **auto-finalized**. This will enable certain [features and performance improvements introduced in v21.1](upgrade-cockroach-version.html#features-that-require-upgrade-finalization). After finalization, however, it will no longer be possible to perform a downgrade to v20.2. In the event of a catastrophic failure or corruption, the only option will be to start a new cluster using the old binary and then restore from one of the backups created prior to performing the upgrade. For this reason, **we recommend disabling auto-finalization** so you can monitor the stability and performance of the upgraded cluster before finalizing the upgrade, but note that you will need to follow all of the subsequent directions, including the manual finalization in a later step.

    1. Start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-2 \
        -- ./cockroach sql \
        --certs-dir cockroach-certs
        ~~~

    1. Set the `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html) to the version you are upgrading from:

        {% include copy-clipboard.html %}
        ~~~ sql
        > SET CLUSTER SETTING cluster.preserve_downgrade_option = '20.2';
        ~~~

    1. Exit the SQL shell and pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

1. Change the desired Docker image:

    Open and edit `example.yaml`:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ vi example.yaml
    ~~~

    ~~~
    image:
      name: cockroachdb/cockroach:{{page.release_info.version}}
    ~~~

    Apply `example.yaml` with the new image:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl apply -f example.yaml
    ~~~

    The Operator will perform the staged update.

1. If you then check the status of your cluster's pods, you should see them being restarted:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS        RESTARTS   AGE
    cockroachdb-0   1/1       Running       0          2m
    cockroachdb-1   1/1       Running       0          2m
    cockroachdb-2   1/1       Running       0          2m
    cockroachdb-3   0/1       Terminating   0          1m
    ...
    ~~~

1. This will continue until all of the pods have restarted and are running the new image. To check the image of each pod to determine whether they've all be upgraded, run:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}'
    ~~~

    ~~~
    cockroachdb-0   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-1   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-2   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-3   cockroachdb/cockroach:{{page.release_info.version}}
    ...
    ~~~

    You can also check the CockroachDB version of each node in the [DB Console](ui-cluster-overview-page.html#node-details).

1. Finish the upgrade.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    If you disabled auto-finalization earlier, monitor the stability and performance of your cluster for as long as you require to feel comfortable with the upgrade (generally at least a day). If during this time you decide to roll back the upgrade, repeat the rolling restart procedure with the old binary.

    Once you are satisfied with the new version, re-enable auto-finalization.

    1. Start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-2 \
        -- ./cockroach sql \
        --certs-dir cockroach-certs
        ~~~    

    1. Re-enable auto-finalization:

        {% include copy-clipboard.html %}
        ~~~ sql
        > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
        ~~~

    1. Exit the SQL shell and pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

</section>

<section class="filter-content" markdown="1" data-scope="manual">
1. Verify that you can upgrade.

    To upgrade to a new version, you must first be on a production release of the previous version. The release does not need to be the latest production release of the previous version, but it must be a production release rather than a testing release (alpha/beta).

    Therefore, if you are upgrading from v20.1 to v20.2, or from a testing release (alpha/beta) of v20.2 to v21.1:

    1. First [upgrade to a production release of v20.2](../v20.2/orchestrate-cockroachdb-with-kubernetes.html#upgrade-the-cluster). Be sure to complete all the steps.

    1. Then return to this page and perform a second upgrade to v21.1.

    1. If you are upgrading from any production release of v20.2, or from any earlier v21.1 release, you do not have to go through intermediate releases; continue to step 2.

1. Verify the overall health of your cluster using the [DB Console](ui-overview.html). On the **Overview**:
    - Under **Node Status**, make sure all nodes that should be live are listed as such. If any nodes are unexpectedly listed as suspect or dead, identify why the nodes are offline and either restart them or [decommission](#remove-nodes) them before beginning your upgrade. If there are dead and non-decommissioned nodes in your cluster, it will not be possible to finalize the upgrade (either automatically or manually).
    - Under **Replication Status**, make sure there are 0 under-replicated and unavailable ranges. Otherwise, performing a rolling upgrade increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability. Therefore, it's important to identify and resolve the cause of range under-replication and/or unavailability before beginning your upgrade.
    - In the **Node List**:
        - Make sure all nodes are on the same version. If any nodes are behind, upgrade them to the cluster's current version first, and then start this process over.
        - Make sure capacity and memory usage are reasonable for each node. Nodes must be able to tolerate some increase in case the new version uses more resources for your workload. Also go to **Metrics > Dashboard: Hardware** and make sure CPU percent is reasonable across the cluster. If there's not enough headroom on any of these metrics, consider [adding nodes](#add-nodes) to your cluster before beginning your upgrade.

1. Review the [backward-incompatible changes in v21.1](../releases/v21.1.0.html#backward-incompatible-changes) and [deprecated features](../releases/v21.1.0.html#deprecations). If any affect your deployment, make the necessary changes before starting the rolling upgrade to v21.1.

1. Decide how the upgrade will be finalized.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    By default, after all nodes are running the new version, the upgrade process will be **auto-finalized**. This will enable certain [features and performance improvements introduced in v21.1](upgrade-cockroach-version.html#features-that-require-upgrade-finalization). After finalization, however, it will no longer be possible to perform a downgrade to v20.2. In the event of a catastrophic failure or corruption, the only option will be to start a new cluster using the old binary and then restore from one of the backups created prior to performing the upgrade. For this reason, **we recommend disabling auto-finalization** so you can monitor the stability and performance of the upgraded cluster before finalizing the upgrade, but note that you will need to follow all of the subsequent directions, including the manual finalization in a later step.

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \-- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=cockroachdb-public
        ~~~

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=cockroachdb-public
        ~~~

    {% endif %}

    1. Set the `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html) to the version you are upgrading from:

        {% include copy-clipboard.html %}
        ~~~ sql
        > SET CLUSTER SETTING cluster.preserve_downgrade_option = '20.2';
        ~~~

    1. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

1. Add a [partition](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#staging-an-update) to the update strategy defined in the StatefulSet. Only the pods numbered greater than or equal to the partition value will be updated. For a cluster with 3 pods (e.g., `cockroachdb-0`, `cockroachdb-1`, `cockroachdb-2`) the partition value should be 2:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    -p='{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~

1. Kick off the upgrade process by changing the Docker image used in the CockroachDB StatefulSet:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    --type='json' \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"cockroachdb/cockroach:{{page.release_info.version}}"}]'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~

1. Check the status of your cluster's pods. You should see one of them being restarted:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS        RESTARTS   AGE
    cockroachdb-0   1/1       Running       0          2m
    cockroachdb-1   1/1       Running       0          2m
    cockroachdb-2   0/1       Terminating   0          1m
    ...
    ~~~

1. After the pod has been restarted with the new image, get a shell into the pod and start the CockroachDB [built-in SQL client](cockroach-sql.html):

    {% if page.secure == true %}
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure \-- ./cockroach sql \
    --certs-dir=/cockroach-certs \
    --host=cockroachdb-public
    ~~~

    {% else %}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl run cockroachdb -it \
    --image=cockroachdb/cockroach \
    --rm \
    --restart=Never \
    -- sql \
    --insecure \
    --host=cockroachdb-public
    ~~~

    {% endif %}

1. Run the following SQL query to verify that the number of under-replicated ranges is zero:

    {% include copy-clipboard.html %}
    ~~~ sql
    SELECT sum((metrics->>'ranges.underreplicated')::DECIMAL)::INT AS ranges_underreplicated FROM crdb_internal.kv_store_status;
    ~~~

    ~~~
      ranges_underreplicated
    --------------------------
                           0
    (1 row)        
    ~~~

    This indicates that it is safe to proceed to the next pod.

1. Exit the SQL shell:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

1. Decrement the partition value by 1 to allow the next pod in the cluster to update:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    -p='{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":1}}}}'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~

1. Repeat steps 4-8 until all pods have been restarted and are running the new image (the final partition value should be `0`).

1. Check the image of each pod to confirm that all have been upgraded:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}'
    ~~~

    ~~~
    cockroachdb-0   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-1   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-2   cockroachdb/cockroach:{{page.release_info.version}}
    ...
    ~~~

    You can also check the CockroachDB version of each node in the [DB Console](ui-cluster-overview-page.html#node-details).


1. Finish the upgrade.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    If you disabled auto-finalization earlier, monitor the stability and performance of your cluster for as long as you require to feel comfortable with the upgrade (generally at least a day). If during this time you decide to roll back the upgrade, repeat the rolling restart procedure with the old binary.

    Once you are satisfied with the new version, re-enable auto-finalization:

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=cockroachdb-public
        ~~~

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=cockroachdb-public
        ~~~

    {% endif %}

    2. Re-enable auto-finalization:

        {% include copy-clipboard.html %}
        ~~~ sql
        > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
        ~~~

    3. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

</section>

<section class="filter-content" markdown="1" data-scope="helm">
1. Verify that you can upgrade.

    To upgrade to a new version, you must first be on a production release of the previous version. The release does not need to be the latest production release of the previous version, but it must be a production release rather than a testing release (alpha/beta).

    Therefore, if you are upgrading from v20.1 to v20.2, or from a testing release (alpha/beta) of v20.2 to v21.1:

    1. First [upgrade to a production release of v20.2](../v20.2/orchestrate-cockroachdb-with-kubernetes.html#upgrade-the-cluster). Be sure to complete all the steps.

    1. Then return to this page and perform a second upgrade to v21.1.

    1. If you are upgrading from any production release of v20.2, or from any earlier v21.1 release, you do not have to go through intermediate releases; continue to step 2.

1. Verify the overall health of your cluster using the [DB Console](ui-overview.html). On the **Overview**:
    - Under **Node Status**, make sure all nodes that should be live are listed as such. If any nodes are unexpectedly listed as suspect or dead, identify why the nodes are offline and either restart them or [decommission](#remove-nodes) them before beginning your upgrade. If there are dead and non-decommissioned nodes in your cluster, it will not be possible to finalize the upgrade (either automatically or manually).
    - Under **Replication Status**, make sure there are 0 under-replicated and unavailable ranges. Otherwise, performing a rolling upgrade increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability. Therefore, it's important to identify and resolve the cause of range under-replication and/or unavailability before beginning your upgrade.
    - In the **Node List**:
        - Make sure all nodes are on the same version. If any nodes are behind, upgrade them to the cluster's current version first, and then start this process over.
        - Make sure capacity and memory usage are reasonable for each node. Nodes must be able to tolerate some increase in case the new version uses more resources for your workload. Also go to **Metrics > Dashboard: Hardware** and make sure CPU percent is reasonable across the cluster. If there's not enough headroom on any of these metrics, consider [adding nodes](#add-nodes) to your cluster before beginning your upgrade.

1. Review the [backward-incompatible changes in v21.1](../releases/v21.1.0.html#backward-incompatible-changes) and [deprecated features](../releases/v21.1.0.html#deprecations). If any affect your deployment, make the necessary changes before starting the rolling upgrade to v21.1.

1. Decide how the upgrade will be finalized.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    By default, after all nodes are running the new version, the upgrade process will be **auto-finalized**. This will enable certain [features and performance improvements introduced in v21.1](upgrade-cockroach-version.html#features-that-require-upgrade-finalization). After finalization, however, it will no longer be possible to perform a downgrade to v20.2. In the event of a catastrophic failure or corruption, the only option will be to start a new cluster using the old binary and then restore from one of the backups created prior to performing the upgrade. For this reason, **we recommend disabling auto-finalization** so you can monitor the stability and performance of the upgraded cluster before finalizing the upgrade, but note that you will need to follow all of the subsequent directions, including the manual finalization in a later step.

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=my-release-cockroachdb-public
        ~~~

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=my-release-cockroachdb-public
        ~~~

    {% endif %}

    1. Set the `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html) to the version you are upgrading from:

        {% include copy-clipboard.html %}
        ~~~ sql
        > SET CLUSTER SETTING cluster.preserve_downgrade_option = '20.2';
        ~~~

    1. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

1. Add a [partition](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#staging-an-update) to the update strategy defined in the StatefulSet. Only the pods numbered greater than or equal to the partition value will be updated. For a cluster with 3 pods (e.g., `cockroachdb-0`, `cockroachdb-1`, `cockroachdb-2`) the partition value should be 2:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set statefulset.updateStrategy.rollingUpdate.partition=2
    ~~~

1. Kick off the upgrade process by changing the Docker image used in the CockroachDB StatefulSet:

    {{site.data.alerts.callout_info}}
    For Helm, you must remove the cluster initialization job from when the cluster was created before the cluster version can be changed.
    {{site.data.alerts.end}}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete job my-release-cockroachdb-init
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set image.tag={{page.release_info.version}} \
    --reuse-values
    ~~~

1. Check the status of your cluster's pods. You should see one of them being restarted:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME                                READY     STATUS              RESTARTS   AGE
    my-release-cockroachdb-0            1/1       Running             0          2m
    my-release-cockroachdb-1            1/1       Running             0          3m
    my-release-cockroachdb-2            0/1       ContainerCreating   0          25s
    my-release-cockroachdb-init-nwjkh   0/1       ContainerCreating   0          6s
    ...
    ~~~

    {{site.data.alerts.callout_info}}
    Ignore the pod for cluster initialization. It is re-created as a byproduct of the StatefulSet configuration but does not impact your existing cluster.
    {{site.data.alerts.end}}

1. After the pod has been restarted with the new image, get a shell into the pod and start the CockroachDB [built-in SQL client](cockroach-sql.html):

    {% if page.secure == true %}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure \
    -- ./cockroach sql \
    --certs-dir=/cockroach-certs \
    --host=my-release-cockroachdb-public
    ~~~

    {% else %}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl run cockroachdb -it \
    --image=cockroachdb/cockroach \
    --rm \
    --restart=Never \
    -- sql \
    --insecure \
    --host=my-release-cockroachdb-public
    ~~~
    {% endif %}

1. Run the following SQL query to verify that the number of underreplicated ranges is zero:

    {% include copy-clipboard.html %}
    ~~~ sql
    SELECT sum((metrics->>'ranges.underreplicated')::DECIMAL)::INT AS ranges_underreplicated FROM crdb_internal.kv_store_status;
    ~~~

    ~~~
      ranges_underreplicated
    --------------------------
                           0
    (1 row)        
    ~~~

    This indicates that it is safe to proceed to the next pod.

1. Exit the SQL shell:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

1. Decrement the partition value by 1 to allow the next pod in the cluster to update:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set statefulset.updateStrategy.rollingUpdate.partition=1 \
    ~~~

1. Repeat steps 4-8 until all pods have been restarted and are running the new image (the final partition value should be `0`).

1. Check the image of each pod to confirm that all have been upgraded:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}'
    ~~~

    ~~~
    my-release-cockroachdb-0    cockroachdb/cockroach:{{page.release_info.version}}
    my-release-cockroachdb-1    cockroachdb/cockroach:{{page.release_info.version}}
    my-release-cockroachdb-2    cockroachdb/cockroach:{{page.release_info.version}}
    ...
    ~~~

    You can also check the CockroachDB version of each node in the [DB Console](ui-cluster-overview-page.html#node-details).


1. Finish the upgrade.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v20.2.x to v21.1. For upgrades within the v21.1.x series, skip this step.
    {{site.data.alerts.end}}

    If you disabled auto-finalization earlier, monitor the stability and performance of your cluster for as long as you require to feel comfortable with the upgrade (generally at least a day). If during this time you decide to roll back the upgrade, repeat the rolling restart procedure with the old binary.

    Once you are satisfied with the new version, re-enable auto-finalization:

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=my-release-cockroachdb-public
        ~~~

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=my-release-cockroachdb-public
        ~~~

    {% endif %}

    2. Re-enable auto-finalization:

        {% include copy-clipboard.html %}
        ~~~ sql
        > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
        ~~~

    3. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

</section>
