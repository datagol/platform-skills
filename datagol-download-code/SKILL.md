---
name: datagol-download-code
description: How to help users download the generated code as a ZIP file.
---

# Downloading Generated Code

When a user asks how to get or download the generated code, explain the following:

## Download Button

There is a **Download ZIP** button in the Code tab of the workbench panel (top right, next to the file count badge). Clicking it downloads all generated files as a `.zip` archive.

The ZIP contains:
- All files written during this session (e.g. `src/App.tsx`, `src/lib/api.ts`, etc.)
- A `README.md` with setup instructions

## After Downloading

Tell the user:
1. Extract the ZIP
2. Run `npm install` in the extracted folder
3. Edit `src/lib/api.ts` and replace `DATAGOL_TOKEN_PLACEHOLDER` with their actual DataGOL token
4. Run `npm run dev` and open http://localhost:5173

## Starting Fresh

If the user wants to start a new project or clear the current session, they can click **New Chat** in the header. This clears all messages, files, and the sandbox.
