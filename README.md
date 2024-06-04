# Octoscan

Octoscan is a static vulnerability scanner for GitHub action workflows.

## Table of Contents

- [Octoscan](#octoscan)
	- [Table of Contents](#table-of-contents)
	- [usage](#usage)
		- [download remote workflows](#download-remote-workflows)
		- [analyze](#analyze)
	- [rules](#rules)
		- [dangerous-checkout](#dangerous-checkout)
		- [dangerous-action](#dangerous-action)
		- [dangerous-write](#dangerous-write)
		- [expression-injection](#expression-injection)
		- [runner-label](#runner-label)
		- [repo-jacking](#repo-jacking)
		- [unsecure-commands](#unsecure-commands)
		- [credentials](#credentials)
		- [shellcheck](#shellcheck)
		- [local-action](#local-action)
		- [oidc-action](#oidc-action)
	- [Credits](#credits)
	- [Resources](#resources)

## usage

### download remote workflows

Octoscan can be run against a local git repository or you can download all the workflows with the `dl` action:
```sh
$ octoscan dl -h  
Octoscan.  

Usage:
	octoscan dl [options] --org <org> [--repo <repo> --token <pat> --default-branch --path <path> --output-dir <dir>]

Options:
	-h, --help  						Show help
	-d, --debug  						Debug output
	--verbose  							Verbose output
	--org <org> 						Organizations to target
	--repo <repo>						Repository to target
	--token <pat>						GHP to authenticate to GitHub
	--default-branch  					Only download workflows from the default branch
	--path <path>						GitHub file path to download [default: .github/workflows]
	--output-dir <dir>					Output dir where to download files [default: octoscan-output]
```

```sh
./octoscan dl --token ghp_<token> --org apache --repo incubator-answer
```

### analyze

If you don't know what to run just run this:
```sh
./octoscan scan path/to/repos/ --disable-rules shellcheck,local-action --filter-triggers external
```

It will reduce false positives and give the most interesting results.


```sh
$ octoscan scan -h
octoscan

Usage:
	octoscan scan [options] <target>
	octoscan scan [options] <target> [--debug-rules --filter-triggers=<triggers> --filter-run --ignore=<pattern> ((--disable-rules | --enable-rules ) <rules>) --config-file <config>]

Options:
	-h, --help
	-v, --version
	-d, --debug
	--verbose
	--json                    				JSON output
	--oneline                    			Use one line per one error. Useful for reading error messages from programs

Args:
	<target>								Target File or directory to scan
	--filter-triggers <triggers>			Scan workflows with specific triggers (comma separated list: "push,pull_request_target")
	--filter-run							Search for expression injection only in run shell scripts.
	--ignore <pattern>						Regular expression matching to error messages you want to ignore.
	--disable-rules <rules>					Disable specific rules. Split on ","
	--enable-rules <rules>					Enable specific rules, this with disable all other rules. Split on ","
	--debug-rules							Enable debug rules.
	--config-file <config>					Config file.

Examples:
	$ octoscan scan ci.yml --disable-rules shellcheck,local-action --filter-triggers external

```

## rules

### dangerous-checkout

#### description

Triggers like `workflow_run` or `pull_request_target` run in a privileged context, as they have read access to secrets and potentially have write access on the targeted repository. Performing an explicit checkout on the untrusted code will result in the attacker code being downloaded in such context.

![excalidraw](img/excalidraw.png)

#### examples

- [FreeRDP](https://github.com/FreeRDP/FreeRDP/pull/10209)
- Excalidraw (Blog post incoming)
- AutoGPT (Blog post incoming)
- Cypress (Blog post incoming)
- Apache Doris (Blog post incoming)
- [Angular](https://github.com/angular/angular/blob/6b20561e1d6810e867c0ee7692d9fae64426a876/.github/workflows/ci-privileged.yml#L4)

### dangerous-action

#### description

This rules warn the user if a dangerous action is used. It's mainly focused on untrusted artifacts.

It is common practice to use artifacts to pass data between different workflows. We often encounter this with the `workflow_run` trigger where the triggering workflow will prepare some data that will then be sent to the triggered workflow. Given the untrusted nature of this artifact data, it is crucial to treat it with caution and recognize it as a potential threat. The vulnerability arises from the fact that external entities, such as malicious actors, can influence the content of the artifact data.

![ant-design](img/ant-design.png)


#### examples

- ant-design (Blog post incoming)
- Swagger-editor (Blog post incoming)
- Firebase (Blog post incoming)
- [Firebase](https://www.legitsecurity.com/blog/github-privilege-escalation-vulnerability-0)
- [Rust](https://www.legitsecurity.com/blog/artifact-poisoning-vulnerability-discovered-in-rust)

### dangerous-write

#### description

GitHub will create default environment variables that can be used inside every step in a workflow. The `GITHUB_ENV` and `GITHUB_OUTPUT` variables are particularly interesting. It is possible to define environment variable in a step and to use this variable in another one. This can be done by writing it to the associated variable variable. If a user can control the content of the variable that is being set it can lead to arbitrary code execution.

![swagger](img/swagger.png)

#### examples

- Swagger-editor (Blog post incoming)
- dgraph-io/badger
- [Firebase](https://www.legitsecurity.com/blog/github-privilege-escalation-vulnerability-0)
- [microsoft/vscode-github-triage-actions](https://bugs.chromium.org/p/project-zero/issues/detail?id=2070)

### expression-injection

Each workflow trigger comes with an associated GitHub context, offering comprehensive information about the event that initiated it. This includes details about the user who triggered the event, the branch name, and other relevant contextual information. Certain components of this event data, such as the base repository name, or pull request number, cannot be manipulated or exploited for injection by the user who initiated the event (e.g., in the case of a pull request). This ensures a level of control and security over the information provided by the GitHub context during workflow execution.

However, some elements can be controlled by an attacker and should be sanitized before being used. Here is the list of such elements:
- `github.event.issue.title`
- `github.event.issue.body`
- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.comment.body`
- `github.event.review.body`
- `github.event.pages.*.page_name`
- `github.event.commits.*.message`
- `github.event.head_commit.message`
- `github.event.head_commit.author.email`
- `github.event.head_commit.author.name`
- `github.event.commits.*.author.email`
- `github.event.commits.*.author.name`
- `github.event.pull_request.head.ref`
- `github.event.pull_request.head.label`
- `github.event.pull_request.head.repo.default_branch`
- `github.head_ref`
- `env.*`
- `steps.*.outputs.*`
- `needs.*.outputs.*`

![autogpt](img/autogpt.png)

#### examples

- AutoGPT (Blog post incoming)
- microsoft/generative-ai-for-beginners (Blog post incoming)

### runner-label

GitHub offers the possibility to host your own runners and customize the environment used to run jobs in workflows. These runners are called self-hosted.

There exists two types of self-hosted runners, ephemeral and non-ephemeral ones. By default, the runners are non-ephemeral, meaning the environment used by the runner is not cleaned after a job completes. If attackers manages to execute code on a non-ephemeral runner, they could backdoor it by adding a process in the background and steal sensitive secrets. These kinds of runners are thus really sensitive.

![excalidraw](img/haskell.png)

Non-ephemeral runners can be identified by looking at run logs. A tool called [gato](https://github.com/praetorian-inc/gato) can be used to automate this process.

#### examples

- Haskell (Blog post incoming)
- lovell/sharp (Blog post incoming)
- WasmEdge
- Scroll (Blog post incoming)
- Akash Network (Blog post incoming)
- [actions/runner-images](https://adnanthekhan.com/2023/12/20/one-supply-chain-attack-to-rule-them-all)
- [tensorflow](https://www.praetorian.com/blog/tensorflow-supply-chain-compromise-via-self-hosted-runner-attack/)
- [pytorch](https://johnstawinski.com/2024/01/11/playing-with-fire-how-we-executed-a-critical-supply-chain-attack-on-pytorch/)


### repo-jacking

#### description
The repo jacking vulnerability was [presented](https://media.defcon.org/DEF%20CON%2031/DEF%20CON%2031%20presentations/Asi%20Greenholts%20-%20The%20GitHub%20Actions%20Worm%20Compromising%20GitHub%20repositories%20through%20the%20Actions%20dependency%20tree.pdf) at DEFCON 31 by Asi Greenholts. This vulnerability occurs when a GitHub action is referencing an action on a non-existing GitHub organization or user.

![AcalaNetwork](img/AcalaNetwork.png)

#### examples

- Azure/bicep-registry-modules (Blog post incoming)
- [HangfireIO/Hangfire](https://www.paloaltonetworks.com/blog/prisma-cloud/github-actions-worm-dependencies/)


### unsecure-commands

#### Description

Actions possess the capability to interact with the runner machine, enabling them to set environment variables, define output values for use by other actions, incorporate debug messages into output logs, and perform various other tasks. However, before 2020, it was possible to control environment variables by writing data to `STDOUT` like this:

```yaml
run: |
   echo "##[set-env name=ENV_NAME;]value"
   # or
   echo "echo "::set-env name=ENV_NAME::value"
```

The implemented workflow commands were inherently insecure due to the common practice of logging to STDOUT. This vulnerability opened avenues for potential attacks, allowing malicious payloads to be easily injected and trigger the set-env command. The ability to modify environment variables introduced multiple paths for remote code execution, with a particularly obvious payload being the one demonstrated earlier. This vulnerability was initially [reported](https://bugs.chromium.org/p/project-zero/issues/detail?id=2070) by a security researcher from Project Zero.

Although the set-env command is deprecated and unusable by default, if a developer sets the `ACTIONS_ALLOW_UNSECURE_COMMANDS` environment variable in a workflow, the `set-env` command becomes available and can be used:

![alibaba](img/alibaba.png)

#### examples

- alibaba/nacos


### credentials

#### description

Checks for credentials in `services:` configuration. This rules comes from [actionlint](https://github.com/rhysd/actionlint).

![test](img/test.png)

### shellcheck

#### description

Run shellcheck on all the bash tasks. This rules comes from [actionlint](https://github.com/rhysd/actionlint).

### local-action

#### description

Raise an alert if a local GitHub action is used. For now the tool can't parse local action files so an alert is raised as they can also contain vulnerabilities.

### oidc-action

#### description

Detect OIDC actions. Workflows using OIDC actions can be a good target to access some cloud providers. There is no vulnerability associated with this rule but taking a closer look at this action can be interesting if there is a vulnerability that is not found by this tool.

## Credits

This tool could not have been developed without [actionlint](https://github.com/rhysd/actionlint). Many thanks to [@rhysd](https://github.com/rhysd).

## Resources

- https://0xn3va.gitbook.io/cheat-sheets/ci-cd/github/actions
- https://cloud.hacktricks.xyz/pentesting-ci-cd/github-security
