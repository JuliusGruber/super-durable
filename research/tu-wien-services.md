# TU Wien services the office agent depends on

Checked 14 November 2026. Answers the three open questions in
[goal.md](goal.md): how the mailbox can be reached, what Aqueduct offers, and
whether pi-durable's model layer can use Aqueduct.

Method: TU Wien and dataLAB web pages, the pi source at commit `c1d4b0a`, and
unauthenticated requests to the mail server's public addresses. No login was
used anywhere, the mailbox was not opened, and no model was called.

## Summary

| Question | Answer | Confidence |
|---|---|---|
| 1. How can the mailbox be reached? | It is an on-premises Exchange server that answers on EWS, ActiveSync, IMAP and SMTP from the internet, with password logins. | Endpoints observed; permission to automate not found |
| 2. What does Aqueduct offer? | Three chat models on TU Wien hardware behind an OpenAI-compatible API. All three pass the nightly tool-calling test. | Documented; not tried |
| 3. Can pi-durable use Aqueduct? | Yes. pi-ai accepts custom OpenAI-compatible endpoints, and dataLAB publishes a Pi configuration for Aqueduct. | Documented on both sides; not tried |

**Nothing found blocks the plan.** The open risk is in question 1: the agent
would hold the owner's main TU Wien password, and no page says whether
automated access to a staff mailbox is permitted.

## 1. Mailbox access

**Observed on `mail.intern.tuwien.ac.at`** (requests without credentials):

- **On-premises Exchange.** The server reports `X-OWA-Version: 15.2.2562.56`
  on Microsoft IIS 10. Version 15.2 is Exchange Server 2019 or its successor,
  Subscription Edition. It is not Microsoft 365, so Microsoft's cloud API
  (Graph) does not apply.
- **Exchange Web Services answers** at `/EWS/Exchange.asmx`, offering
  Negotiate, NTLM and Basic authentication. EWS covers mail, calendar and
  tasks in one interface.
- **ActiveSync answers** at `/Microsoft-Server-ActiveSync` with Basic
  authentication.
- **Mail ports are open:** 993 and 143 (IMAP), 587 and 25 (SMTP). Port 465 is
  not.
- **Reachable without VPN** from the machine the check ran on, which was not
  confirmed to be outside the TU Wien network.
- **No modern authentication was offered.** Every endpoint asked for a
  password login; none offered OAuth.

An endpoint that asks for a login shows the service runs. It does not show
that a given account is allowed to use it.

**Documented by TU.it:**

- **IMAP and EWS are in use by staff clients.** An outage notice of
  16 February 2026 names "Outlook, Thunderbird, em Client, etc., using IMAP
  and EWS protocols".
- **Login name:** `shortname@intern.tuwien.ac.at` or `intern\shortname`, with
  the TU-AD password.
- **No second factor is listed** for the Exchange mailbox in the staff
  services table; the column is blank.
- **Outgoing mail:** the table lists mail submission through
  `mr.tuwien.ac.at` with the network password. Whether that or the Exchange
  server is the right route for the mailbox is not stated.

**Not found:**

- Any rule on automated access, service mailboxes, or letting another account
  act on a mailbox (delegation).
- App passwords or tokens that could replace the main password.
- Official server settings for staff. The upTUdate service page returned 404.
- Whether Exchange push notifications can reach a service in the Datafactory.
- Any announced move of staff mailboxes to Microsoft 365. Student mailboxes
  are already there.

**What this means for the design:**

- **EWS is the natural interface,** since one protocol covers mail, calendar
  and to-dos. IMAP and SMTP cover mail only. A maintained Node library for EWS
  against an on-premises server was not looked for.
- **Polling is the safe default** for new mail and calendar changes.
- **The password is the main exposure.** It would sit in an OpenShift secret
  and unlocks far more than mail. A dedicated mailbox with delegate rights on
  the owner's mailbox would limit that, if TU.it offers one.
- **Ask TU.it before building.** Whether this use is allowed is a policy
  question that no page answers.

## 2. Aqueduct

Aqueduct is the dataLAB's AI gateway: open source (BSD-3-Clause), built on
the LiteLLM Router SDK and Django, with the base URL
`https://aqueduct.ai.datalab.tuwien.ac.at/v1`.

**Chat models** (models page and the compatibility report generated
14 November 2026):

| Model | Context | Status | Tool calling | Structured output | Images |
|---|---|---|---|---|---|
| `deepseek-v4-vision-exp-284b` | 393,216 | Stable, "our main model" | Pass | Pass | Pass |
| `glm-5.3-744b-experimental` | 262,144 | Experimental, may be removed at short notice | Pass | Pass | Not tested |
| `qwen-3.8-27b` | 262,144 | Stable, the fast small model | Pass | Pass | Pass |

- **Not tested by the report:** parallel tool calls, and tool calls while
  streaming. Plain streaming passes for all three.
- **Cached input tokens** pass for the DeepSeek and GLM models and fail for
  `qwen-3.8-27b`.
