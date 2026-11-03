# NitroMTA 0.1.6 on macOS x86_64 — one real delivery, run notes

Run on 3 Nov 2026 (UTC). Everything below happened in this run; nothing is copied from the docs without being done.

- **Install and version.** No Homebrew: the release archive, checked before use.
  `gh release download v0.1.6 -R nitrosend/nitromta -p 'nitromta-0.1.6-x86_64-macos.tar.gz'`, then
  `gh attestation verify nitromta-0.1.6-x86_64-macos.tar.gz --repo nitrosend/nitromta --format json` →
  one SLSA v1 provenance attestation, signer `nitrosend/nitromta/.github/workflows/release.yml@refs/tags/v0.1.6`,
  subject digest matching the archive's sha256 `41fc072f70660539fd143f5c2b97d07d36f5f66b12edd2dfd7eaf444e6bac1d6`.
  `tar -xzf` → `nitromta --version` printed **`nitromta 0.1.6`**. The extracted binary carried no quarantine attribute, so it launched without a Gatekeeper prompt. Below, `nitromta` means that extracted binary, run by path.

- **OS and architecture.** macOS 26.7.1, Intel `x86_64`, as my own unprivileged user. SMTP client: Python 3.14 `smtplib`.

- **Exact commands and every change to `config.toml`.** In an empty directory:
  ```sh
  nitromta setup --config config.toml --capture
  # config.toml edits (the only three):
  #   [delivery]  "# port = 25"   ->  "port = 2525"
  #   [delivery]  "# dane = true" ->  "dane = false"
  #   removed the [capture] table (directory = "./capture") and its two comment lines
  #   require_tls left at its default (true)
  nohup nitromta run --config config.toml > engine.log 2>&1 &   # my own user, not root
  openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out dkim.pem
  nitromta admin --config config.toml <<END
  {"op":"put_account","account":1}
  {"op":"put_identity","account":1,"name":"example.test","selector":"s1","private_key":<dkim.pem as one JSON string>}
  {"op":"create_login","account":1,"user":"app"}
  END
  ```
  The PEM was JSON-encoded with `python3 -c 'import json;print(json.dumps(open("dkim.pem").read()))'`. No `put_egress`:
  the engine delivered from the host's default route without one. One message, submitted over SMTP to `127.0.0.1:2587`
  with STARTTLS trusting `./tls.crt` and AUTH as `app`: From `app@example.test`, To `proof@nitromta.com`,
  Reply-To my own mailbox. Reply: `250 2.0.0 Ok: queued as 00c54b084c800000`. Then
  `nitromta explain --config config.toml --message 00c54b084c800000` → delivery `00c54b084d600001`,
  `finished, delivered, at 2026-11-03T11:37:14.071Z`, 1 attempt. `/metrics` agreed:
  `nitromta_messages_accepted_total 1`, `nitromta_attempts_total{result="delivered"} 1`.
  Before relying on port 2525 I checked it was reachable: the MX for nitromta.com is `mta-proof.nitrosend.com`, which answered on 2525.

- **Minutes from install to the receipt: 3.9.** Archive download started 11:33:22 UTC; the receipt email is dated 11:37:14 UTC.
  Submission to delivery took 4 seconds (11:37:10 → 11:37:14).

- **The step that slowed me most: my first SMTP submission.** I passed `EmailMessage.as_bytes()` to smtplib's low-level
  `data()`. That serialises with bare LF, and `data()` only rewrites line endings for `str` input, so the engine answered
  `554 5.6.0 Every line must end in CRLF and fit in 998 characters` and accepted nothing (the accepted-messages counter stayed at 0).
  `msg.as_bytes(policy=email.policy.SMTP)` fixed it. The refusal was correct and the reply said exactly what was wrong.
  Worth knowing: the receipt was delivered to Gmail's **Spam** folder, not the inbox (I searched both); anyone watching only the inbox would think it never came.

- **One doc gap, with an exact repro.** `setup --capture` prints, and `docs/capture.md` says, that `127.0.0.1:9090` serves
  "health, metrics and the capture inspector", but names no health path. The obvious guess fails:
  `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:9090/health` → `404` (also `/healthz`, `/livez`, `/ready`).
  The paths are `/health/live`, `/health/accept` and `/health/dispatch` (all `200` here), listed only in
  `docs/operations.md` under "Is it healthy?". Also observed, harmless here: this Mac's `/etc/resolv.conf` lists a scoped
  link-local resolver first (`nameserver fe80::…%en0`, then `192.168.0.1`); `nitromta run` logged
  `skipped a nameserver that cannot be used` for it **twice** at startup, then resolved through `192.168.0.1`.

- **One change I would make.** Add a short "from capture to real delivery" section to `docs/capture.md`: the three
  `[delivery]` edits above, `put_identity` with `selector` and `private_key` (the quickstart's identity has no key), and a
  note that a local run needs no `put_egress` — the only DKIM example today is in the install guide, written for a systemd
  host with an egress address. Naming the `/health/*` paths in the same guide would close the gap above.
