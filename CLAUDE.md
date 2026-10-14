# Working conventions

## Google Docs, Sheets and Drive
When asked to push, export or copy anything into Google Docs, Sheets or Drive,
load the `anthropic-skills:google-workspace` skill first and use the HTML upload
route via the Google Drive `create_file` tool (a .docx upload is rejected by the
connector). Link the resulting file in the reply.
