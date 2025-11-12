# SWO OneClick Skills - GitHub Content Repository

This folder contains example markdown files and the manifest for the **SWO OneClick Skills** VS Code extension.

## Purpose

This content is designed to be uploaded to a GitHub private repository and consumed by the extension to provide:
- Technology-specific Copilot instructions
- Development guides and best practices
- Chat mode configurations
- Code templates and examples

## Folder Structure

```
github-content/
├── manifest.json                    # Main manifest file (REQUIRED)
├── README.md                        # This file
├── javascript/
│   ├── copilot-instructions.md     # JavaScript best practices
│   └── react-chatmode.md           # React component development guide
├── typescript/
│   └── copilot-instructions.md     # TypeScript strict typing guide
└── python/
    ├── copilot-instructions.md     # Python PEP 8 best practices
    └── django-guide.md             # Django web development guide
```

## Manifest Structure

The `manifest.json` file is the **single source of truth** for all downloadable content. It contains:

### Top-Level Properties
- `version`: Manifest format version (e.g., "1.0.0")
- `lastUpdated`: ISO 8601 timestamp of last update
- `repository`: GitHub repository information
  - `owner`: GitHub organization/user (e.g., "jeitsonbarajas")
  - `name`: Repository name (e.g., "swo-oneclick-skills-content")
  - `branch`: Branch to fetch from (e.g., "main")
  - `url`: Full repository URL

### Technologies Array
Each technology has:
- `id`: Unique identifier (e.g., "javascript")
- `name`: Display name (e.g., "JavaScript")
- `icon`: Emoji or icon character (e.g., "🟨")
- `description`: Brief description
- `color`: Hex color code for UI (e.g., "#f7df1e")
- `totalFiles`: Number of files for this technology

### Files Array
Each file has:
- `id`: Unique identifier (e.g., "js-copilot-1")
- `name`: Display name (e.g., "JavaScript Copilot Instructions")
- `description`: Brief description
- `filename`: Original filename (e.g., "copilot-instructions.md")
- `type`: File category ("copilot-instructions", "chat-mode", "guide", "template", "other")
- `technology`: Technology ID this file belongs to
- `downloadPath`: **Relative path** from repository root (e.g., "javascript/copilot-instructions.md")
- `tags`: Array of searchable tags
- `lastModified`: ISO 8601 timestamp
- `author`: Content author
- `version`: Content version

## How to Add New Content

### 1. Create the Markdown File
Create your markdown file in the appropriate technology folder:
```bash
github-content/
└── <technology>/
    └── <your-file>.md
```

### 2. Update manifest.json

#### Add Technology (if new):
```json
{
  "id": "java",
  "name": "Java",
  "icon": "☕",
  "description": "Enterprise Java development",
  "color": "#007396",
  "totalFiles": 1
}
```

#### Add File Entry:
```json
{
  "id": "java-spring-1",
  "name": "Spring Boot Guide",
  "description": "Spring Boot REST API development",
  "filename": "spring-boot-guide.md",
  "type": "guide",
  "technology": "java",
  "downloadPath": "java/spring-boot-guide.md",
  "tags": ["spring", "boot", "rest", "api", "java"],
  "lastModified": "2025-11-09T10:00:00Z",
  "author": "SWO Team",
  "version": "1.0"
}
```

### 3. Update Metadata
- Increment `totalFiles` count for the technology
- Update `lastUpdated` timestamp in manifest root
- Update `metadata.totalFiles` count

### 4. Commit and Push
```bash
git add .
git commit -m "Add Spring Boot guide"
git push origin main
```

## File Types

The extension recognizes these file types:

| Type | Description | Example |
|------|-------------|---------|
| `copilot-instructions` | GitHub Copilot configuration | JavaScript best practices |
| `chat-mode` | Copilot Chat mode settings | React development mode |
| `guide` | Development guides | Django project guide |
| `template` | Code templates | Express.js API template |
| `other` | Miscellaneous content | General tips |

## Content Guidelines

### Markdown Best Practices
1. **Use clear headings** - H1 for title, H2 for sections
2. **Include metadata** - Add header with generation date, version, author
3. **Provide examples** - Include code snippets with syntax highlighting
4. **Add context** - Explain why, not just how
5. **Keep updated** - Update `lastModified` when changing content

### Code Examples
- Use proper syntax highlighting (```python, ```typescript, etc.)
- Include comments explaining key concepts
- Show realistic, working examples
- Demonstrate best practices

### Length
- **Copilot Instructions**: 100-200 lines (comprehensive but focused)
- **Chat Modes**: 100-150 lines (specific patterns)
- **Guides**: 200-400 lines (detailed walkthroughs)
- **Templates**: 50-100 lines (ready-to-use code)

## GitHub Repository Setup

### Repository Settings
1. **Visibility**: Private (recommended for proprietary content)
2. **Branch**: `main` (or specify in manifest)
3. **Access**: Read access via Personal Access Token (PAT)

### Authentication
The extension uses a GitHub Personal Access Token with **repo** scope:
- Token is embedded in extension configuration
- Token should have **read-only** access for security
- Token is used for private repository access

### CDN Acceleration
GitHub serves raw files via:
```
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>
```

This URL is automatically CDN-accelerated by GitHub for fast global access.

## Testing Your Content

### 1. Validate JSON
```bash
# Check manifest.json syntax
cat manifest.json | python -m json.tool
```

### 2. Check File Paths
Ensure `downloadPath` values match actual file locations:
```bash
# Example: verify javascript/copilot-instructions.md exists
ls javascript/copilot-instructions.md
```

### 3. Verify Totals
- Count files in each technology folder
- Ensure `totalFiles` matches in both technology entry and metadata
- Verify `files` array length matches `metadata.totalFiles`

### 4. Test in Extension
1. Upload content to GitHub
2. Open VS Code extension
3. Click "Refresh" to reload from GitHub
4. Verify technologies display correctly
5. Test downloading each file

## Troubleshooting

### Files Not Appearing
- Check `downloadPath` matches actual file location
- Verify `technology` ID matches an entry in `technologies` array
- Ensure manifest.json is valid JSON

### Download Errors
- Confirm files exist at specified paths
- Check file permissions (should be readable)
- Verify GitHub token has repo access

### Caching Issues
- Extension caches content for 1 hour by default
- Use "Refresh" button to force reload
- Check browser DevTools console for errors

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-09 | Initial content with JS, TS, Python examples |

## Support

For issues or questions:
1. Check extension logs in VS Code Output panel
2. Verify manifest.json structure
3. Test GitHub repository access
4. Contact SWO Team for support

---

**Author:** SoftwareOne Team  
**Last Updated:** 2025-11-09  
**Extension:** SWO OneClick Skills v0.2.7+
