#### AOEngine: 

- Run playbook 

```python
def run(tenant=None, run_playbook=None):
    logger = current_app.logger
    if not run_playbook:
        return {}

    playbook_id = run_playbook["playbook_id"]
    config_id = run_playbook.get("config_id", None)
    input_event = run_playbook.get("input_event")
    args = run_playbook.get("args")
    kwargs = run_playbook.get("kwargs")
    celery_task_id = str(uuid4())

    task_info = {
        "owner": run_playbook.get("owner"),
        "celery_task_id": celery_task_id,
        "playbook_id": playbook_id,
        "config_id": config_id,
        "input_event": json.dumps(input_event),
        "time_start": int(time.time() * 1000),
        "time_end": None,
        "result": None,
        "status": Status.INITIALIZED.value,
        "instance_name": make_instance_name(playbook_id, config_id),
    }

    task = records.create(tenant=tenant, table=TASK_TABLE, data=task_info)

    task_id = task["_id"]
    logger.info(
        f"Sent task to worker: task_id={task_id} tenant_id={tenant}"
        f" playbook_id={playbook_id} config_id={config_id}"
        f" input_event={str(input_event)[:100]} ..."
    )

    instance = {
        "_id": None,
        "tenant_id": tenant,
        "playbook_id": playbook_id,
        "config_id": config_id,
        "name": make_instance_name(playbook_id, config_id),
    }

    context = {
        "tenant": tenant,
        "instance": instance,
        "task_id": task_id,
        "input_event": input_event,
        "args": args,
        "kwargs": kwargs,
    }
    try:
        queue = get_queue(tenant, instance["playbook_id"])
        call_instance.apply_async(kwargs=context, task_id=celery_task_id, queue=queue)
        task["status"] = Status.SENT_TO_WORKER.value
        logger.info(f"Send task {task['_id']} to queue {queue}")
    except OperationalError as ex:
        logger.exeption("Sending task raised: %r", ex)
        task["status"] = Status.FAILED.value
        task["result"] = str(ex)

    # task = records.create(tenant=tenant,
    #                       table=TASK_TABLE,
    #                       data=task_info)

    task = records.update(tenant, TASK_TABLE, task_id, data={"status": task["status"]})

    return task

```



---------------------------------

#### AOWorker

- Hàm thực hiện task trong các worker.

```python
@celery.task(bind=True)
def call_instance(self, **context):
    """Call playbook's function. This function implements the process flow
    to run a playbook. The detail steps are implemented in
    __call_playbook_fn.

    """
    logger = current_app.logger

    logger.info(f"Received task context: {context}.")
    context["task_obj"] = self
    # Check for assign dupplication
    try:
        prepare = prepare_instance(**context)
        if prepare["status"] == Status.ABORTED:
            return
        if prepare["status"] == Status.CANCELLED:
            result = prepare
        else:
            result = call_instance_fn(**context)
        context["result"] = result
    except Exception as ex:
        logger.error(ex)
        logger.debug(traceback.print_exc(10))
        context["result"] = {"status": Status.FAILED, "error": Error.ERROR}
    finalize_instance(**context)

```

-------------------------------------------------

#### Alert -> Case -> Ticket flow

- case được tạo từ alert.
- nếu chưa có case cho alert (không tìm thấy similar alert) thì tự động tạo case từ alert.
- ticket phải được tạo từ case.
- ticket được tạo từ case khi 2 điều sau được đáp ứng: 
  - trong ao_config phải có trường ticket_craeted_allow: True.
  - tất cả các ticket trong case đều đã được đóng? 
- ticket đầu tiên phải do người tạo? (nếu không thì sẽ không có ticket cho alert mới, tất cả alert sẽ chỉ được gộp vào case mà không tạo ticket). 
- Ticket data 

```json
    ticket_data = {
        "linked_case": case["case_id"],
        "description": case.get("description", "No Title"),
        "title": case.get("title", "No Title"),
        "creator": playbook_name,
        "ticket_type_id": ticket_type["_id"], // from mapping
        "ticket_severity_id": ticket_severity["_id"], // from mapping
        "ticket_status_id": ticket_status["_id"], // from mapping
        "resolution": ticket_resolution, // from mapping
    }
```

```python
def find_assigned_group(case, config, pilot_alert):

    assigned_group = case.get("_assigned_group")

    if assigned_group is None:
        if not pilot_alert:
            return config.get("default_assigned_group")
        assigned_group = (
            pilot_alert.get("partial_group")
            or pilot_alert.get("organization_group")
            or config.get("default_assigned_group")
        )
    return assigned_group

```

```python
def find_assignee(case, pilot_alert, assigned_group):
    if is_se_alert(pilot_alert):
        alert_owner = pilot_alert.get("owner")
        member_list = get_users_in_group(assigned_group)
        if alert_owner in member_list:
            return alert_owner
    else:
        case_assignee = case.get("_assignee")
        if case_assignee:
            return case_assignee
    return None
```