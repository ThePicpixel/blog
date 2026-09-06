---
title: "Designing a Detection-as-Code workflow"
date: 2026-09-02T13:00:20+00:00
hero: /images/logos/opentofu.png
draft: true
menu:
  sidebar:
    name: Detection-as-Code workflow
    identifier: threat-hunting-detection-as-code-workflow
    parent: threat-hunting
    weight: 20
---

In the privous post, we managed to deploy a training lab with an Elastic stack and a tool called [Mocker](https://github.com/ThePicpixel/Mocker). Thanks to this tool, we now can practice on emulated APT attacks!

Unfortunately, we don't have any detection rules or whatsoever right now... Let's remediate that!

## Objectives

The error here would be to think that we only want to design detection rules. We don't. Writing rules is not enough if you want to assess your detection quality.

This is a minimal list of things we need to do: 
- Design rules in a format that can be shared with others
- Automatically lint our rules to prevent any syntax issues
- Validate our rules against unit tests 
- Deploy in Preproduction/Production depending on the situation

Most of these steps can be automated to let us deal with the most important: detection engineering. This is what we will discuss next!

## Designing a detection rule with Sigma

[Sigma](https://github.com/SigmaHQ/sigma) is a repository that gathers more than 3000 detection rules in a YAML format. This generic format can then be convert into different query languages to integrate these rules in different SIEM (Splunk, Elastic, ...).

This is going to be our starting point. I decided to go with the definition of a custom rule rather than using a rule from the `Sigma` repository.

Imagine we want to detect the impersonation of a service account by an attacker on a Windows machine.

Create a file named `proc-creation-suspicious-svc-account.yaml`. A Sigma rule is composed of:
- `metadata`: properties of the rule
- `detection`: the actual logic of the detection
- `logsource`: the source of the event we based our detection on

First let's define the `detection`:
```yaml
detection:
    selection:
        EventID: 4624
        LogonType: 2
        TargetUserName|startswith: 'svc_'
    condition: selection
```

In our case, we want to detect whenever a service account is connecting on a machine in an unusual way, that is with an interactive session. Service accounts usually doesn't need interactive sessions since they are used to authenticate third-party softwares. Therefore, the opening of an interactive session with this account is rather suspicious. When we look at the Windows documentation, the EventID 4624 means a successful authentication and LogonType 2 means that an interactive session has started. To prevent from triggering this rule everytime someone connects to its session, we add a filter on the TargetUserName.

It is very important to understand here that this filter is infrastructure-dependent. Not every company will have that convention of naming service accounts with `svc_<whatever>`. Therefore, even if a `Sigma` rule is supposed to be generic, there is still some fine-tuning to do in order to use `Sigma` rules coming from somewhere else.

This `detection` part suggests that the rule will trigger only when `condition` is true, which here means `selection` is matched. Please refer to [Sigma's documentation for further explanation](https://sigmahq.io/docs/basics/rules.html).


Ok, we have a detection logic now. But wich log source does it need exactly? That's where the `logsource` component gets in!

Add to your rule the following:
```yaml
logsource:
    category: process_creation
    product: windows
```

In this component, you only need to put what is the product you are monitoring (`windows` here), and a `category`. You can find the list of the supported `products`/`categories` [here](https://sigmahq.io/docs/basics/log-sources.html#logsource-types).

Now we have everything we need. We will just add some fancy `metadata` to make it clean:
```yaml
title: Suspicious Interactive Logon by Service Account
id: 5f2b1c8a-6e2d-4b3f-9a1e-3c7d9e2f4a11
status: experimental
description: Detects an interactive logon (type 2) performed by an account following a service-account naming convention.
author: detection-as-code-demo
date: 2026-09-03
references:
    - https://attack.mitre.org/techniques/T1078/
tags:
    - attack.initial-access
    - attack.t1078
falsepositives:
    - Legitimate administrative interactive use of a service account
level: high
```

The `metadata` are pretty self-explanatory but keep in mind that this need to be updated whenever you include this in your SIEM. For instance you may want to set the `level` to whatever you estimate to be the level, or to add some other `falsepositives` scenarios regarding your operational context. The only thing you probably don't to change is the `id`. This `id` is a randomly generated UID that uniquely identify your rule. You may want to reference you rule using this `id` so don't change it. 

Now we have everything we need, your rule should look like this:
```yaml
title: Suspicious Interactive Logon by Service Account
id: 5f2b1c8a-6e2d-4b3f-9a1e-3c7d9e2f4a11
status: experimental
description: Detects an interactive logon (type 2) performed by an account following a service-account naming convention.
author: detection-as-code-demo
date: 2026-09-03
references:
    - https://attack.mitre.org/techniques/T1078/
tags:
    - attack.initial-access
    - attack.t1078
logsource:
    category: process_creation
    product: windows
detection:
    selection:
        EventID: 4624
        LogonType: 2
        TargetUserName|startswith: 'svc_'
    condition: selection
falsepositives:
    - Legitimate administrative interactive use of a service account
level: high
```

Now it would be great to deploy it in our SIEM. But first we need to convert it to our query language and test it!

## Designing a CI/CD pipeline

Remember that the objective is to create a CI/CD pipeline to automate most of the work. The following parts focuses on the automation of the conversion, validation and deployment step. I chose to work with gitlab CI/CD for the purpose of this article.

### Converting our rule

Before we perform any conversion, it is important to check the syntax of our rule to prevent any issue during the process.

Create a `.gitlab-ci.yaml` file, and add the following content:
```yaml
stages:
  - lint

variables:
  PYTHON_IMAGE: python:3.12-slim

.python_sigma_deps: &python_sigma_deps # This will be used by a lot of jobs
  before_script:
    - pip install --quiet sigma-cli pysigma-backend-elasticsearch pyyaml yamllint pytest "elasticsearch>=8.15,<9"
    - sigma plugin install elasticsearch

lint:sigma:
  stage: lint
  image: $PYTHON_IMAGE
  <<: *python_sigma_deps    # include the before_script to install all dependencies
  script:
    - yamllint rules/       # lint all YAML rules
    - sigma check rules/    # check all SIGMA rules syntax
```

This will ensure that the rules are well-written before we perform the conversion.

Now for the conversion part: we need to convert our YAML rules into ES|QL, and then turn them into `terrform` resources. Converting YAML rules to ES|QL is pretty straight forward, just use the `sigma` CLI. The `terraform` part is not so easy. We need a python script to process every rules and transform them in `terraform` resources.

Here is a working script, feel free to change it the way you want/need:
```python3
print("Hello, World!")
```

Now just wire that up in the following job:
```yaml
convert:esql:
  stage: convert
  image: $PYTHON_IMAGE
  <<: *python_sigma_deps    # include the before_script to install all dependencies
  script:
    - mkdir -p build
    - sigma convert -t esql -p ecs_windows --disable-pipeline-check -f siem_rule_ndjson -o build/rules.ndjson rules/
    - python3 tools/adapter.py build/rules.ndjson terraform
  artifacts:
    paths:
      - terraform/*.tf.json
```

Now you should have your rules in the form of `terraform` resources containing ES|QL rules! Let's move on to the validation step!

### Validating our rule

### Deploying our rule
