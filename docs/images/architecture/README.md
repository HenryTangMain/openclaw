# Architecture SVG Diagrams

This directory contains auto-generated SVG diagrams from the Mermaid diagrams in the architecture documentation.

## Diagram Sources

Diagrams are defined in Mermaid syntax within the architecture documentation files:
- `docs/concepts/system-architecture.md`
- `docs/concepts/plugin-architecture.md`
- `docs/concepts/channel-provider-architecture.md`
- `docs/concepts/agent-runtime-architecture.md`
- `docs/concepts/data-state-architecture.md`

## Generating SVGs

To generate SVG diagrams from the Mermaid source:

### Prerequisites

1. Install Mermaid CLI:
```bash
npm install -g @mermaid-js/mermaid-cli
```

2. Install required fonts for proper rendering (optional):
```bash
# On macOS
brew install --cask font-fira-code

# On Ubuntu/Debian
sudo apt-get install fonts-firacode

# On Fedora
sudo dnf install fira-code-fonts
```

### Generate All Diagrams

```bash
# From repository root
cd docs/concepts

# Generate system architecture diagrams
mmdc -i system-architecture.md -o ../images/architecture/system-context.svg -b transparent -w 1200 -H 800 --extractSvg
mmdc -i system-architecture.md -o ../images/architecture/system-containers.svg -b transparent -w 1200 -H 800 --extractSvg

# Generate plugin architecture diagrams
mmdc -i plugin-architecture.md -o ../images/architecture/plugin-system.svg -b transparent -w 1200 -H 800 --extractSvg
mmdc -i plugin-architecture.md -o ../images/architecture/plugin-registration.svg -b transparent -w 1200 -H 800 --extractSvg

# Generate channel and provider architecture
mmdc -i channel-provider-architecture.md -o ../images/architecture/channel-architecture.svg -b transparent -w 1600 -H 900 --extractSvg
mmdc -i channel-provider-architecture.md -o ../images/architecture/message-flow.svg -b transparent -w 1200 -H 800 --extractSvg
mmdc -i channel-provider-architecture.md -o ../images/architecture/provider-architecture.svg -b transparent -w 1400 -H 800 --extractSvg

# Generate agent runtime architecture
mmdc -i agent-runtime-architecture.md -o ../images/architecture/agent-runtime.svg -b transparent -w 1400 -H 900 --extractSvg
mmdc -i agent-runtime-architecture.md -o ../images/architecture/session-lifecycle.svg -b transparent -w 1200 -H 800 --extractSvg
mmdc -i agent-runtime-architecture.md -o ../images/architecture/tool-execution.svg -b transparent -w 1200 -H 800 --extractSvg

# Generate data and state architecture
mmdc -i data-state-architecture.md -o ../images/architecture/configuration-flow.svg -b transparent -w 1400 -H 800 --extractSvg
mmdc -i data-state-architecture.md -o ../images/architecture/state-management.svg -b transparent -w 1200 -H 800 --extractSvg
mmdc -i data-state-architecture.md -o ../images/architecture/memory-system.svg -b transparent -w 1200 -H 800 --extractSvg
```

### Individual Diagram Generation

To generate a specific diagram, first extract the Mermaid diagram content and save it as a `.mmd` file, then:

```bash
# Example for system context diagram
mmdc -i system-context.mmd -o ../images/architecture/system-context.svg
```

## Diagram Inventory

### System Architecture
1. **system-context.svg** - C4 System Context diagram showing external actors and systems
2. **system-containers.svg** - C4 Container diagram showing major system components

### Plugin Architecture
3. **plugin-system.svg** - Component diagram showing plugin system components
4. **plugin-registration.svg** - Sequence diagram of plugin registration flow

### Channel and Provider Architecture
5. **channel-architecture.svg** - Component diagram of channel adapters
6. **message-flow.svg** - Sequence diagram of end-to-end message processing
7. **provider-architecture.svg** - Component diagram of provider system

### Agent Runtime Architecture
8. **agent-runtime.svg** - Component diagram of agent runtime components
9. **session-lifecycle.svg** - State diagram of agent session lifecycle
10. **tool-execution.svg** - Sequence diagram of tool execution flow

### Data and State Architecture
11. **configuration-flow.svg** - Flow diagram of configuration loading
12. **state-management.svg** - Component diagram of state management
13. **memory-system.svg** - Architecture diagram of memory system

## Usage in Documentation

The SVG diagrams can be embedded in documentation using:

```markdown
![System Context](../images/architecture/system-context.svg)
```

Or referenced via URL in web-based documentation platforms that support it.

## Maintenance

When updating architecture documentation:

1. Update the Mermaid diagram source in the markdown files
2. Regenerate SVGs using the commands above
3. Commit both the markdown changes and SVG exports
4. Ensure diagrams remain readable and accurate

## Troubleshooting

### Diagram Rendering Issues

If diagrams have rendering problems:

1. **Missing fonts**: Install the fonts mentioned in prerequisites
2. **Layout issues**: Adjust the `-w` (width) and `-H` (height) parameters
3. **Colors not showing**: Ensure `--extractSvg` is used to get the styling
4. **Blank output**: Check that the Mermaid syntax is valid with [Mermaid Live Editor](https://mermaid.live/)

### Mermaid CLI Installation Issues

```bash
# Alternative installation via yarn
yarn global add @mermaid-js/mermaid-cli

# Or via pnpm
pnpm add -g @mermaid-js/mermaid-cli

# If you get puppeteer errors
cd ~/.npm-global/lib/node_modules/@mermaid-js/mermaid-cli
npm rebuild puppeteer
```

### Font Issues on Linux

If you see warnings about missing fonts:

```bash
# Install Noto fonts for better coverage
sudo apt-get install fonts-noto fonts-noto-cjk

# Or on Fedora
sudo dnf install google-noto-sans-cjk-fonts
```

## Alternative: Online Tools

If Mermaid CLI is not available, you can use online tools:

1. [Mermaid Live Editor](https://mermaid.live/) - Paste Mermaid code and download SVG
2. [HackMD](https://hackmd.io/) - Create markdown with Mermaid diagrams and export
3. [GitHub](https://github.com/) - View diagrams in markdown files (but cannot download easily)

Remember to set transparent backgrounds and appropriate sizing for documentation use.
