<skill name="dotnet-sdk-resolver">

## PURPOSE
Independently locate, verify, and export the .NET 10 SDK before
executing any dotnet command. This procedure is mandatory for
every agent that runs dotnet commands. Shell state, PATH, and
DOTNET_ROOT from other workflow nodes are never available.

## SDK RESOLUTION PROCEDURE
Execute these steps in order before any dotnet command:

STEP 1 — Locate the SDK binary:
  DOTNET_BIN="$HOME/.dotnet/dotnet"

STEP 2 — Verify the binary exists and is executable:
  if [ ! -x "$DOTNET_BIN" ]; then
    echo "SDK_STATUS=NOT_FOUND"
    exit 1
  fi

STEP 3 — Verify the SDK family is 10.*:
  SDK_VERSION=$("$DOTNET_BIN" --version)
  if [[ "$SDK_VERSION" != 10.* ]]; then
    echo "SDK_STATUS=WRONG_VERSION: $SDK_VERSION"
    exit 1
  fi

STEP 4 — Export for all subsequent commands in this agent:
  export DOTNET_ROOT="$HOME/.dotnet"
  export PATH="$DOTNET_ROOT:$PATH"
  export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
  export DOTNET_BIN="$DOTNET_ROOT/dotnet"

STEP 5 — Record the verified version:
  echo "SDK_VERSION_USED=$SDK_VERSION"

## MANDATORY RULES
- Never hardcode a patch version (e.g. 10.0.100) unless the
  repository's global.json explicitly requires it.
- A missing PATH entry alone is NOT an SDK failure if the
  binary can be invoked directly via DOTNET_BIN.
- Never claim build or test success if STEP 2 or STEP 3 fails.
- Never substitute source-code analysis for actual SDK execution.
- Record SDK_VERSION_USED in every artifact this agent writes.

## FAILURE BEHAVIOUR
If the SDK cannot be located or verified:
- Do not run any dotnet command.
- Set the stage status to FAILED.
- Record SDK_STATUS=NOT_FOUND or SDK_STATUS=WRONG_VERSION.
- Stop and return control to the orchestrator.

</skill>
