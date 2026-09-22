# Agentic AI for Security Review: A Measured Test

An evaluation of what an autonomous AI agent actually catches in an infrastructure security review, run locally, scored against a known answer key.

**Headline result:** 16 of 18 planted misconfigurations found, zero false positives, unreliable severity ranking, and fabricated technical justifications that got worse on a second pass over identical input.

---

## What Ornith is

Ornith 1.0 (9B) is a language model built by DeepReinforce AI. It is not a general purpose chatbot. It was trained specifically for agentic coding work: reading codebases, operating in terminals, and producing well formed tool calls that other software can execute.

Its distinguishing feature is what its authors call self scaffolding. During training the model learned to generate not only answers but the reasoning structure that produces them, which is how a relatively small model manages multi step work that normally needs a much larger one. It is MIT licensed, carries a 262,000 token context window, and scores 69.4% on SWE bench Verified, which is strong for its size.

At roughly 5.6 GB when quantised, it runs on ordinary consumer hardware.

## What Hermes is

Hermes Agent, built by Nous Research, is the framework that surrounds a model and turns it into something that can act. It supplies 24 tools, a skills system, persistent memory across sessions, scheduling, sandboxed execution backends, and integrations with roughly twenty messaging platforms.

What Hermes deliberately does not supply is a model. It is an empty shell until pointed at one. That separation is the whole point: the harness and the intelligence are independent choices.

## How the two fit together

The division of labour is clean. Hermes decides what needs doing and provides the means to do it. Ornith decides how, turning intent into concrete tool calls.

In practice the loop runs like this. Hermes sends the model a description of the task and the tools available. Ornith replies with a function call. Hermes executes that call inside its sandbox and feeds the result back. The cycle repeats until the task is finished.

The pairing is unusually well matched. Hermes is built around tool calling, Ornith is trained for tool calling, both are MIT licensed, and both run entirely on local hardware. Nothing needs to leave the machine, which matters a great deal in consulting work where client material cannot be sent to a third party API.

---

## Why this experiment

Agentic AI is being marketed hard to security teams. The claims are that these tools will triage findings, review code, and catch what humans miss. Very little of that arrives with evidence attached.

Our team was asked to experiment with agentic AI for security work, so rather than form an impression, we built something that could produce a number. Two open source components, running entirely on one laptop, pointed at an infrastructure as code file where every correct answer was known in advance.

The question was not whether AI is useful in security. It was narrower and more answerable: what does this class of tool genuinely do well, where does it fail, and how would anyone measure that repeatably.

Everything described below can be reproduced on a laptop. No cloud API, no data leaving the machine, no cost beyond electricity.

---

## Sandboxing comes first

One sentence in the Hermes documentation shaped the entire setup:

> "The agent has the same filesystem access as your user account."

That is the default behaviour. On a working consultant's machine, that user account typically has cached cloud credentials, SIEM configuration, VPN tokens and SSH keys sitting in the home directory. An agent that writes its own skills and browses the web should not have read access to any of it.

So the agent runs inside a Docker container with no host folders mounted at all.

Three choices are worth explaining, because each one is somewhere a person could reasonably go wrong.

**Container rather than VM or host.** A virtual machine offers stronger isolation but adds enough daily friction that people quietly abandon it within a fortnight. Running on the host with a sandboxed terminal backend looks like a sensible middle ground, but the documentation does not state whether the agent's own file tools are routed through that sandbox or still reach the host directly. That ambiguity is exactly what should not be load bearing in a security boundary. A full container leaves no such question open.

**Native Docker volume rather than a bind mount.** On Windows, Docker Desktop runs on WSL2. Bind mounting a Windows path into a container crosses a virtual machine filesystem boundary, and that can corrupt the SQLite databases the agent uses for its state. Agent state belongs on a native volume. Project folders can be bind mounted; the agent's own data cannot.

**Approval gates on self modification.** Hermes writes its own skills and curates its own memory, and by default both happen silently. Three settings change that:

