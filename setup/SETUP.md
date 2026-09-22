# Hermes Agent — sandboxed setup on Windows

Fully local models, no host folders mounted, no messaging integrations.

---

## Before you start

You need Docker Desktop running (WSL2 backend) and LM Studio serving a model.

## 0. Set up LM Studio first

Do this before touching Docker — it is the step most likely to cost you time.

1. Open LM Studio and load the model you want to use.
2. Go to the **Developer** tab (called **Local Server** in older versions).
3. Turn the server **on**. Note the port — it is `1234` by default.
4. Enable **"Serve on Local Network"**. This is the critical one. Without it
   LM Studio listens only on `127.0.0.1`, which a container cannot reach, and
   Hermes will fail with a connection error that does not explain itself.
5. Copy the **model identifier** shown next to the loaded model. It usually
   looks like a publisher/model path. You need it exactly.

Sanity check it from PowerShell before going further:

    curl http://localhost:1234/v1/models

You should get JSON listing your loaded model. If you get nothing, the server
is not running.

---

## 1. Edit config.yaml

Open `config.yaml` in Notepad and replace the placeholder on the `model:` line
with the model identifier you copied from LM Studio.

If you changed LM Studio's port from 1234, update `base_url` to match.

Leave everything else alone for now.

---

## 2. Start the container

From this folder, in PowerShell:

    docker compose up -d

First run pulls the image, which takes a few minutes.

---

## 3. Place the config inside the volume

The volume starts empty, so copy the config in and restart:

    docker cp config.yaml hermes:/opt/data/config.yaml
    docker restart hermes

---

## 4. Talk to it

Interactive CLI inside the running container:

    docker exec -it hermes /opt/hermes/.venv/bin/hermes

Or open the dashboard at http://127.0.0.1:9119

---

## Verifying the sandbox actually holds

Do this before you trust it with anything. In the Hermes CLI, ask it to run:

    ls /
    ls ~
    cat /etc/hostname

You should see a Linux container filesystem and a random hostname — not
`C:\Users\<your-username>`. Then ask it to find your Azure credentials. It should come up
empty. If it finds anything belonging to you, stop and re-check that no bind
mount crept into the compose file.

---

## Reviewing what it teaches itself

With `write_approval: true`, nothing it learns persists until you say so.

    /skills pending
    /skills approve <name>
    /skills reject <name>

Read these. The staged skills are the most interesting window into what the
agent actually decided to do, and it is where a bad generalization shows up
first. Once you have reviewed a dozen and they all look sane, consider setting
`memory.write_approval: false` to cut the noise — but leave the skills gate on.

---

## If it cannot reach LM Studio

Symptom: Hermes starts but every message fails with a connection error.

Check, in order:

1. Is LM Studio's server actually on, with a model loaded? The Developer tab
   shows a running indicator.
2. Is **"Serve on Local Network"** enabled? This is the usual culprit.
3. Test from inside the container itself:

       docker exec -it hermes curl http://host.docker.internal:1234/v1/models

   If that returns JSON, the network path is fine and the problem is the model
   name in `config.yaml`. If it hangs or refuses, it is step 2.
4. Windows Firewall may prompt the first time LM Studio binds to the network.
   If you dismissed that prompt, allow `LM Studio` for private networks in
   Windows Defender Firewall settings.

---

## Useful commands

    docker compose logs -f hermes     # follow logs
    docker compose down               # stop (state is preserved)
    docker compose down -v            # stop AND delete all state — irreversible
    docker compose pull && docker compose up -d    # upgrade

The container is stateless; everything lives in the `hermes-data` volume.
Upgrades are safe.

---

## What this setup does NOT protect against

Worth being honest about the limits:

- **Network egress.** The container can reach the internet freely. Nothing is
  going to an LLM provider since your model is local, but if the web toolset is
  enabled the agent can still fetch pages and, in principle, send data outward.
  Cut it with `disabled_toolsets: [web]` if that matters to you.
- **Container escape.** Rare, but Docker is not a hypervisor. If you ever want
  to let this run unattended with broad permissions, move it to a VirtualBox
  guest instead.
- **Your own mounts.** Every folder you add later is a hole you made on
  purpose. Keep them dedicated and narrow.
