---
name: make-video-pro-cli
description: Use the Make Video Pro CLI to generate or translate subtitles, translate or dub video speech, create clips, render or export videos, and download processed results. Trigger for these tasks even when Make Video Pro is not mentioned. Also use for authentication, languages, templates, status, credits, and troubleshooting.
---

# Make Video Pro CLI

Use the public Make Video Pro CLI as the execution layer. Keep the skill focused on workflow and safety; retrieve the current command contract from the CLI instead of memorizing its flags.

## Discover the Current CLI Contract

Run this before composing commands:

```bash
npx -y make-video-pro@latest help --json
```

Treat the returned invocation, commands, options, workflow, and agent guidance as authoritative. Do not invent commands or flags. Run help again after an unknown-command or unsupported-option result.

Respect a version explicitly pinned by the user. Otherwise use `@latest` so the command reference and implementation stay aligned. Do not install the CLI globally.

## Execute the Workflow

1. Identify the requested result, input media path or video ID, desired languages, optional template, and output path.
2. Retrieve `help --json` and select the smallest workflow that produces the requested result.
3. Use `--json` on every agent-driven command. Parse structured output instead of scraping human-readable text.
4. If authentication is required, run the documented `login --json` flow and let the user complete browser authorization. Never request, print, or manually store the session token.
5. Run `languages --json` before selecting language codes. Use the exact returned codes.
6. Run `templates --json` only when a saved template is requested or could materially change the requested processing.
7. Use `--wait` when the user expects the processing or render to finish in the current task. Otherwise return the video ID and current status.
8. Inspect `status <videoId> --json` before rendering when readiness is uncertain. Follow the structured next action instead of guessing.
9. If upload or restore reaches the active-video limit, run `list --json`, identify an unneeded active video, show its exact ID to the user, and obtain explicit confirmation before running `archive <videoId> --confirm --json`.
10. Restore an archived video with `restore <videoId> --json` when an active-video slot is available.
11. For follow-up changes to a processed video, reuse its existing video ID and render again with updated options. Do not upload the source again unless the source file changed or the requested change requires upload-time reprocessing. If the CLI cannot apply the change, explain the limitation instead of re-uploading.
12. Prefer the current `render-local` invocation returned by help. For long videos, run local rendering in the agent or runtime's durable background mode and monitor it until completion. Do not keep a long local render inside one foreground tool call with a fixed timeout, because the runner may terminate it with `SIGTERM` when that timeout expires. If local rendering is unavailable or returns an error, stop and report it. Never switch to `render-server` as an automatic fallback.
13. Verify the final file exists at the requested path before reporting completion.

Use the production service by default. Pass `--base-url` or environment overrides only when the user explicitly requests a development or staging environment.

## Require Confirmation for Costs and Destructive Actions

Do not silently authorize credit usage.

- If a command returns a credit estimate or asks for `--confirm-credits`, show the operation and estimated cost to the user and wait for explicit approval before retrying with confirmation.
- Before `render-server`, explain that it is paid cloud rendering, check the available credits with the current documented command, show the estimated cost when available, and wait for explicit approval. Never treat local-render unavailability or failure as permission to run `render-server`; approval is required even when the account has enough credits or the CLI accepts `--confirm-credits`.
- Do not use `--force` to start a new server render unless the user explicitly asks to replace or repeat an existing render.
- Do not use `--overwrite` when an output file already exists unless the user explicitly approves replacing it.
- Before archiving, show the exact video ID and wait for explicit user confirmation. Only then run `archive <videoId> --confirm --json`.
- Do not upload a different file, choose a different language, or enable optional paid processing merely to work around an error.

Approval for one paid or destructive action does not authorize later actions with a different target, output, or cost.

## Manage Archive Lifecycle

- Active videos are automatically archived after 10 days without updates.
- Archived videos remain restorable for 30 days from archiving.
- After 30 days in the archive, videos are permanently deleted automatically and can no longer be restored.

## Handle Structured Results

Use the CLI's structured error code and details to choose the next action.

- On an authentication-required or expired-session result, run the login flow and then retry the original command.
- On a processing or render-in-progress result, use the documented status or wait flow.
- On insufficient credits, report the required and available amounts when provided and give the returned recharge URL. Do not retry repeatedly.
- On `ACTIVE_VIDEO_LIMIT_REACHED`, list active videos and ask the user to confirm one unneeded video to archive before retrying upload or restore.
- On a missing local renderer, explain the requirement and use the exact local-render invocation from help. Do not switch to paid server rendering without approval.
- On an expired or unavailable download, follow the documented recovery action and preserve the cloud-render confirmation boundary.
- On a local-render failure, report the CLI-provided message and debug-log path, then stop. Do not run `render-server` unless the user explicitly approves paid cloud rendering after being told about the credit charge. Do not expose signed URLs, authorization headers, or tokens from logs.

Ask the user only for information or approval that cannot be derived safely from the request, local files, or structured CLI output.

## Report the Outcome

Return the most useful concrete result:

- completed output file path;
- video ID and processing or render status;
- selected language and template choices;
- credits charged when reported by the CLI; or
- a concise blocker with the exact next user action.

Do not claim success until the CLI reports completion and, for local downloads or renders, the output file is present.
