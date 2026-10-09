# bash-trap-handler-sigpipe

Minimal reproduction: a bash error/exit handler that writes to stdout is killed
by SIGPIPE when the stdout reader is already gone, so it loses the real exit
code and skips its cleanup. `trap '' PIPE` at the top of the handler fixes it.

## Run

Requires bash and GNU coreutils >= 8.31 (`env --default-signal`). An older,
busybox or BSD `env` rejects that option: `repro` then prints
`env: unrecognized option` and fails both cases without running the subject.

    ./repro

Expected output:

    --without-fix  exit=141 work-dir-left=yes  as expected
    --with-fix     exit=3   work-dir-left=no   as expected
    OK: bug reproduced without the fix; fix verified.

The `--with-fix` run also prints `printf: write error: Broken pipe` to stderr:
that is the EPIPE the fixed handler now survives.

## When it happens

The script's stdout reader closes early, and later the script fails:

    some-installer | head --lines=5    # head exits after 5 lines
    some-installer | less              # user quits the pager
    ssh host some-installer            # ssh session drops

The script keeps working silently, then fails. Its ERR/EXIT handler prints a
diagnostic to stdout. That write raises SIGPIPE, and the default action
terminates bash immediately:

- the exit status becomes 141 (128 + SIGPIPE) instead of the real error code;
- the rest of the handler never runs -- temp dirs, FIFOs and background
  helpers are left behind.

`set +o errexit` / `set +o pipefail` in the handler do not help: SIGPIPE is a
signal, not a failed command.

## The fix

    handle_exit_trap() {
      local exit_code="$?"
      trap - ERR EXIT
      trap '' PIPE    # writes to a gone reader now fail with EPIPE, not a kill
      ...
    }

With SIGPIPE ignored, the write fails with EPIPE (a normal error return, which
`set +o errexit` already tolerates), so the handler completes its cleanup and
exits with the real code.

## Scope

- Bash only cares about SIGPIPE if it is not already ignored at startup; a
  shell cannot un-ignore an inherited ignored signal. `repro` runs the subject
  under `env --default-signal=PIPE` so the result does not depend on the
  caller.
- Only writes made AFTER the reader is gone matter. If the main body writes to
  stdout after that point, it dies the same way before the handler is reached;
  this fix covers the handler, which is where the cleanup lives.

## Files

- `demo-trap-handler` -- the minimal handler, `--without-fix` / `--with-fix`.
- `repro` -- runs both modes against a reader-less pipe and checks the result.

AI-Assisted.