```yaml
skills:
  write_approval: true       # every skill staged for review
  guard_agent_created: true  # scans for credential harvesting, exfiltration
memory:
  write_approval: true       # memory updates need approval too
```

Reviewing staged skills turns out to be the clearest window into what the agent actually decided to do, and it is where a badly generalised rule shows up first.

The model itself runs outside the container, in LM Studio, because it needs direct GPU access. That is fine. LM Studio is a text completion service with no awareness of any filesystem. The boundary that matters sits around the agent, not around the model.

Full configuration is in [`setup/`](setup/).

---

## Verifying the sandbox holds

Before trusting any of it, the agent was asked to look around:

```
ls -la ~, ls /, cat /etc/hostname, whoami
Then search the filesystem for Azure, AWS, or SSH credentials.
```

It returned a container hostname, a plain Linux filesystem, a nearly empty home directory, and an empty handed credential search. It also guessed there might be a `/mnt/c` mount, which is a sensible thing for an agent on a Windows host to try, and found nothing there either.

This check takes about thirty seconds and it is the difference between believing a sandbox works and knowing it does.

---

## The test

An Azure Bicep template was written that deploys storage, networking, a Key Vault and a SQL server. Eighteen specific misconfigurations were planted in it, documented in a separate answer key kept well away from anything the agent could reach.

The template is in [`corpus/azure-bicep-01/target/main.bicep`](corpus/azure-bicep-01/target/main.bicep). The answer key is in [`corpus/azure-bicep-01/ANSWER-KEY.md`](corpus/azure-bicep-01/ANSWER-KEY.md), one directory level above the folder that gets mounted, so it cannot be read by accident.

The planted issues spanned a realistic range of severity: a hardcoded SQL administrator password, a storage account key emitted as a deployment output, a SQL firewall rule spanning 0.0.0.0 to 255.255.255.255, a blob container named `customer-exports` with anonymous public read, NSG rules exposing RDP, SSH and port 1433 to the internet, TLS 1.0 minimums on storage and SQL, a Key Vault with soft delete off and a wildcard access policy on a placeholder GUID, network ACLs defaulting to Allow, and no diagnostic settings or audit logging anywhere.

That last item is deliberately the hardest. Spotting an absence requires knowing what ought to be present, which is a different cognitive task from matching a bad value against a pattern.

Then one prompt, cold:

> Read /workspace/main.bicep and perform a security review. For every issue, give the resource name, the specific property, why it's a risk, and the fix. Rank by severity. Don't summarize the file, I want findings.

---

## Scoring

Both raw outputs from this run are preserved unedited in [`results/2026-09-22-hermes-ornith-1.0-9b/`](results/2026-09-22-hermes-ornith-1.0-9b/).

### Recall: 16 of 18

| Category | Planted | Covered |
|---|---|---|
| Critical | 5 | 5 |
| High | 8 | 8 |
| Medium | 5 | 3 |
| **Total** | **18** | **16** |

### Why the agent reported 17 findings but covered 16 planted issues

The two counts measure different things and the mapping is not one to one.
Seventeen findings were reported. Sixteen of the eighteen planted issues were
covered. The full mapping:

| Answer key item | Agent finding | Note |
|---|---|---|
| 1. Hardcoded SQL password | 3 | One finding covers key items 1 and 2 |
| 2. Password not `@secure()` | 3 | Same finding, notes "no secureParameter" |
| 3. Storage key in output | 17 | Rated LOW, should be critical |
| 4. SQL firewall 0.0.0.0 to 255.255.255.255 | 2 | |
| 5. `customer-exports` public read | 5 | |
| 6. `allowBlobPublicAccess: true` | 4 | |
| 7. `supportsHttpsTrafficOnly: false` | 9 | |
| 8. NSG RDP 3389 from `*` | 6 | |
| 9. NSG SSH 22 from `Internet` | 8 | |
| 10. NSG 1433 from `*` | 7 | |
| 11. `enableSoftDelete: false` | 13 | |
| 12. `enablePurgeProtection: false` | 14 | |
| 13. Wildcard access policy on placeholder GUID | 16 | |
| 14. `enableRbacAuthorization: false` | 15 | Misreported as "unspecified" in the file version |
| 15. `networkAcls.defaultAction: 'Allow'` | — | **Missed** |
| 16. TLS 1.0 on storage and SQL | 10 and 12 | One key item, reported as two findings |
| 17. `publicNetworkAccess: 'Enabled'` on SQL | 1 | |
| 18. No diagnostic settings or audit logging | — | **Missed** |
| *(bonus)* `allowSharedKeyAccess: true` | 11 | Correct catch, not one of the planted 18 |

