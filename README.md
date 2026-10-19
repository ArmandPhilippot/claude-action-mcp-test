# Claude Code Action + Astro Docs MCP test

Checks that the snippet in [withastro/docs#14640](https://github.com/withastro/docs/pull/14640) works: `claude-code-action@v1` connects to the Astro Docs MCP server through `claude_args` and is allowed to call `search_astro_docs`.

The workflow is [`.github/workflows/claude.yml`](.github/workflows/claude.yml): the official `examples/claude.yml`, trimmed to issue events, with the PR snippet in the `Run Claude Code` step. Two additions are for the test only:

- `display_report: true` shows the tool calls in the run summary.
- The owner check in the job's `if` skips runs started by anyone else. The action already refuses users without write access, but the job would still start and fail.

## 1. Local pre-check (optional, no repository needed)

The action passes `claude_args` to the Claude Code CLI, so the same flags can be checked locally:

```sh
claude -p "Use the astro-docs MCP server to find how to add a sitemap to an Astro site. Cite the docs page." \
  --mcp-config '{"mcpServers": {"astro-docs": {"type": "http", "url": "https://mcp.docs.astro.build/mcp"}}}' \
  --allowedTools mcp__astro-docs__search_astro_docs \
  --output-format stream-json --verbose | grep -o '"name":"mcp__astro-docs__[a-z_]*"' | sort -u
```

Expected output: `"name":"mcp__astro-docs__search_astro_docs"`. Running it without `--allowedTools` should show the tool call denied, which confirms the flag matters.

This does not check how the action parses `claude_args`. The next steps do.

## 2. Create a public repository

The repository is public so the PR can link to the issue and the run as proof.

```sh
cd claude-action-mcp-test
git add . && git commit -m "Test Claude Code Action with the Astro Docs MCP server"
gh repo create claude-action-mcp-test --public --source . --push
```

## 3. Set up Claude

1. Install the Claude GitHub App on that repository only: https://github.com/apps/claude
2. Add the API key as a repository secret:

   ```sh
   gh secret set ANTHROPIC_API_KEY --repo <you>/claude-action-mcp-test
   ```

   With a Pro or Max subscription instead of an API key: run `claude setup-token`, store the result with `gh secret set CLAUDE_CODE_OAUTH_TOKEN`, and replace the `anthropic_api_key` line in the workflow with `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`. The part under test (`claude_args`) stays the same.

## 4. Trigger the workflow

```sh
gh issue create --repo <you>/claude-action-mcp-test \
  --title "MCP test" \
  --body "@claude Using the Astro docs, explain how to add a sitemap to an Astro site and link the docs page you used."
```

## 5. Check the result

- **Workflow run** (`gh run watch --repo <you>/claude-action-mcp-test`): it succeeds.
- **Run summary** (Claude Code Report): it lists a call to `mcp__astro-docs__search_astro_docs` with a result, not a permission denial.
- **Issue**: Claude's comment answers with a link to `docs.astro.build`.

If the report does not list the tool, make the repository private first (`gh repo edit --visibility private --accept-visibility-change-consequences`), then set `show_full_output: true` in the workflow and look for the `astro-docs` server status (`connected` or `failed`) in the `init` message of the logs. The full output can include sensitive data, and run logs of a public repository are visible to anyone. Remove the setting before making the repository public again.

## 6. Link the proof in the PR

Add to the PR description:

- the issue URL, where Claude's reply stays visible,
- the workflow run URL, whose summary lists the `mcp__astro-docs__search_astro_docs` call. GitHub deletes run logs after 90 days at most, so the issue is the lasting proof.

## 7. Clean up

Once the PR is merged, if you no longer need the proof:

```sh
gh repo delete <you>/claude-action-mcp-test
```

Then remove the app from the repository, and revoke the API key if you created one for this test.
