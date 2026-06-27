---
layout: post
title: 'Password spraying blacklist bypass'
tags:
  - web
hero: https://images.unsplash.com/photo-1541955193702-9ca03b1bb11a?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1470&q=80
overlay: red
---

Password spraying is a type of brute force attack where an attacker will use a small list of commonly used passwords, that may match the complexity policy of the domain. {: .lead} <!–-break-–>

# Password spraying blacklist bypass

Password spraying had been a form of brute force attack in which a small set of commonly used passwords was tried across many accounts. These passwords were typically chosen so that they still complied with the target domain’s complexity policy, which reduced the likelihood of immediate rejection and increased the chances of successful authentication attempts.

In many modern environments, blacklist mechanisms had been introduced to hinder this type of activity by blocking known malicious patterns, repeated authentication failures, or suspicious login behaviour. However, these controls had been bypassed by distributing requests through infrastructure that masked origin consistency.

## AWS based distribution approach

To circumvent blacklist based detection, an AWS IAM proxy had been used so that authentication attempts were routed through multiple regional netblocks. This approach had effectively fragmented traffic origins, making it more difficult for defensive systems to correlate repeated login attempts as part of a single attack sequence.

A dedicated AWS user had been configured in advance, and it had been assigned the roles AWSServiceRoleForOrganizations, AWSServiceRoleForSupport, and AWSServiceRoleForTrustedAdvisor. These roles had been required in order to ensure sufficient service integration and operational permissions for the proxying mechanism.

## Attack execution using CredMaster

Once the AWS environment had been prepared, the tool CredMaster had been used, available at https://github.com/knavesec/CredMaster. It had supported multiple authentication modules, including OWA and O365, among others, which had made it suitable for testing cloud based authentication resilience.

In this case, the attack had been executed against the O365 environment of the organisation in order to evaluate SOC detection capabilities under controlled conditions. AWS access keys had been supplied to the tool, and two wordlists had been prepared in advance. One list had contained email addresses, while the other had included commonly used passwords aligned with the domain’s password policy.

The execution had then been carried out with a command that specified the O365 plugin, authentication credentials, input files, and concurrency settings. The structure had been as follows:

`python3 credmaster.py --plugin 0365 --access_key XYZ --secret_access_key XYZ -u emails.txt -p passwords.txt -t 10`

Through this configuration, authentication attempts had been distributed and automated, while the underlying AWS routing had contributed to dispersing network origin signals, which had complicated blacklist based detection mechanisms.
