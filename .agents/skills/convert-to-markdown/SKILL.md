---
name: convert-to-markdown
description: Convert .docx or .pdf files to markdown format. Use this skill whenever the user wants to convert a Word document or PDF to markdown, turn a PDF or docx into a markdown file, or needs to convert a document for use in a markdown-based system. This skill handles image extraction and produces clean markdown output with proper image references. Trigger when user mentions converting documents,  or requests file format conversion to markdown.
---

# Convert to Markdown

## Overview

This skill converts `.docx` and `.pdf` files to markdown format. Images are extracted to an `assets` subdirectory and the markdown is saved to a `converts` subdirectory.

## Input Parameters

- **source_file**: Path to the `.docx` or `.pdf` file to convert
- **output_dir**: Output directory (default: `raw` in current directory)

## Output Structure

```
{output_dir}/
├── assets/
│   └── {原文件名}/
│       ├── image-001.png
│       └── image-002.png
└── converts/
    └── {原文件名}.md
```

> Note: Pandoc extracts images to an internal `media/` subdirectory; the post-conversion script moves them up to the `{SOURCE_NAME}/` root for cleaner paths.

## Image Reference Format

All images in the markdown file are referenced using the format:
```
![](/assets/{原文件名}/{图片名})
```

## Conversion Process

### Step 1: Validate Input

Verify the source file exists and has a valid extension (`.docx` or `.pdf`). Get the base name (without extension) for use in directory naming.

### Step 2: Create Output Directories

```bash
SOURCE_NAME=$(basename "source_file" .ext)
mkdir -p "{output_dir}/assets/${SOURCE_NAME}"
mkdir -p "{output_dir}/converts"
```

### Step 3: Convert Based on File Type

**For `.docx` files:** use `pandoc` to convert to markdown with image extraction

```bash
pandoc --track-changes=all --extract-media="{output_dir}/assets/${SOURCE_NAME}" "source_file" -o "{output_dir}/converts/${SOURCE_NAME}.md"
```

> **Note:** `--extract-media` extracts all embedded images to the specified directory. Pandoc may wrap images in paragraphs (`<p>`) or use `![](media/...)` paths — this is normal and will be fixed in the path correction step below.

**For `.pdf` files:** use `pymupdf4llm` to convert to markdown with images

```python
import pymupdf4llm
import os

md_text = pymupdf4llm.to_markdown(
    doc="source_file",
    write_images=True,
    image_path="{output_dir}/assets/${SOURCE_NAME}",
    image_format="png",
    rel_path="/assets/{SOURCE_NAME}/"
)

with open("{output_dir}/converts/{SOURCE_NAME}.md", "w", encoding="utf-8") as f:
    f.write(md_text)
```

**CRITICAL path fix**: After conversion, pandoc stores images in `assets/{SOURCE_NAME}/media/` but the desired markdown reference is `![](/assets/{SOURCE_NAME}/image-001.png)` (no `media/` in path). Move images up one level and fix references:

```python
import re
import shutil
import os

MEDIA_DIR = "{output_dir}/assets/{SOURCE_NAME}/media"
ASSETS_DIR = "{output_dir}/assets/{SOURCE_NAME}"

for fname in os.listdir(MEDIA_DIR):
    shutil.move(os.path.join(MEDIA_DIR, fname), os.path.join(ASSETS_DIR, fname))

with open("{output_dir}/converts/{SOURCE_NAME}.md", "r", encoding="utf-8") as f:
    content = f.read()

content = re.sub(r'!\[\]\(media/', f'![](/assets/{SOURCE_NAME}/', content)
content = re.sub(r'!\[\]\(\.\./media/', f'![](/assets/{SOURCE_NAME}/', content)

with open("{output_dir}/converts/{SOURCE_NAME}.md", "w", encoding="utf-8") as f:
    f.write(content)
```

The final format must be: `![](/assets/{SOURCE_NAME}/image-001.png)` (leading `/` is required)

### Step 4: Verify Output

Ensure:
- All images are in `{output_dir}/assets/{SOURCE_NAME}/` with clean names (`image-001.png`, etc.)
- Markdown file exists at `{output_dir}/converts/{SOURCE_NAME}.md`
- All image references use the correct format: `![](/assets/{SOURCE_NAME}/image-xxx.png)`

## Example

**Input:** `/home/user/docs/report.pdf`
**Command:** Convert with default output directory

**Output:**
```
./raw/
├── assets/
│   └── report/
│       ├── image-001.png
│       └── image-002.png
└── converts/
    └── report.md
```

The markdown file `report.md` would reference images as:
```
![](/assets/report/image-001.png)
```
