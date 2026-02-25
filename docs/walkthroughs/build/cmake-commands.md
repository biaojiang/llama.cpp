# CMake Command Log

## Configure

```bash
cmake -S . -B build
```

## Build

```bash
cmake --build build
```

## Notes

- `compile_commands.json` is generated in `build/` after configure.
- Optional symlink for editor/LSP support:

```bash
ln -sf build/compile_commands.json compile_commands.json
```
