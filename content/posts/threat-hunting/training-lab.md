---
title: "Deploy your Threat Hunting training lab"
date: 2026-08-31T13:00:20+00:00
hero: /images/logos/elastic.png
menu:
  sidebar:
    name: Training Lab
    identifier: threat-hunting-training-lab
    parent: threat-hunting
    weight: 10
---

Lately, I was trying to find good free materials to test and improve my Threat Hunting skills. Unfortunately, I couldn't find anything that suits me. Here is what I was looking for:
- a platform that hosts a basic ELK stack
- a platform that can easily be destroyed and rebuilt
- a platform that exposes API to deploy my own set of rules (Detection-as-Code)
- a platform with good benign activity simulation
- a platform with good APT scenarios

Since nothing convinced me, here is the setup I use, with my own open-source tools (and a bit of Claude Code here and there).

My stack consists of:
- an automated deployment of a configurable EFK stack (I prefer fluentd over logstash, sorry)
- a tool to generate event logs

## Setup Sparassidae

### Installation

[Sparassidae](https://github.com/ThePicpixel/Sparassidae) is a collection of scripts and configuration to deploy a local Threat Hunting lab, composed of:
- `elasticsearch`
- `kibana`
- `fluentd`

The lab is entirely containerized, therefore you can set it up and destroy it without any cost.

To install it:
```bash
git clone git@github.com:ThePicpixel/Sparassidae.git
chmod +x deploy.sh setup.sh
./setup.sh   # Install prerequisites
./deploy.sh  # Deploy the lab
```

The script handles everything for you, including the TLS configuration of the Elasticsearch cluster.

The script also outputs credentials for the default user and an API token for Terraform in order to start Detection-as-Code right away! (Look for the `out` folder after the script is finished)

### Configuration

You can configure the deployment of the lab by editing the `docker-compose.yml` file.

You can configure the fluentd ingestion pipeline by editing the `fluent.conf` file.

## Setup Mocker

[`Mocker`](https://github.com/ThePicpixel/Mocker) is a tool that generates benign log activity. Use it to forward these logs to your `Sparassidae` environment.

`Mocker` can also inject APT campaign logs to simulate an attack. Therefore, you can train on a variety of scenarios!

### Installation

To install `Mocker`, run the following commands:

```bash
git clone git@github.com:ThePicpixel/Mocker.git
pip install -e ".[dev]"
```

### Usage

You can specify your organization in the `org.yaml` file:
```yaml
employees:
  - name: "Alice Chen"
    email: "alice.chen@example-corp.com"
    iam_user: "alice.chen"
    k8s_subject: "alice.chen@example-corp.com"
  - name: "Bilal Rana"
    email: "bilal.rana@example-corp.com"
    iam_user: "bilal.rana"
    k8s_subject: "bilal.rana@example-corp.com"
  - name: "Chidi Okoro"
    email: "chidi.okoro@example-corp.com"
    iam_user: "chidi.okoro"
    k8s_subject: "chidi.okoro@example-corp.com"
  - name: "Dana Kowalski"
    email: "dana.kowalski@example-corp.com"
    iam_user: "dana.kowalski"
    k8s_subject: "dana.kowalski@example-corp.com"

aws_accounts:
  - account_id: "111122223333"
    region: "us-east-1"
    iam_users: ["alice.chen", "bilal.rana", "chidi.okoro", "dana.kowalski"]
    s3_buckets:
      - "example-corp-app-logs"
      - "example-corp-backups"
      - "example-corp-data-lake"
    ec2_instance_ids:
      - "i-0a1b2c3d4e5f60789"
      - "i-0f9e8d7c6b5a41230"
      - "i-0123456789abcdef0"
    access_keys:
      alice.chen: "AKIAEXAMPLE0000001"
      bilal.rana: "AKIAEXAMPLE0000002"
      chidi.okoro: "AKIAEXAMPLE0000003"
      dana.kowalski: "AKIAEXAMPLE0000004"
    iam_roles: ["example-corp-admin-role"]
    lambda_functions: ["billing-webhook-processor"]
    codebuild_projects: ["example-corp-web-deploy"]
    s3_objects:
      example-corp-app-logs:
        - "2026-08-01/app.log"
        - "2026-08-02/app.log"
      example-corp-backups:
        - "db/prod-snapshot-2026-08-15.sql.gz"
      example-corp-data-lake:
        - "customers/2026-08-01.parquet"
        - "customers/2026-07-01.parquet"

eks_clusters:
  - name: "prod-cluster"
    account_id: "111122223333"
    namespaces: ["default", "payments", "checkout", "kube-system"]
    pods:
      - "payments/api-7d9f8c6b5-abcde"
      - "checkout/worker-6b7d8f9c5-fghij"
      - "default/gateway-5c8d7b6a4-klmno"
    service_accounts:
      - "payments/payments-sa"
      - "checkout/checkout-sa"
      - "default/default"

network:
  internal_cidr: "10.42.0.0/16"
  saas_ips: ["203.0.113.10", "203.0.113.20", "203.0.113.30"]
  corporate_egress_ips: ["203.0.113.5"]
  attacker_ips: ["198.51.100.23", "198.51.100.47"]
  attacker_aws_account_id: "999988887777"

```

That way, you can simulate consistently the benign lifecycle of your environment.

Then, simply run:
```bash
mocker start --org org.yaml --fluentd-url http://localhost:9880
```

And watch your logs in your SIEM!

![Mocker logs in Kibana](/images/posts/threat-hunting-training-lab/mocker-in-sparassidae.png)

You can see what event sources are active by running:
```bash
mocker status
```
```json
{
  "generators": [
    {
      "tag": "mocker-aws-cloudtrail",
      "rate_per_sec": 2.0,
      "running": true
    },
    {
      "tag": "mocker-k8s-audit",
      "rate_per_sec": 2.0,
      "running": true
    },
    {
      "tag": "mocker-network-flow",
      "rate_per_sec": 2.0,
      "running": true
    }
  ],
  "active_campaign": null
}
```

You can list all available scenarios by running:
```bash
mocker list-scenarios
```
```json
{
  "iam-key-k8s-pivot-exfil": [
    "T1078.004",
    "T1580",
    "T1528",
    "T1610",
    "T1041"
  ],
  "logging-disabled-exfil": [
    "T1078.004",
    "T1562.008",
    "T1580",
    "T1537"
  ],
  "post-incident-cleanup": [
    "T1078.004",
    "T1580",
    "T1531",
    "T1070.004"
  ],
  "s3-ransomware": [
    "T1078.004",
    "T1530",
    "T1486",
    "T1485"
  ],
  "iam-privesc-chain": [
    "T1078.004",
    "T1548.005",
    "T1098.001",
    "T1537"
  ],
  "imds-ssrf-privesc": [
    "T1190",
    "T1552.005",
    "T1619"
  ],
  "eks-cryptojacking": [
    "T1078.004",
    "T1580",
    "T1528",
    "T1610",
    "T1496",
    "T1562.001"
  ],
  "cicd-supply-chain": [
    "T1195.002",
    "T1078.004",
    "T1580",
    "T1528",
    "T1610",
    "T1041"
  ],
  "lambda-backdoor-persistence": [
    "T1078.004",
    "T1098.001",
    "T1525",
    "T1567.002"
  ]
}
```

To start a scenario, run the following with the name of the scenario you want to run:
```bash
mocker inject imds-ssrf-privesc
```
```json
{
  "status": "injecting",
  "campaign": "imds-ssrf-privesc"
}
```

Finally, stop `Mocker` with:
```bash
mocker stop
```
```json
{
  "status": "stopping"
}
```

### Customize Mocker

If you want to customize `Mocker` with new event sources or new scenarios, follow the instructions in the [README](https://github.com/ThePicpixel/Mocker) to create your own!

## Conclusion

You are now ready to play around with your training lab! Everything should be up and running for you to test your new detection techniques, CTI witchcraft and so on!

If you'd like to contribute to `Sparassidae` or `Mocker` with generators or APT campaigns that you designed, feel free to open a Pull Request on GitHub!