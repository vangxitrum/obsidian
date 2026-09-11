# headless-chrome-no-localhost

Google Chrome launched from an agent Bash session (`google-chrome --headless=new ... --dump-dom`
or over CDP on `--remote-debugging-port`) cannot reach a server on `localhost`/`127.0.0.1` in this
sandbox, even though `curl` from the same shell reaches it fine.

Symptoms, all seen on 2026-09-09 against a Next.js app on :3011:
- `--dump-dom http://127.0.0.1:3011/` exits 124 (timeout) with 0 bytes, while
  `--dump-dom "data:text/html,<b>hi</b>"` works - so Chrome itself is healthy, the network path is not.
- Over CDP the tab reports `location.href === "about:blank"`; `Page.navigate` and
  `Runtime.evaluate` on a tab opened at the URL time out.
- `--no-proxy-server` and `--no-sandbox` do not help; the JSON API endpoint hangs too, so it is
  not a page-level problem.

Consequence: browser end-to-end checks of a locally served app are not available from the agent
shell. Verify instead with `curl` for server-rendered markup, a grep of `.next/static/chunks` for
the client behaviour that shipped, plus unit tests - and ask the human to eyeball the live page.

CDP notes worth keeping if this is retried elsewhere: `/json/new` needs `PUT`, not `GET`; a
background tab is `document.hidden`, so any poll that pauses when hidden will look dead.

Also beware: `pkill -f "remote-debugging-port=9222"` from the agent shell kills the agent's own
Bash process (exit 144). Kill the launcher pid instead.

Related: [[nextjs-e2e-isolated-copy]]
