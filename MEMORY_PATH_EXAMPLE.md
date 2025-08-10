# Memory Path Feature Example

This document demonstrates how to use the new `--memory-path` feature in Serena.

## Overview

The `--memory-path` option allows you to specify a base directory for storing Serena's memory files. The final memory path will be automatically constructed as: `{base_path}/{project_name}/foundation`

## Usage

### Default Behavior (No Custom Path)

When you don't specify a memory path, Serena uses the default location:

```bash
uv run serena-mcp-server --context ide-assistant --project /path/to/project
```

This stores memories in: `/path/to/project/.serena/memories/`

### Custom Base Memory Path

To use a custom base memory path:

```bash
uv run serena-mcp-server --context ide-assistant --project /path/to/project --memory-path /custom/memory/base
```

This stores memories in: `/custom/memory/base/{project_name}/foundation`

### Examples

#### Example 1: Shared Memory Bank

If you have multiple projects and want to store all memories in a centralized location:

```bash
# Base memory path
--memory-path /Users/quannl/01.Work/91.Coding/04.Cursor/d2-cursor-v2/memory-bank

# For project "sendo-autoweb"
# Final path: /Users/quannl/01.Work/91.Coding/04.Cursor/d2-cursor-v2/memory-bank/sendo-autoweb/foundation

# For project "beauty-clinic" 
# Final path: /Users/quannl/01.Work/91.Coding/04.Cursor/d2-cursor-v2/memory-bank/beauty-clinic/foundation
```

#### Example 2: Project-Specific Organization

```bash
# Base memory path
--memory-path /Users/quannl/01.Work/91.Coding/04.Cursor/d2-cursor-v2/memory-bank

# Project: /Users/quannl/01.Work/91.Coding/04.Cursor/02.MCP/serena
# Project name extracted: "serena"
# Final memory path: /Users/quannl/01.Work/91.Coding/04.Cursor/d2-cursor-v2/memory-bank/serena/foundation
```

## Examples

### Example 1: Shared Memory Directory

If you want to share memories across multiple projects:

```bash
# Project A
uv run serena-mcp-server --project /path/to/project-a --memory-path /shared/memories

# Project B  
uv run serena-mcp-server --project /path/to/project-b --memory-path /shared/memories
```

Both projects will now share the same memory directory.

### Example 2: External Storage

Store memories on an external drive or network location:

```bash
uv run serena-mcp-server --project /path/to/project --memory-path /Volumes/External/memories
```

### Example 3: Temporary Memory Location

For testing or temporary work:

```bash
uv run serena-mcp-server --project /path/to/project --memory-path /tmp/serena_memories
```

## Implementation Details

The feature works by:

1. **CLI Level**: The `--memory-path` option is added to the `start-mcp-server` command
2. **Factory Level**: The `SerenaMCPFactorySingleProcess` accepts and stores the memory path
3. **Agent Level**: The `SerenaAgent` receives and stores the memory path
4. **Manager Level**: The `MemoriesManager` uses the custom path if provided, otherwise falls back to the default

## Backward Compatibility

The feature is fully backward compatible:
- If no `--memory-path` is specified, it uses the original default behavior
- Existing projects continue to work without any changes
- Memory files are automatically created in the specified directory

## Memory File Structure

Regardless of the path, memories are stored as `.md` files:
```
/custom/memory/path/
├── memory1.md
├── memory2.md
└── ...
```

Each memory file contains markdown-formatted content that Serena can read and write. 

# Create virtual environment
uv venv

# Activate environment
# macOS/Linux:
source .venv/bin/activate
# Windows:
.\.venv\Scripts\activate

# Install all dependencies with extras
uv pip install --all-extras -r pyproject.toml -e .