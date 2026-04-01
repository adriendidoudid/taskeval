# taskeval

## Installation

```text
mkdir -p ~/.openclaw/skills
git clone https://github.com/adriendidoudid/taskeval ~/.openclaw/skills/taskeval
openclaw gateway restart
```

## Add taskeval to a task

```text
Edit agent [AGENT_NAME], task [TASK_NAME]: if the task does not already end with /taskeval (from skill taskeval), add /taskeval as the final step. Leave everything else unchanged.
```

## Delete taskeval from a task

```text
Edit agent [AGENT_NAME], task [TASK_NAME]: if the task ends with /taskeval (from skill taskeval), remove that final /taskeval. Leave everything else unchanged.
```