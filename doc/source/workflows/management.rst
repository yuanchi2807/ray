Workflow Management
===================

Workflow IDs
------------
Each workflow has a unique ``workflow_id``. By default, when you call ``.run()`` or ``.run_async()``, a random id is generated. It is recommended you explicitly assign each workflow an id via ``.run(workflow_id="id")``.

If ``.run()`` is called with a previously used workflow id, the workflow will be resumed from the previous execution.

Workflow Status
---------------
A workflow can be in one of several status:

=================== =======================================================================================
Status              Description
=================== =======================================================================================
RUNNING             The workflow is currently running in the cluster.
PENDING             The workflow is queued and waited to be executed.
FAILED              This workflow failed with an application error. It can be resumed from the failed task.
RESUMABLE           This workflow failed with a system error. It can be resumed from the failed task.
CANCELED            The workflow was canceled. Its result is unavailable, and it cannot be resumed.
SUCCESSFUL          The workflow has been executed successfully.
=================== =======================================================================================

Single workflow management APIs
-------------------------------

.. code-block:: python

    from ray import workflow

    # Get the status of a workflow.
    try:
        status = workflow.get_status(workflow_id="workflow_id")
        assert status in {
            "RUNNING", "RESUMABLE", "FAILED",
            "CANCELED", "SUCCESSFUL"}
    except ValueError:
        print("Workflow doesn't exist.")

    # Resume a workflow.
    print(workflow.resume(workflow_id="workflow_id"))
    # return is an ObjectRef which is the result of this workflow

    # Cancel a workflow.
    workflow.cancel(workflow_id="workflow_id")

    # Delete the workflow.
    workflow.delete(workflow_id="workflow_id")

Bulk workflow management APIs
-----------------------------

.. code-block:: python

    # List all running workflows.
    print(workflow.list_all("RUNNING"))
    # [("workflow_id_1", "RUNNING"), ("workflow_id_2", "RUNNING")]

    # List RUNNING and CANCELED workflows.
    print(workflow.list_all({"RUNNING", "CANCELED"}))
    # [("workflow_id_1", "RUNNING"), ("workflow_id_2", "CANCELED")]

    # List all workflows.
    print(workflow.list_all())
    # [("workflow_id_1", "RUNNING"), ("workflow_id_2", "CANCELED")]

    # Resume all resumable workflows. This won't include failed workflow
    print(workflow.resume_all())
    # [("workflow_id_1", ObjectRef), ("workflow_id_2", ObjectRef)]

    # To resume workflows including failed ones, use `include_failed=True`
    print(workflow.resume_all(include_failed=True))
    # [("workflow_id_1", ObjectRef), ("workflow_id_3", ObjectRef)]

Recurring workflows
-------------------

Ray Workflow currently has no built-in job scheduler. You can however easily use any external job scheduler to interact with your Ray cluster (via :ref:`job submission <jobs-overview>` or :ref:`client connection <ray-client-ref>`) trigger workflow runs.

Storage Configuration
---------------------
Ray Workflow supports two types of storage backends out of the box:

*  Local file system: the data is stored locally. This is only for single node testing. It needs to be a NFS to work with multi-node clusters. To use local storage, specify ``ray.init(storage="/path/to/storage_dir")``.
*  S3: Production users should use S3 as the storage backend. Enable S3 storage with ``ray.init(storage="s3://bucket/path")``.

Additional storage backends can be written by subclassing the ``Storage`` class and passing a storage instance to ``ray.init()`` [TODO: note that the Storage API is not currently stable].

If left unspecified, ``/tmp/ray/workflow_data`` will be used for temporary storage. This default setting *will only work for single-node Ray clusters*.

Concurrency Control
-------------------
Ray Workflow supports concurrency control. You can support the maximum running workflows and maximum pending workflows via ``workflow.init()``
before executing any workflow. ``workflow.init()`` again with a different configuration would raise an error.

For example, ``workflow.init(max_running_workflows=10, max_pending_workflows=50)`` means there will be at most 10 workflows running, 50 workflows pending.
Submitting workflows when the number of pending workflows are at maximum would raise ``queue.Full("Workflow queue has been full")``. Getting the output of a pending workflow would be blocking until the workflow finishes running later.

A pending workflows has the ``PENDING`` status. After the pending workflow get interrupted (e.g., a cluster failure), it can be resumed.
When resuming interrupted workflows that were running and pending with ``workflow.resume_all()``, running workflows have higher priority than pending workflows (i.e. the pending workflows would still likely be pending).

.. note::

  We currently does not guarantee that resumed pending workflows are running in the same order as they originally did.

Handling Dependencies
---------------------

**Note: This feature is not yet implemented.**

Ray logs the runtime environment (code and dependencies) of the workflow to storage at submission time. This ensures that the workflow can be resumed at a future time on a different Ray cluster.

You can also explicitly set the runtime environment for a particular task (e.g., specify conda environment, container image, etc.).
