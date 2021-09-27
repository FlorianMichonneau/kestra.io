---
order: 1
---

# Use variables

You will find here some examples in order to understand how variables exists and how get the value you need.

Here is a typical payload for variables.

```yaml
globals:
  my-global-string: string
  my-global-int: 1
  my-global-bool: true

task:
  id: float
  type: io.kestra.core.tasks.debugs.Return

taskrun:
  id: 5vPQJxRGCgJJ4mubuIJOUf
  startDate: 2020-12-18T12:46:36.018869Z
  attemptsCount: 0
  value: value2

parent:
  taskrun:
    value: valueA
  outputs:
    int: 1

parents:
  - taskrun:
      value: valueA
    outputs:
      int: 1
  - taskrun:
      value: valueB
    outputs:
      int: 2

flow:
  id: inputs
  namespace: io.kestra.tests

execution:
  id: 42mXSJ1MRCdEhpbGNPqeES
  startDate: 2020-12-18T12:45:28.489187Z

outputs:
  my-task-id-1: # standard task outputs
    value: output-string
  my-task-id-2: # standard task outputs
    value: 42
  my-each-task-id: # dynamic task (each)
    value1: # outputs for value1
      value: here is value1
    value2: # outputs for value2
      value: here is value2

inputs:
  file: kestra:///org/kestra/tests/inputs/executions/42mXSJ1MRCdEhpbGNPqeES/inputs/file/application.yml
  string: myString
  instant: 2019-10-06T18:27:49Z
```


## Common variables
As you can see this a lot of common variable that can be used in your flow, some most common examples are : <code v-pre>{{ execution.id }}</code>, <code v-pre>{{ execution.startDate }}</code> that allow you to change file name, sql query for example.

## Inputs variables
Inputs variables are simply accessible with <code v-pre>{{ execution.NAME }}</code>, with `NAME` is the name of the declared in your flow. The data will be depending of `type` of the inputs.
Special case are `FILE` type where the file will be prepend by `kestra://` that mean file inside the internal Kestra storage. Most of task will take as property this kind of uri and will output also this one. These allow and full file generated by one task to be used in another tasks.

## Outputs variables
Most important in Kestra is the ability to use all outputs from previous tasks in next one.

### Without dynamic tasks (Each)
This is the most common way and the simplest one. In order to get a variables, just use <code v-pre>{{ outputs.ID.NAME }}</code> where :
* `ID` is the task id
* `NAME` is the name of the outputs, each task type can have any outputs that is documentated on the part outputs of their docs. For example, [Bash task](/plugins/core/tasks/scripts/io.kestra.core.tasks.scripts.Bash.html#outputs) can have <code v-pre>{{ outputs.ID.exitCode }}</code>, <code v-pre>{{ outputs.ID.outputFiles }}</code>, <code v-pre>{{ outputs.ID.stdErrLineCount }}</code>, ...

### With dynamic tasks (Each)
This case are more complicated since Kestra will change the way the outputs are generated, since you multiple task with the same id, you will need to use <code v-pre>{{ outputs.ID.VALUE.NAME }}</code>.

But most of the time, using Dynamic Task, you will need to fetch the current value of the iteration, this is done easily with <code v-pre>{{ taskrun.value }}</code>.

But what about if I have a more complex flow, for example, an each containning 1 task (`t1`) to download a file (based on each value), and a second one (`t2`) that need the output of `t1`. The flow look like :

```yaml
id: each-sequential-nested
namespace: io.kestra.tests

tasks:
  - id: each
    type: io.kestra.core.tasks.flows.EachSequential
    value: '["s1", "s2", "s3"]'
    tasks:
      - id: t1
        type: io.kestra.core.tasks.debugs.Return
        format: "{{task.id}} > {{taskrun.value}}"
      - id: t2
        type: io.kestra.core.tasks.debugs.Return
        format: "{{task.id}} > {{ (get outputs.t1 taskrun.value).value }} > {{taskrun.startDate}}"
  - id: end
    type: io.kestra.core.tasks.debugs.Return
    format: "{{task.id}}"
```

In this case, you need to use <code v-pre>{{ (get outputs.t1 taskrun.value).value }}</code>, meaning give me from `outputs.t1` the index `taskrun.value`

### With [Flowable Task](docs/developer-guide/flowable) nested.
If you have many [Flowable Task](docs/developer-guide/flowable), it can be complex to use the `get` function, and more, the `taskrun.value` is only available during the direct task from each, if you have any Flowable after, the `taskrun.value` of the first iteration will be lost (or overwrite). In order to deal this, we have include the `parent` & `parents` vars.

Look at this flow :

```yaml
id: each-switch
namespace: io.kestra.tests

tasks:
  - id: t1
    type: io.kestra.core.tasks.debugs.Return
    format: "{{task.id}} > {{taskrun.startDate}}"
  - id: 2_each
    type: io.kestra.core.tasks.flows.EachSequential
    value: '["a", "b"]'
    tasks:
      # Switch
      - id: 2-1_switch
        type: io.kestra.core.tasks.flows.Switch
        value: "{{taskrun.value}}"
        cases:
          a:
            - id: 2-1_switch-letter-a
              type: io.kestra.core.tasks.debugs.Return
              format: "{{task.id}}"
          b:
            - id: 2-1_switch-letter-b
              type: io.kestra.core.tasks.debugs.Return
              format: "{{task.id}}"

            - id: 2-1_each
              type: io.kestra.core.tasks.flows.EachSequential
              value: '["1", "2"]'
              tasks:
              - id: 2-1-1_switch
                type: io.kestra.core.tasks.flows.Switch
                value: "{{taskrun.value}}"
                cases:
                  1:
                    - id: 2-1-1_switch-number-1
                      type: io.kestra.core.tasks.debugs.Return
                      format: "{{parents.[0].taskrun.value}}"
                  2:
                    - id: 2-1-1_switch-number-2
                      type: io.kestra.core.tasks.debugs.Return
                      format: "{{parents.[0].taskrun.value}} {{parents.[1].taskrun.value}}"
  - id: 2_end
    type: io.kestra.core.tasks.debugs.Return
    format: "{{task.id}} > {{taskrun.startDate}}"

```

As you can see, the `parent` will give the direct accès to the first parent output and value of the current one and `parents.INDEX` let go you deeper on the tree.

In the task `2-1-1_switch-number-2`:
- <code v-pre>{{taskrun.value}}</code>: mean the value of the task `2-1-1_switch`
- <code v-pre>{{parents.[0].taskrun.value}}</code> or <code v-pre>{{parent.taskrun.value}}</code>: mean the value of the task `2-1_each`
- <code v-pre>{{parents.[1].taskrun.value}}</code>: mean the value of the task `2-1_switch`
- <code v-pre>{{parents.[2].taskrun.value}}</code>: mean the value of the task `2_each`