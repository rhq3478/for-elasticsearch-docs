---
layout: default
title: Policies
nav_order: 1
parent: Index State Management
has_children: false
---

# Policies

Policies are JSON documents that define the following:

- The *states* that an index can be in, including the default state for new indices. For example, you might name your states "hot," "warm," "delete," and so on. For more information, see [States](#states).
- Any *actions* that you want the plugin to take when an index enters a state, such as performing a rollover or taking a snapshot. For more information, see [Actions](#actions).
- The conditions that must be met for an index to move into a new state, known as *transitions*. For example, if an index is more than eight weeks old, you might want to move it to the "delete" state. For more information, see [Transitions](#transitions).

In other words, a policy defines the *states* that an index can be in, the *actions* to perform when in a state, and the conditions that must be met to *transition* between states.

This table lists the parameters that you can define in a policy.

Parameter | Description | Type | Required | Read Only
:--- | :--- |:--- |:--- |
`policy_id` |  The name of the policy. | `string` | Yes | No
`description` |  A human-readable description of the policy. | `string` | Yes | No
`schema_version` | The version number of the policy. Increment as you add or remove new fields. | `number` | Yes | No
`last_updated_time`  |  The time the policy was last updated. | `timestamp` | Yes | Yes
`error_notification` |  The destination and message template for error notifications. The destination could be Amazon Chime, Slack, or a webhook URL. | `object` | No | No
`default_state` | The default starting state for each index that uses this policy. | `string` | Yes | No
`states` | The states that you define in the policy. | `nested list of objects` | Yes | No

---

#### Table of contents
1. TOC
{:toc}


---

## States  

A state is the description of the status that the managed index is currently in. A managed index can be in only one state at a time. Each state has associated actions that are executed sequentially on entering a state and transitions that are checked after all the actions have been completed.

This table lists the parameters that you can define for a state.

Parameter | Description | Type | Required
:--- | :--- |:--- |:--- |
`name` |  The name of the state. | `string` | Yes
`actions` | The actions to execute after entering a state. For more information, see [Actions](#actions). | `nested list of objects` | Yes
`transitions` | The next states and the conditions required to transition to those states. If no transitions exist, the policy assumes that it's complete and can now stop managing the index. For more information, see [Transitions](#transitions). | `nested list of objects` | Yes

---

## Actions

Actions are the steps that the policy sequentially executes on entering a specific state.

This table lists the parameters that you can define for an action.

Parameter | Description | Type | Required | Default
:--- | :--- |:--- |:--- |
`timeout` |  The timeout period for the action. Accepts time units for minutes, hours, and days. | `time unit` | No | Specific to action
`retry` | The retry configuration for the action. | `object` | No | Specific to action
`count` | The number of retry counts. | `number` | No | Specific to action
`backoff` | The backoff policy type to use when retrying. | `string` | No | Exponential
`delay` | The time to wait between retries. Accepts time units for minutes, hours, and days. | `time unit` | No | Specific to action

All actions have common `timeout`, `retry`, and `backoff` settings. If you don't specify these settings, ISM uses its defaults.

The following example action has a timeout period of one hour. The policy retries this action three times with an exponential backup policy, with a delay of five seconds between each retry.

```json
"actions": {
    "timeout": "1h",
    "retry": {
        "count": 3,
        "backoff": "exponential",
        "delay": "5s"
    }
}
```

## ISM supported operations
ISM supports the following operations:

- [forcemerge](#forcemerge)
- [read_only](#read_only)
- [read_write](#read_write)
- [replica_count](#replica_count)
- [close](#close)
- [open](#open)
- [delete](#delete)
- [snapshot](#snapshot)
- [rollover](#rollover)
- [notification](#notification)

### forcemerge

Reduces the number of Lucene segments by merging the indices. This operation attempts to set the index to a `read-only` state before starting the merging process.

Parameter | Description | Type | Required | Default
:--- | :--- |:--- |:--- |
`max_num_segments` | The number of segments to reduce the shard to. | `number` | Yes | Null

```json
{
    "forcemerge": {
        "max_num_segments": 1
    }
}
```

### read_only

Sets a managed index to be read only.

```json
{
    "read_only": {}
}
```

### read_write

Sets a managed index to be writeable.

```json
{
    "read_write": {}
}
```

### replica_count

Sets the number of replicas to assign to an index.

Parameter | Description | Type | Required
:--- | :--- |:--- |:--- |
`number_of_replicas` | Defines the number of replicas to assign to an index. | `number` | Yes

```json
{
    "replica_count": {
        "number_of_replicas": 2
    }
}
```

### close

Closes the managed index.

```json
{
    "close": {}
}
```

### open

Opens a managed index.

```json
{
    "open": {}
}
```

### delete

Deletes a managed index.

```json
{
    "delete": {}
}
```

### rollover

Rolls an alias over to a new index when the managed index meets one of the rollover conditions.

The index format must match the pattern: `^.*-\\d+$`. For example, `(logs-000001)`.
Set `index.index_state_management.rollover_alias` as the alias to rollover.

Parameter | Description | Type | Required
:--- | :--- |:--- |:--- |
`min_size` | The minimum size of the primary shard index storage required to roll over. | `number` | No
`min_docs` |  The minimum number of documents required to roll over. | `number` | No
`min_age` |  The minimum age from index creation required to roll over. | `number` | No

```json
{
    "rollover": {
        "min_docs": 100000000
    }
}
```

### notification

Sends you a notification.

Parameter | Description | Type | Required
:--- | :--- |:--- |:--- |
`destination` | The destination URL. | `Slack, Amazon Chime, or webhook URL` | Yes
`message_template` |  The text of the message. | `object` | Yes

```json
{
    "notification": {
        "destination": {
            "chime": {
                "url": "<url>"
                             }
                },
        "message_template": {
            "source": "{% raw %}The index {index_name} is being deleted{% endraw %}"
                }
        }
}
```

---

## Transitions

Transitions define the conditions that need to be met for a state to change. After all actions in the current state are completed, the policy starts checking the conditions for transitions.

This table lists the parameters you can define for transitions.

Parameter | Description | Type | Required
:--- | :--- |:--- |:--- |
`state` |  The name of the state to transition to if the conditions are met. | `string` | Yes
`empty` |  Assumed to be the equivalent of always `true`, meaning that the policy transitions the index to this state the moment it checks. | `string` | No
`min_index_age` | The minimum age of the index required to transition. | `string` | No
`min_doc_count` | The minimum document count of the index required to transition. | `number` | No
`min_size` | The minimum size of the index required to transition. | `string` | No
`cron` | The `cron` job that triggers the transition if no other transition happens first. | `object` | No
`expression` | The `cron` expression that triggers the transition. | `string` | Yes
`timezone` | The timezone that triggers the transition. | `string` | Yes

If no conditions are specified, ISM transitions the index to the state the moment it checks. ISM checks the conditions every 5 minutes by default.

## Example policy

The following example policy implements a `hot`, `warm`, and `delete` workflow.

In this case, an index is initially in a `hot` state. After a day, it changes to a `warm` state, where the number of replicas increases to 5 to improve the read performance.

After 30 days, the policy moves this index into a `delete` state. The service sends a notification to a Chime room that the index is being deleted, and then permanently deletes it.

```json
{
   "policy":{
      "description":"hot warm delete workflow",
      "default_state":"hot",
      "schema_version":1,
      "states":[
         {
            "name":"hot",
            "actions":[
               {
                  "rollover":{
                     "min_age":"1d"
                  }
               }
            ],
            "transitions":[
               {
                  "state_name":"warm"
               }
            ]
         },
         {
            "name":"warm",
            "actions":[
               {
                  "replica_count":{
                     "number_of_replicas":5
                  }
               }
            ],
            "transitions":[
               {
                  "state_name":"delete",
                  "conditions":{
                     "min_index_age":"30d"
                  }
               }
            ]
         },
         {
            "name":"delete",
            "actions":[
               {
                  "notification":{
                     "destination":{
                        "chime":{
                           "url":"<URL>"
                        }
                     },
                     "message_template":{
                        "source":"The index {% raw %}{ctx.index}{% endraw %} is being deleted"
                     }
                  }
               },
               {
                  "delete":{

                  }
               }
            ]
         }
      ]
   }
}
```

This diagram shows the `states`, `transitions`, and `actions` of the above policy as a finite-state machine. For more information about finite-state machines, see [Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine).  

![Policy State Machine](../../images/ism.png)
