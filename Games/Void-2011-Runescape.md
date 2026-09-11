# Void 2011 RuneScape


Server: https://github.com/GregHib/void

Client: https://github.com/GregHib/void-client

<br>

Huge thanks to Greg!

<br>

```bash
#!/usr/bin/env bash
#
# run.sh — start the void server, wait for it to load, then launch the client.
# Place in ~/Games/2011scape/ alongside void-2.8.2/ and void-client-1.2.0.jar
#
# Usage:
#   ./run.sh            start server, wait, launch client
#   ./run.sh server     start only the server (foreground)
#   ./run.sh client     launch only the client (assumes a server is already up)

set -u

# ---- config ---------------------------------------------------------------
BASE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

SERVER_DIR="$BASE/void-2.8.2"
SERVER_JAR="$SERVER_DIR/void-server-2.8.2.jar"
CLIENT_JAR="$BASE/void-client-1.2.0.jar"

# JDKs. Server ran fine on your default; client may need 11 if it uses applets.
# If the client launches fine with plain `java`, set CLIENT_JAVA="java".
SERVER_JAVA="java"
CLIENT_JAVA="/usr/lib/jvm/java-11-openjdk/bin/java"

SERVER_OPTS="-Xms1g -Xmx4g"
CLIENT_OPTS="-Xmx1g"

HOST="127.0.0.1"
PORT="43594"
WAIT_SECS="120"
# ---------------------------------------------------------------------------

start_server() {
    # Run the server jar directly from its own directory so its relative
    # ./data/cache/ path resolves. This avoids run-server.sh's interactive
    # "Press enter to continue" pause, which doesn't suit a scripted launch.
    cd "$SERVER_DIR" || exit 1
    exec "$SERVER_JAVA" $SERVER_OPTS -jar "$SERVER_JAR"
}

start_client() {
    exec "$CLIENT_JAVA" $CLIENT_OPTS -jar "$CLIENT_JAR"
}

# ---- subcommands ----------------------------------------------------------
case "${1:-all}" in
    server)
        echo "Starting server (foreground)..."
        start_server
        ;;
    client)
        echo "Launching client..."
        start_client
        ;;
    all)
        [ -f "$SERVER_JAR" ] || { echo "Server jar not found: $SERVER_JAR" >&2; exit 1; }
        [ -f "$CLIENT_JAR" ] || { echo "Client jar not found: $CLIENT_JAR" >&2; exit 1; }

        SERVER_PID=""
        cleanup() {
            if [ -n "$SERVER_PID" ] && kill -0 "$SERVER_PID" 2>/dev/null; then
                echo "Shutting down server (pid $SERVER_PID)..."
                kill "$SERVER_PID" 2>/dev/null
                wait "$SERVER_PID" 2>/dev/null
            fi
        }
        trap cleanup EXIT INT TERM

        echo "Starting server..."
        ( start_server ) &
        SERVER_PID=$!

        echo "Waiting for $HOST:$PORT (up to ${WAIT_SECS}s)..."
        waited=0
        until (exec 3<>"/dev/tcp/$HOST/$PORT") 2>/dev/null; do
            if ! kill -0 "$SERVER_PID" 2>/dev/null; then
                echo "Server exited before binding $PORT -- see its output above." >&2
                exit 1
            fi
            sleep 1
            waited=$((waited + 1))
            if [ "$waited" -ge "$WAIT_SECS" ]; then
                echo "Timed out waiting for $HOST:$PORT." >&2
                exit 1
            fi
        done

        echo "Server is up. Launching client..."
        "$CLIENT_JAVA" $CLIENT_OPTS -jar "$CLIENT_JAR"

        echo "Client closed."
        # server torn down by EXIT trap
        ;;
    *)
        echo "Usage: $0 [all|server|client]" >&2
        exit 1
        ;;
esac

