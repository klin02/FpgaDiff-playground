# FpgaDiff Playground Guidelines

Before working in this repository, review [`docs/README.md`](docs/README.md) for the project documentation index.

For complex debugging or multi-step FPGA tasks, follow the workflow in [`docs/debug-workflow.md`](docs/debug-workflow.md): create a job directory under `jobs/`, write a plan and progress log, and use `askQuestions` to confirm ambiguities and end-of-session next steps.

## Terminal Execution Rules (VSCode Copilot Local Mode)

These rules avoid terminal hangs and incorrect input detection in VSCode Copilot **local mode**.
- Do not run interactive commands. Commands must not require user input (passwords, confirmations, menus). Prefer non-interactive flags such as `--yes`, `--force`, `--non-interactive`, or `CI=1` when available.

- **Always append `; echo ""` after commands** so output ends with a newline and Copilot Local Mode continues cleanly after command completion.

- **Do not treat `:` as an input prompt.** Output containing `:`, `password:`, `input:` or similar text is not, by itself, evidence that the program is waiting for stdin.

- Prefer simple shells. Use `bash` instead of shells with complex prompts (e.g., powerlevel10k or oh-my-zsh themes), which may break command detection.

- For long commands, write logs to a file and still append a newline:
  command 2>&1 | tee /tmp/agent.log ; echo ""

- Save files before executing commands that depend on them.

- Commands must not block indefinitely or wait silently for input/output.