- **Models rotate.** They usually stay "3–4 months, at most about 6". An agent
  meant to run for months must expect its model to change.
- **Other services:** speech-to-text, text-to-speech, image generation, files,
  batches, vector stores, a Responses API and hosted MCP servers.

**Access:**

- **Staff need no request.** Log in with the TU Wien account and create a
  personal API key. Students need an approved form.
- **Service accounts exist** for large or long-running jobs, with higher
  limits, requested in the Matrix room `#llm-service:tuwien.ac.at`. An
  always-on agent fits that description.
- **Limits are per key,** by minute, hour and day, and return HTTP 429. The
  numbers are shown only inside the Aqueduct interface. `qwen-3.8-27b` counts
  double.

**Data handling:**

- **All models run on TU Wien hardware.** Inputs are not passed to outside
  providers and are not used for training.
- **Chat Completions prompts and answers are not stored.** Responses API
  answers are stored up to 30 days unless `store` is false. Per-request usage
  data is kept 7 days and logs 30 days.
- **The hosted web search is the exception.** It may send query text to an
  external search provider and is meant for non-personal data only, so the
  agent should not use it on mail content.
- **Other people's personal data** is allowed "only when necessary and with a
  legal basis". Mail contains such data, so this deserves a look.
- **The privacy page is preliminary.** Approval by the TU Wien data protection
  officers was still pending when it was last updated on 5 November 2026.

**Not found:** whether the API is reachable from the public internet or from
the Datafactory cluster, and the actual rate-limit numbers.

## 3. pi-durable with Aqueduct

- **pi-ai supports custom OpenAI-compatible endpoints.** `createProvider()` is
  documented "for local inference servers, proxies, or any
  OpenAI/Anthropic-compatible endpoint", with a model entry that carries
  `baseUrl`, `api: 'openai-completions'`, context window and compatibility
  flags (`packages/ai/README.md`, version 1.5.2).
- **pi-durable takes its models from pi-ai.** `HarnessOptions.models` is
  "pi-ai model access used by generation, compaction, hooks and tools".
- **dataLAB documents Pi with Aqueduct.** Its integration page configures the
  Pi coding agent with an `aqueduct` provider, `"api": "openai-completions"`,
  `maxTokens` 65536, reasoning enabled, a thinking-level map and
  `chatTemplateKwargs` for `reasoning_effort`. That page targets the Pi coding
  agent's `models.json`; the same values would go into `createProvider()` for
  pi-durable.
- **Two settings need care,** both named on the models page: a client's
  `maxTokens` can cut answers short, and clients that do not pass
  `reasoning_effort` in the expected form get an `UnsupportedParamsError`.
- **Compaction thresholds can be set per model** with `modelOverrides`
  (see [source-reading.md](source-reading.md#context-compaction)), which
  suits the different context windows above.

This closes the "Models: Not checked" entry in
[platform-assessment.md](platform-assessment.md) and point 7 of
[blind-spots.md](blind-spots.md).

## What is still unverified

- **Whether these models route and delegate well.** A passing tool-calling
  test is not evidence of Router quality. The article's weaker setups failed
  to delegate at all.
- **Whether the Datafactory cluster can reach Aqueduct and the mail server.**
- **Whether TU.it permits automated access** to a staff mailbox.
- **Rate limits** against the token use of an always-on Router and Watchdog.

## Sources

**Aqueduct**

- [Aqueduct at dataLAB](https://datalab.tuwien.ac.at/aiml/aqueduct/)
- [Models](https://datalab.tuwien.ac.at/aiml/aqueduct/models/)
- [Model compatibility report](https://datalab.tuwien.ac.at/aiml/aqueduct/model-compatibility/)
- [Pi integration](https://datalab.tuwien.ac.at/aiml/aqueduct/pi/)
- [Quickstart](https://datalab.tuwien.ac.at/aiml/aqueduct/quickstart/)
- [Access](https://datalab.tuwien.ac.at/aiml/access/)
- [FAQ](https://datalab.tuwien.ac.at/aiml/faq/)
- [Privacy](https://datalab.tuwien.ac.at/aiml/privacy/)
- [Aqueduct documentation](https://tu-wien-datalab.github.io/aqueduct/)
- [API reference](https://tu-wien-datalab.github.io/aqueduct/api)
- [Source repository](https://github.com/TU-Wien-dataLAB/aqueduct)

**Mail**

- [upTUdate outage notice, 16 February 2026](https://www.it.tuwien.ac.at/en/all-service-outages/news/stoerung-uptudate-13)
- [Accounts for employees](https://www.it.tuwien.ac.at/en/support/rights-and-roles/accounts-at-the-tuw/accounts-for-employees)
- [Services and passwords table for employees](https://www.it.tuwien.ac.at/en/support/rights-and-roles/accounts-at-the-tuw/accounts-for-employees/services-and-passwords-table-for-employees)

**pi**

- [pi-ai README](https://github.com/earendil-works/pi/tree/main/packages/ai)
- [pi-durable README](https://github.com/earendil-works/pi/tree/main/packages/durable)
