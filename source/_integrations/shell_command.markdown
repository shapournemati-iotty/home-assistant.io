---
title: Shell Command
description: Instructions on how to integrate Shell commands into Home Assistant.
ha_category:
  - Automation
ha_iot_class: Local Push
ha_release: 0.7.6
ha_quality_scale: internal
ha_codeowners:
  - '@home-assistant/core'
ha_domain: shell_command
ha_integration_type: integration
---

This integration can expose regular shell commands as actions. Actions can be called from a [script] or in [automation].
Shell commands aren't allowed for a camel-case naming, please use lowercase naming only and separate the names with underscores.

Note that the shell command process will be terminated after 60 seconds, full stop. There is no option to alter this behavior, this is by design because Home Assistant is not intended to manage long-running external processes.

[script]: /integrations/script/
[automation]: /getting-started/automation/

## Configuration

```yaml
# Example configuration.yaml entry
# Exposes action shell_command.restart_pow
shell_command:
  restart_pow: touch ~/.pow/restart.txt
```

{% configuration %}
alias:
  description: Give the shell command a name (alias) as a variable and set the command you want to execute after the colon. e.g., `alias`:`the shell command you want to execute`.
  required: true
  type: string
{% endconfiguration %}

The commands can be dynamic, using templates to insert values for arguments. When using templates, shell_command runs in a more secure environment which doesn't allow any shell helpers like automatically expanding the home dir `~` or using pipe symbols to run multiple commands. Similarly, only content after the first space can be generated by a template. This means the command name itself cannot be generated by a template, but it must be literally provided.

Any action data passed into the action to activate the shell command will be available as a variable within the template.

`stdout` and `stderr` output from the command are both captured and will be logged by setting the [log level](/integrations/logger/) to debug.

## Execution

The `command` is executed within the [configuration directory](/docs/configuration/).

{% tip %}
If you are using [Home Assistant Operating System](https://github.com/home-assistant/operating-system), the commands are executed in the `homeassistant` container context. So if you test or debug your script, it might make sense to do this in the context of this container to get the same runtime environment.
{% endtip %}

A `0` exit code means the commands completed successfully without error. In case a command results in a non `0` exit code or is terminated after a timeout of 60 seconds, the result is logged to Home Assistant log.

## Response

Shell commands provide an action response in a dictionary containing `stdout`, `stderr`, and `returncode`. These can be used in automations to act upon the command results using [`response_variable`](/docs/scripts/perform-actions#use-templates-to-handle-response-data).

## Examples

### Defining multiple shell commands

You can also define multiple shell commands at once. This is an example
that defines three different (unrelated) shell commands.

```yaml
# Example configuration.yaml entry
shell_command:
  restart_pow: touch ~/.pow/restart.txt
  call_remote: curl http://example.com/ping
  my_script: bash /config/shell/script.sh
```

### Automation example

This is an example of a shell command used in conjunction with an input
helper and an automation.

{% raw %}

```yaml
# Apply value of a GUI slider to the shell_command
automation:
  - alias: "run_set_ac"
    triggers:
      - trigger: state
        entity_id: input_number.ac_temperature
    actions:
      - action: shell_command.set_ac_to_slider

input_number:
  ac_temperature:
    name: A/C Setting
    initial: 24
    min: 18
    max: 32
    step: 1

shell_command:
  set_ac_to_slider: 'irsend SEND_ONCE DELONGHI AC_{{ states("input_number.ac_temperature") }}_AUTO'
```

{% endraw %}

The following example shows how the shell command response may be used in automations.

{% raw %}

```yaml
# Create a ToDo notification based on file contents
automation:
  - alias: "run_get_file_contents"
    triggers:
      - ...
    actions:
      - action: shell_command.get_file_contents
        data:
          filename: "todo.txt"
        response_variable: todo_response
      - if: "{{ todo_response['returncode'] == 0 }}"
        then:
          - action: notify.mobile_app_iphone
            data:
              title: "ToDo"
              message: "{{ todo_response['stdout'] }}"
        else:
          - action: notify.mobile_app_iphone
            data:
              title: "ToDo file error"
              message: "{{ todo_response['stderr'] }}"


shell_command:
  get_file_contents: "cat {{ filename }}"
```

{% endraw %}