Reconciling the arithmetic: 17 findings reported, less 1 bonus finding, leaves
16 mapping to the key. One of those covers two key items, which adds one. Two
of them collapse into a single key item, which subtracts one. The result is 16
of 18 planted issues covered.

### The two misses

**`networkAcls.defaultAction: 'Allow'`** on both the storage account and the Key Vault. Not mentioned anywhere in the output. This is a genuine gap, and it pairs directly with the public access findings that were caught.

**No diagnostic settings or audit logging** anywhere in the template. Expected. Absence detection requires knowing what should be present, which is the hardest class of finding for a model to produce.

### False positives: zero

All 17 reported findings map to something genuinely present in the file. One of them, `allowSharedKeyAccess: true`, was on the answer key's bonus list rather than the main eighteen and counts as a correct additional catch.

For a security tool this is the number that matters most. A tool that invents findings costs more analyst time than it saves.

### Severity disagreements

| Finding | Agent rated | Should be | Why |
|---|---|---|---|
| Storage account key in deployment output | LOW | Critical | Live credential, retrievable by anyone with resource group read access, persists in deployment history, grants full control of the storage account |

One disagreement out of seventeen, but it is the worst possible one to get wrong. Anyone triaging by the agent's ranking would have deferred the single finding that leaks a working key.

### Groundedness

Line number citations were checked against the template. Nearly all are correct. One is off by one: `enableRbacAuthorization` is cited at line 142 and sits at line 141.

So the agent knows where things are. Whether it knows what they mean is a separate question, addressed next.

---

## The three failure modes

### Severity ranking is unreliable

Covered above. The pattern generalises beyond this one miscall: the tool finds the right things and judges them poorly.

### The justifications are confidently wrong

The findings themselves were correct. The explanations attached to them frequently were not:

- Anonymous public blob access described as allowing "read, write, and delete". It is read only.
- TLS 1.0 described as allowing encrypted connections to be "downgraded to plaintext". TLS 1.0 is weak, but it is not plaintext.
- ZeroLogon cited as an RDP exploit. It is a Netlogon flaw against domain controllers.
- EternalBlue cited as a SQL Server risk, identified as "MS17-014". EternalBlue is MS17-010 and it is an SMB vulnerability.
- Port 1433 exposure described as enabling "SQL injection attacks from outside the network perimeter". SQL injection is an application layer flaw unrelated to network exposure.
- Log4Shell and Cobalt Strike beacons listed as SSH brute force risks. Neither belongs there.
- A remediation snippet suggesting `reference(extensionData.sqlAdminPassword)`, which is not valid Bicep syntax.

An experienced practitioner discards all of these instantly. A junior analyst pastes them into a report.

### The written report drifted from the live output

This is the most consequential finding, and the one this repository exists to document.

Having produced the review in the session, the agent was asked to write the same review to a file. The file was not a transcript. It regenerated the content, and the regenerated version was worse.

One finding became factually wrong. In the session the agent correctly reported `enableRbacAuthorization: false`. In the file that became "unspecified". The template sets the value explicitly, so the written report misdescribes its own source.

Three fabrications appeared in the file that had not been in the session output: the EternalBlue reference, the Log4Shell and Cobalt Strike reference, and the SQL injection claim. Every error from the first pass also persisted into the second.

