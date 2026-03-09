# Import Icons from Figma

Automates downloading SVG icons from Figma nodes and organizing them in the project assets folder. Uses the Figma REST API to export SVG files from specific node IDs and saves them with proper naming conventions.

---

## When to Use This Skill

- Importing icon assets from a Figma file into a React Native project
- Batch-downloading SVGs from Figma for use with `react-native-svg`
- Setting up icon variant components backed by Figma-exported assets
- Exporting icons in bulk with a consistent naming convention

---

## Usage

```
@import-icons --file <figma-file-id> --nodes <node-ids> [options]
```

## Options

### Required
- `--file <figma-file-id>`: Figma file ID (found in the Figma file URL)
- `--nodes <node-id-list>`: Comma-separated list of node IDs (hyphen format, e.g., `1-10369,1-10395`)

### Optional
- `--output <path>`: Output directory relative to `assets/` (default: `images/icons`)
- `--type <type>`: Icon type label for subdirectory and naming (e.g., `diamond`, `coin`)
- `--naming <pattern>`: Naming pattern using `{type}`, `{amount}`, `{index}` placeholders (default: `{type}-{index}.svg`)
- `--format <svg|png>`: Export format (default: `svg`)
- `--scale <number>`: Export scale for raster formats (default: `1`)

## Examples

### Export a set of icons by type
```bash
@import-icons --file <your-figma-file-id> \
  --nodes <node-id-1>,<node-id-2>,<node-id-3> \
  --output images/shop/diamonds \
  --type diamond \
  --naming "{type}-{amount}.svg"
```

### Export a single icon with a fixed name
```bash
@import-icons --file <your-figma-file-id> \
  --nodes <node-id-1> \
  --output images/shop/video \
  --naming "play-video.svg"
```

### Export multiple icon groups at once
```bash
@import-icons --file <your-figma-file-id> \
  --nodes <node-id-1>,<node-id-2>,<node-id-3>,<node-id-4> \
  --output images/shop
```

---

## Implementation Details

### Step 1: Parse Arguments
Extract and validate required and optional parameters from the command arguments.

### Step 2: Convert Node IDs
Figma API uses colon format (`1:10369`) instead of the hyphen format (`1-10369`) used in URLs. Convert all node IDs:
```bash
# Convert 1-10369 to 1:10369
echo "1-10369" | sed 's/-/:/'
```

### Step 3: Call Figma API
Request SVG export URLs from Figma:
```bash
curl -H "X-Figma-Token: $FIGMA_PERSONAL_ACCESS_TOKEN" \
  "https://api.figma.com/v1/images/{fileId}?ids={nodeIds}&format=svg&scale=1"
```

Response format:
```json
{
  "err": null,
  "images": {
    "1:10369": "https://figma-alpha-api.s3.us-west-2.amazonaws.com/images/...",
    "1:10395": "https://figma-alpha-api.s3.us-west-2.amazonaws.com/images/..."
  }
}
```

### Step 4: Download SVG Files
For each image URL in the response:
1. Fetch the SVG content via HTTP GET
2. Generate filename based on naming pattern
3. Save to output directory
4. Verify file was created successfully

### Step 5: Generate Report
Output a summary including:
- Number of files downloaded
- File paths and sizes
- Suggested import statements for React components
- Next steps for implementation

---

## Environment Variables

### Required
- `FIGMA_PERSONAL_ACCESS_TOKEN`: Figma API personal access token
  - Set in `~/.zshrc`: `export FIGMA_PERSONAL_ACCESS_TOKEN="figd_..."`
  - Or in `.claude-project/secrets/figma-token.env`

---

## Integration with Components

After downloading SVGs, update React Native components to use the new assets:

### 1. Import SVG files
```typescript
import Icon1 from '@/assets/images/icons/icon-1.svg';
import Icon2 from '@/assets/images/icons/icon-2.svg';
// ... etc
```

### 2. Create variant map
```typescript
const iconMap: Record<string, React.FC<SvgProps>> = {
  variant1: Icon1,
  variant2: Icon2,
};
```

### 3. Use variant-based rendering
```typescript
interface IconProps {
  size?: number;
  variant: keyof typeof iconMap;
}

export function Icon({ size = 35, variant }: IconProps) {
  const IconComponent = iconMap[variant];
  if (!IconComponent) return null;
  return <IconComponent width={size} height={size} />;
}
```

---

## Prerequisites

- `react-native-svg`: ^15.12.1
- `react-native-svg-transformer`: ^1.5.2
- `metro.config.js` configured with SVG transformer support

---

## Notes

- Figma API export URLs are temporary (expire after ~30 days) — download and commit SVGs to version control immediately
- Test on both iOS and Android as SVG rendering may differ
- Consider optimizing SVG files with SVGO if file sizes are large
- Node IDs are project-specific — find them in the Figma URL when a node is selected (`?node-id=1-10369`)

---

## Troubleshooting

### Error: "Invalid token"
- Verify `FIGMA_PERSONAL_ACCESS_TOKEN` is set correctly
- Check token has not expired
- Ensure token has read access to the file

### Error: "Node not found"
- Verify node IDs are correct in Figma
- Check node ID format (use hyphen format in the CLI: `1-10369`)
- Ensure you have access to the Figma file

### Error: "Failed to download SVG"
- Check network connection
- Verify export URL is accessible
- Ensure output directory exists and is writable

---

## Related Skills

- `converters/figma-to-react-native.md` — Convert full Figma screens to React Native components
- `@figma` MCP server — Direct Figma integration for node inspection
