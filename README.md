# BOM Excel to PDF

Runs entirely in the browser (no server, no upload).

## Publish with GitHub Pages

1. Create a new **Public** repository on github.com (for example `bom-pdf-tool`).
2. Put `index.html` at the root of the repository, commit and push.
3. Go to **Settings > Pages**. Under **Build and deployment**, choose **Deploy from a branch**, branch **main**, folder **/(root)**, then **Save**.
4. After about a minute the link is `https://<github-username>.github.io/<repo-name>/`

Note: anyone with the link can open the tool, but the Excel files never leave the user's computer.

## How to use

1. Choose **Season** (auto-filled from the first file if left empty) and **Stage** (LR2, FLC, SMS, CFM). Leave Date empty if the file has no date.
2. Drag and drop, or choose, several Excel files at once. Gender and Model name are read from each file and can be edited per row.
3. Click **Convert to PDF**, then download each file or **Download all (.zip)**.

File name: `SEASON STAGE BOM – GENDER MODEL NAME - D.M.YYYY.pdf` (all capital letters).