One thing did improve. The file cited specific line numbers, and nearly all are correct.

Both artefacts are in [`results/`](results/) so this comparison can be verified rather than taken on trust. [`session-output.md`](results/2026-09-22-hermes-ornith-1.0-9b/session-output.md) ends with a line by line diff of the two.

The finding list is stable across runs. The reasoning around it is not, and asking for more detail produced more fabrication rather than more accuracy.

---

## What this means in practice

**Use these tools to find, not to judge.** The finding list carries real value. Severity assignment needs a human every time.

**Generated rationale should never reach a deliverable.** Two runs on identical input produced different technical claims, and the second was worse than the first. This is not a prompting problem that tuning fixes. It is a property of the tool at this scale.

**Confidence is not calibrated to accuracy.** The claim about MS17-014 was stated in exactly the same tone as the correct findings. Nothing in the writing distinguishes the two.

**The architecture recommendation follows directly.** Any deployment of this class of tool needs a human severity review in the loop, and its output should be treated as a triage list rather than an analysis. That is an architectural finding rather than a product review, and it will hold for the next model as well as this one.

**The weaknesses observed are the kind that scale addresses.** Recall came from pattern recognition, which small models handle well. Judgment, factual accuracy and calibration are what additional parameters buy. A 20B or 70B model would likely surface a similar set of findings and explain them correctly, which is the obvious next experiment.

---

## Reproducing this

Docker Desktop, LM Studio, and roughly an hour. Detailed steps are in [`setup/SETUP.md`](setup/SETUP.md).

1. Load Ornith in LM Studio and set the context length to at least 64,000. Hermes enforces a hard floor there and refuses to start below it. Avoid maximising it either, because an oversized KV cache spills out of VRAM and causes the agent to time out waiting for responses.
2. Enable "Serve on Local Network" in LM Studio. Without it the server listens only on localhost and the container cannot reach it. This is the single most common setup failure.
3. Run Hermes in Docker with its state on a native volume, no host folders mounted, ports bound to localhost only, and the approval gates enabled.
4. Verify the sandbox before trusting it.
5. Mount test material read only. The agent has no reason to modify the thing it is auditing.
6. Build a seeded file with an answer key. This is the step that turns an anecdote into a measurement.

### Run configuration

| Setting | Value |
|---|---|
| Model | `ornith-1.0-9b`, Q4_K_M |
| Serving | LM Studio 0.4.24, OpenAI compatible endpoint on port 1234 |
| Context length | 65,000 |
| Parallel slots | 1 |
| GPU offload | 24 layers |
| Harness | Hermes Agent v0.21.3, `nousresearch/hermes-agent:latest` |
| Terminal backend | local, inside the container |
| Host | Windows, Docker Desktop on WSL2 |

Roughly 14,000 tokens of the context window are consumed by the Hermes system prompt before any conversation begins.

---

## Honest limitations

One model, one harness, one artefact, one file type. That is an anecdote with a methodology attached rather than a benchmark.

Turning it into a benchmark means separating the variables: the same model under a different harness, and a different model under the same harness. It also means widening the corpus to Terraform, Kubernetes manifests, log triage and detection rules. That work is the next phase, and the answer key approach scales to all of them.

It is also worth stating plainly that neither component's marketing claims were tested here. What was measured is how the pairing behaved on one task. Everything else remains unverified.

---

## Repository contents

```
setup/           Docker compose, Hermes config, setup instructions
corpus/          Test cases. Each has target/ (mounted read only) and an answer key beside it
results/         Raw outputs per run, unedited, named by date and configuration
```

> **Warning:** the template in `corpus/azure-bicep-01/target/` is deliberately insecure and exists only as test material. Do not deploy it.

---

*Tools used: [Hermes Agent](https://hermes-agent.nousresearch.com/) by Nous Research (MIT) and [Ornith-1.0-9B](https://huggingface.co/ornith-ai/Ornith-1.0-9B) by DeepReinforce AI (MIT), running locally via LM Studio and Docker.*
