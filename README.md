# opencode-anthropic-login-via-cli

An [OpenCode](https://github.com/sst/opencode) plugin that lets you use Anthropic models with your Claude Pro or Max subscription. No API key required.

The plugin handles authentication, token management, and request transformation so OpenCode can talk to Anthropic's API the same way the official Claude CLI does.

## How It Works

When you connect through this plugin, a few things happen behind the scenes.

### Binary Introspection

Anthropic's API expects specific headers, scopes, and version strings that change with each Claude CLI release. Instead of hardcoding these values and falling behind, the plugin reads them directly from your installed Claude CLI binary.

It runs `strings` against the binary to extract beta header identifiers (like `claude-code-20250219` or `interleaved-thinking-2025-05-14`), OAuth scope definitions, and the CLI version number. These get used in every API request to make sure the plugin stays compatible with whatever version of the CLI you have installed.

If the CLI isn't available or the extraction fails, the plugin falls back to a set of known defaults that ship with the package.

### Request Transformation

Every API request goes through a custom fetch layer that does several things:

- Swaps the API key header for a Bearer token from your OAuth session
- Merges the extracted beta headers into each request
- Adds `?beta=true` to the messages endpoint
- Prefixes all tool names with `mcp_` on the way out and strips the prefix from streaming responses on the way back
- Replaces references to "OpenCode" in system prompts with "Claude Code" so the model behaves as expected

Since you're on a Pro/Max subscription, the plugin also zeros out cost tracking for all models. You're not paying per token, so the usage display won't show misleading numbers.

### Token Lifecycle

OAuth tokens expire. The plugin handles this transparently:

1. Before each request, it checks if the current token is within five minutes of expiring.
2. If it is, the plugin tries to refresh it using the standard OAuth refresh flow.
3. If that fails (which can happen if the refresh token itself has expired), it falls back to reading fresh credentials from the Claude CLI's credential store.
4. As a last resort, it triggers the CLI to refresh by running a lightweight command (`claude -p . --model haiku hi`), then reads the updated credentials.

Concurrent requests share a single in-flight refresh promise, so you won't hit race conditions or duplicate refresh calls.

## Authentication Methods

The plugin gives you three ways to connect, listed in order of convenience.

### Claude Code (auto)

If you have the [Claude CLI](https://github.com/anthropics/claude-code) installed and logged in, this is the easiest path. The plugin reads your existing OAuth credentials directly:

- On macOS, it pulls them from the system Keychain (under the "Claude Code-credentials" service entry).
- On Linux, it reads `~/.claude/.credentials.json`.

You don't need to do anything. Just select the "Claude Code (auto)" option when connecting in OpenCode and the plugin takes care of the rest.

### Claude Pro/Max (browser)

If you don't have the CLI installed or prefer not to use it, this method opens a standard OAuth flow through `claude.ai`. The plugin generates a PKCE challenge, gives you a URL to open in your browser, and waits for you to paste back the authorization code. It then exchanges that code for tokens and stores them in OpenCode.

### API Key (manual)

You can also enter a regular Anthropic API key if you have one. This bypasses all the OAuth machinery and uses standard API key authentication. Useful if you have API credits or prefer the traditional approach.

## Installation

Add the plugin to your `opencode.json`:

```json
{
  "plugin": ["opencode-anthropic-login-via-cli"]
}
```

Open OpenCode and go to **Connect Provider > Anthropic**. Pick whichever auth method works for you.

## Requirements

- [OpenCode](https://github.com/sst/opencode)
- A Claude Pro or Max subscription (for the OAuth methods)
- Claude CLI installed (only needed for the auto method)

## License

MIT
