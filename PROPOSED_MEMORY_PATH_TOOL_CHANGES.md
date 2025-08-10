## Proposal: Enable Serena tools to read/write under external memory-path (no deletes)

Goal: Allow `list`, `read`, and `create/write` operations for files located under the centralized memory bank path specified by `--memory-path`, restricted to the current `{PROJECT_NAME}` subtree, while preserving existing in-repo safety for all other paths. Deletion is explicitly out of scope.

### Scope and Safety
- Allowed external areas (absolute paths):
  - `{MEMORY_PATH}/{PROJECT_NAME}/**` → read/list/write (no delete)
  - `{MEMORY_PATH}/templates/**` → read/list only (writes blocked; deletes never allowed)
- No delete operations are exposed by tools for any memory-bank paths.
- All other operations remain unchanged and restricted to the active project root.

---

### Files to change

1) `src/serena/agent.py`
- Add a small getter to expose the configured base memory path to tools.

Changes:
```startLine:150:endLine:170:src/serena/agent.py
    def get_memory_base_path(self) -> str | None:
        """
        Returns the configured base memory path ("--memory-path") or None if not set.
        The effective memory bank root for the current project is
        {get_memory_base_path()}/{project_name}/.
        """
        return self._base_memory_path
```

2) `src/serena/tools/file_tools.py`

- New private helper: resolve if an input path is inside the memory bank for the active project or the shared templates, and return absolute targets.

Insert (near top-level imports or just above the tools that use it):
```startLine:1:endLine:40:src/serena/tools/file_tools.py
from pathlib import Path

...

    def _resolve_in_memory_bank(self, input_path: str) -> tuple[str | None, Path | None, Path | None]:
        """
        Classify an absolute input path if it resides under one of the permitted memory-bank roots.

        Returns a tuple (kind, base_root, resolved_abs_target) where:
        - kind: one of {"project", "templates"} or None if not allowed
        - base_root: the base root path for the matched kind
        - resolved_abs_target: absolute resolved path of input_path if inside an allowed root; otherwise None
        """
        base = self.agent.get_memory_base_path()
        if not base:
            return (None, None, None)
        memory_project_root = Path(base) / self.project.project_name
        templates_root = Path(base) / "templates"
        ipath = Path(input_path)
        if ipath.is_absolute():
            try:
                resolved = ipath.resolve()
                if resolved.is_relative_to(memory_project_root.resolve()):
                    return ("project", memory_project_root, resolved)
                if resolved.is_relative_to(templates_root.resolve()):
                    return ("templates", templates_root, resolved)
            except Exception:
                pass
        return (None, None, None)
```

- Update `ReadFileTool.apply` to support absolute paths under memory bank:

Before:
```startLine:41:endLine:52:src/serena/tools/file_tools.py
        self.project.validate_relative_path(relative_path)
        result = self.project.read_file(relative_path)
```

After:
```startLine:41:endLine:52:src/serena/tools/file_tools.py
        # Allow absolute paths inside the memory bank (project or templates)
        kind, base_root, abs_target = self._resolve_in_memory_bank(relative_path)
        if kind in {"project", "templates"} and abs_target is not None:
            result = abs_target.read_text(encoding="utf-8")
        else:
            self.project.validate_relative_path(relative_path)
            result = self.project.read_file(relative_path)
```

- Update `ListDirTool.apply` to support absolute paths under memory bank:

Before:
```startLine:97:endLine:106:src/serena/tools/file_tools.py
        self.project.validate_relative_path(relative_path)
        dirs, files = scan_directory(
            os.path.join(self.get_project_root(), relative_path),
            relative_to=self.get_project_root(),
            recursive=recursive,
            is_ignored_dir=self.project.is_ignored_path,
            is_ignored_file=self.project.is_ignored_path,
        )
```

After:
```startLine:97:endLine:109:src/serena/tools/file_tools.py
        # If an absolute path inside the memory bank (project or templates) is given, list from there; otherwise keep project-relative behavior
        kind, base_root, abs_target = self._resolve_in_memory_bank(relative_path)
        if kind in {"project", "templates"} and abs_target is not None and abs_target.is_dir():
            dirs, files = scan_directory(
                abs_target,
                relative_to=base_root,
                recursive=recursive,
                is_ignored_dir=lambda p: False,
                is_ignored_file=lambda p: False,
            )
        else:
            self.project.validate_relative_path(relative_path)
            dirs, files = scan_directory(
                os.path.join(self.get_project_root(), relative_path),
                relative_to=self.get_project_root(),
                recursive=recursive,
                is_ignored_dir=self.project.is_ignored_path,
                is_ignored_file=self.project.is_ignored_path,
            )
```

- Update `CreateTextFileTool.apply` to support absolute paths under memory bank:

Before:
```startLine:68:endLine:75:src/serena/tools/file_tools.py
        self.project.validate_relative_path(relative_path)
        abs_path = (Path(self.get_project_root()) / relative_path).resolve()
        will_overwrite_existing = abs_path.exists()
        abs_path.parent.mkdir(parents=True, exist_ok=True)
        abs_path.write_text(content, encoding="utf-8")
```

After:
```startLine:68:endLine:78:src/serena/tools/file_tools.py
        kind, base_root, abs_target = self._resolve_in_memory_bank(relative_path)
        if kind == "project" and abs_target is not None:
            abs_path = abs_target
        else:
            self.project.validate_relative_path(relative_path)
            abs_path = (Path(self.get_project_root()) / relative_path).resolve()
        will_overwrite_existing = abs_path.exists()
        abs_path.parent.mkdir(parents=True, exist_ok=True)
        abs_path.write_text(content, encoding="utf-8")
```

- Plan/Review file generation in the project's memory bank `develop/` directory:
  - With the above `CreateTextFileTool` enhancement, Serena can create:
    - `{MEMORY_PATH}/{PROJECT_NAME}/develop/{TASK_ID}/plan-mode.md`
    - `{MEMORY_PATH}/{PROJECT_NAME}/develop/{TASK_ID}/review-mode.md`
  - Parent directories must be auto-created via `abs_path.parent.mkdir(parents=True, exist_ok=True)`.

---

### Rationale
- The safety gate is enforced in `Project.validate_relative_path`. Tools currently call it unconditionally, which blocks access to external paths.
- By introducing a narrowly scoped exception for absolute paths under `{MEMORY_PATH}/{PROJECT_NAME}`, we enable Serena-driven maintenance of the centralized memory bank without weakening repository safety, while explicitly forbidding delete operations.

### Notes
- `--memory-path` is read via `.cursor/mcp.json` when launching the Serena MCP server; at runtime it is available inside `SerenaAgent` and passed down to tools.
- The memory-bank allowance deliberately uses `{MEMORY_PATH}/{PROJECT_NAME}` (not just the `foundation` subfolder) to cover the full project memory structure (e.g., `develop/{TASK_ID}/...`, `docs-rule/`, `templates/`).

### Out of scope
- No changes to `Project.validate_relative_path` (keeps default sandbox for repo-relative paths).
- No delete operations or tools that remove files from the memory bank or templates.
- No new public tools for arbitrary external access beyond the memory-bank project subtree.


