# OpenClaw Architecture Documentation - Diagram Approach

This document describes the approach used for architecture diagrams in OpenClaw documentation.

## Diagram Format: Mermaid in Markdown

All architecture diagrams are embedded directly in markdown files using **Mermaid syntax**. This approach was chosen for the following benefits:

### ✅ Advantages of Mermaid in Markdown

1. **Single Source of Truth**: Diagrams and documentation text live together
2. **Version Control Friendly**: Changes to diagrams are tracked in git alongside text
3. **Editable Anywhere**: No special tools needed - just a text editor
4. **Preview Everywhere**: GitHub, GitLab, VS Code, and many other platforms render Mermaid natively
5. **Easy to Update**: Simple text changes to update diagrams
6. **Consistent Style**: Uses Mermaid's built-in theming for dark/light mode compatibility

### 📁 Files with Embedded Diagrams

| File | Diagram Count | Description |
|----------|----------|------------|
| `system-architecture.md` | 2 | C4 Model diagrams for system overview |
| `plugin-architecture.md` | 3 | Plugin system components and flows |
| `channel-provider-architecture.md` | 5 | Channel and provider patterns |
| `agent-runtime-architecture.md` | 7 | Agent execution and lifecycle |
| `data-state-architecture.md` | 5 | Configuration and state management |
| **TOTAL** | **22** | Comprehensive visual documentation |

### 🎨 Diagram Types Used

- **Flowcharts**: For processes and data flows
- **Sequence Diagrams**: For interaction flows and lifecycles
- **Component Diagrams**: For system architecture
- **State Diagrams**: For state machines and lifecycles
- **Architecture Diagrams**: Custom layouts for complex systems

### 🔄 Why Not SVG?

While SVGs were considered, we determined that:
- SVGs require generation step and maintenance overhead
- Difficult to version control effectively
- Requires additional tools (Mermaid CLI)
- Less accessible for contributors to edit
- No advantage over embedded Mermaid for most use cases

### ✨ Benefits Summary

**For Contributors:**
- Easy to fork and modify diagrams
- No special tools required
- Inline context with documentation

**For Viewers:**
- Rendered automatically in markdown viewers
- Consistent with documentation style
- Dark/light mode support in modern viewers

**For Maintainers:**
- Simplified workflow (no generation step)
- Atomic changes (text + diagrams together)
- Reduced maintenance burden

### 📝 Editing Diagrams

To edit any diagram:
1. Open the markdown file in your editor
2. Find the diagram in the ````mermaid` block
3. Make changes to the Mermaid syntax
4. Preview in a Mermaid-supported viewer or tool
5. Commit changes alongside documentation updates

### 🎯 Best Practices

1. **Keep diagrams focused**: One concept per diagram
2. **Use subgraphs**: Group related components logically
3. **Clear labels**: Ensure all nodes and edges are clearly labeled
4. **Consistent styling**: Use similar colors for similar concepts
5. **Dark mode friendly**: Avoid hardcoded white backgrounds
6. **Document complex diagrams**: Add explanations for non-obvious elements

### 🔧 Tools for Development

While Mermaid is text-based, these tools can help:
- **VS Code**: Built-in Mermaid preview with extensions
- **Mermaid Live Editor**: https://mermaid.live/ (online editor)
- **GitHub**: Native rendering in markdown files
- **GitLab**: Native rendering with Mermaid support

The embedded Mermaid approach provides the best balance of maintainability, accessibility, and viewing experience for OpenClaw's architecture documentation.
