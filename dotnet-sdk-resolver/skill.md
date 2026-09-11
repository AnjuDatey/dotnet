## PURPOSE

Independently locate, verify, and export the .NET 10 SDK
before executing any dotnet command. This procedure is
mandatory for every agent that runs dotnet commands.

Shell state, PATH, and DOTNET_ROOT from other workflow nodes
are never available. Resolve independently every time.

---

## SDK RESOLUTION PROCEDURE

Execute these exact steps before any dotnet command:

STEP 1 — Locate the SDK binary by checking paths in order:

if [ -x "/tmp/.dotnet/dotnet" ]; then
    DOTNET_PATH="/tmp/.dotnet/dotnet"
elif [ -x "$HOME/.dotnet/dotnet" ]; then
    DOTNET_PATH="$HOME/.dotnet/dotnet"
elif [ -x "/root/.dotnet/dotnet" ]; then
    DOTNET_PATH="/root/.dotnet/dotnet"
elif [ -x "/home/user/.dotnet/dotnet" ]; then
    DOTNET_PATH="/home/user/.dotnet/dotnet"
else
    echo "SDK_RESOLUTION=FAILED"
    echo "SDK_STATUS=NOT_FOUND"
    exit 1
fi

STEP 2 — Verify the SDK family is 10.*:

SDK_VERSION=$("$DOTNET_PATH" --version)
if [[ "$SDK_VERSION" != 10.* ]]; then
    echo "SDK_STATUS=WRONG_VERSION: $SDK_VERSION"
    exit 1
fi

STEP 3 — Export for all subsequent commands:

export DOTNET_ROOT="$(dirname "$DOTNET_PATH")"
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1

STEP 4 — Record the verified version:

echo "DOTNET_PATH=$DOTNET_PATH"
echo "SDK_VERSION=$SDK_VERSION"

---

## MANDATORY RULES

- Always use "$DOTNET_PATH" for all dotnet commands.
- Never use bare: dotnet restore / dotnet build / dotnet test
- A missing PATH entry alone is NOT an SDK failure.
- Never install another SDK.
- Never use an OS package manager to install .NET.
- Record SDK_VERSION in every artifact this agent writes.

---

## FAILURE BEHAVIOUR

If the SDK cannot be located or verified:
- Do not run any dotnet command.
- Set the stage status to FAILED.
- Record SDK_RESOLUTION=FAILED in the artifact.
- Stop and return control to the orchestrator.
