# [Self-Review Questionnaire: Security and Privacy](https://w3c.github.io/security-questionnaire/)

> 01. What information does this feature expose, and for what purposes?

WebMCP exposes author-defined tool metadata and tool return values to the built-in AI agent. It does not expose new information about the user or their environment to origins.

Cross-origin iframes may discover these tools only if the tool author explicitly opts in via [`exposedTo`](https://webmachinelearning.github.io/webmcp/#dom-modelcontextregistertooloptions-exposedto).

> 02. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes. The API surface exposes only what is necessary for agents to discover and invoke tools. The information that flows through tool metadata like parameters and annotations, as well as tool return values, is entirely scoped to what the author declares.

> 03. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No, the API itself does not expose PII, but the tools that authors choose to implement _can_, depending on their nature.

We note a novel challenge for agent implementers: malicious tools can request a non-minimal set of personal data via their input parameters, causing privacy leakage. See [Privacy Leakage through Over-Parameterization](https://webmachinelearning.github.io/webmcp/#privacy-leakage-over-parameterization) for details. WebMCP does not increase the attack vector compared to tools in non-WebMCP contexts, but agent implementers should be aware that this risk exists.

> 04. How do the features in your specification deal with sensitive information?

WebMCP is not a source of sensitive information. Tools may wrap sensitive or high-privilege operations (e.g., purchases, account changes), but that risk is not WebMCP-specific. We discuss this risk in [Tool Implementation as Attack Targets](https://webmachinelearning.github.io/webmcp/#tool-implementation-targets).

As of now, the spec does not include normative guidance against the misuse of tools that expose sensitive or high-privilege operations.

We do see room for the API design to help. For example, we intend to add a hint for consequential actions (see [#176](https://github.com/webmachinelearning/webmcp/issues/176)) so authors can flag higher-risk actions for the user agent to safeguard. We will continue to explore further mitigations, which will be documented at [Mitigations](https://webmachinelearning.github.io/webmcp/#mitigations).

> 05. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No, the API surface itself does not carry related but distinct information.

> 06. Do the features in your specification introduce state that persists across browsing sessions?

No. Tool registrations are tied to the document's lifetime. There are discussions about persisting tools across navigation, but that is not currently specified.

> 07. Do the features in your specification expose information about the underlying platform to origins?

No. While the API introduces a communication channel to agents that could have information about the underlying platform, these agents were already able to provide this information to the site through other channels, e.g. local network endpoints or filling HTML forms.

> 08. Does this specification allow an origin to send data to the underlying platform?

Yes, tool inputs and outputs flow between an origin and the platform's built-in agent. The data is structured JSON-serializable values conforming to declared schemas.

> 09. Do features in this specification enable access to device sensors?

No. While the API introduces a communication channel to agents that could have access to device sensors, these agents were already able to communicate with the site through other channels, e.g. local network endpoints or filling HTML forms.

> 10. Do features in this specification enable new script execution/loading mechanisms?

Yes, it introduces a new script invocation mechanism. Cross-origin documents authorized via [`exposedTo`](https://webmachinelearning.github.io/webmcp/#dom-modelcontextregistertooloptions-exposedto), as well as built-in agents, can directly invoke a tool's [`execute`](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttool-execute) callback with structured, schema-conforming arguments. 

These callbacks are ordinary JavaScript running in the registering document's existing realm, no new script content can be loaded.

> 11. Do features in this specification allow an origin to access other devices?

No.

> 12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

Yes, origin-supplied tools can influence the user agent's UI in the following ways:

- A tool's [`title`](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttool-title) is displayed by the user agent when referencing the tool in its UI.
- Tool responses (the return value of [`execute`](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttool-execute)) may be shown in, or influence, the user agent's UI.
- [Tool annotations](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttoolannotations) can indirectly influence how an agent presents a tool invocation (e.g., a `readOnlyHint` may cause the agent to skip a confirmation step).

There is also discussion of `requestUserInput` in [Issue #165](https://github.com/webmachinelearning/webmcp/issues/165).

> 13. What temporary identifiers do the features in this specification create or expose to the web?

None.

> 14. How does this specification distinguish between behavior in first-party and third-party contexts?

The feature is gated by the [`"tools"`](https://webmachinelearning.github.io/webmcp/#permissiondef-tools) permission policy. It is allowed in top-level documents and same-origin descendants by default; The permission policy can be used to allow it in cross-origin iframes and/or to disallow it in same-origin frames.

Additionally, tools can specify [`exposedTo`](https://webmachinelearning.github.io/webmcp/#dom-modelcontextregistertooloptions-exposedto) to control which origins (or `native-agents`, name to be bikeshed per [#179](https://github.com/webmachinelearning/webmcp/pull/179)) can discover them.

> 15. How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?

We do not anticipate any differences, but implementers should be aware of how to safely handle private browsing modes. See [Interaction with Private Browsing Modes](https://webmachinelearning.github.io/webmcp/#interaction-with-private-browsing).

> 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes. See [Security and Privacy Considerations](https://webmachinelearning.github.io/webmcp/#security-privacy).

> 17. Do features in your specification enable origins to downgrade default security protections?

No.

> 18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

A BFCached document's registered tools remain in memory but are unavailable while the document is non-fully-active: tools cannot be invoked, registered, or retrieved. On restoration, registered tools become available again.

> 19. What happens when a document that uses your feature gets disconnected?

A disconnected document's tools are no longer discoverable or invokable by agents. Pending tool invocations associated with the document are abandoned:

- In-page agents: the caller's Promise will be rejected
- Built-in agents: the agent will be notified that the tool call failed

Note: this behavior is not yet spec'd but is the intended direction.

> 20. Does your spec define when and how new kinds of errors should be raised?

Yes. `registerTool()` throws `InvalidStateError` for inactive documents, duplicate names, or invalid name/description; `NotAllowedError` when the `"tools"` Permissions Policy is disallowed; `SecurityError` for non-trustworthy [`exposedTo`](https://webmachinelearning.github.io/webmcp/#dom-modelcontextregistertooloptions-exposedto) origins; and `TypeError` when `inputSchema` or `outputSchema` serialization fails. These errors only reflect the page's own state and inputs, so they do not leak new information.

> 21. Does your feature allow sites to learn about the user's use of assistive technology?

No.

> 22. What should this questionnaire have asked?

We would like to surface a risk not directly surfaced by this questionnaire: agents browsing multiple origins may carry state from one origin to another if not handled securely by the user agent. This risk is inherent to multi-origin agent browsing and exists independently of WebMCP.

This section is currently marked as TODO in the draft spec, and we intend to update it with our analysis soon. See [Violation of Same-Origin Boundaries](https://webmachinelearning.github.io/webmcp/#violation-same-origin-boundaries).