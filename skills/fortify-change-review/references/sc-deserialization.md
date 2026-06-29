# Deserialization and Dynamic Execution

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Unsafe deserialization or insecure parser | `ObjectInputStream.readObject()`, `pickle.loads()`, `unserialize(`, `BinaryFormatter.Deserialize()`, or `yaml.load()` without SafeLoader called with data from an external source | Dynamic Code Evaluation: Unsafe Deserialization | CWE-502 | A08 Software and Data Integrity Failures | Actions 1, 2 |
| R2 | OS command injection | `subprocess.run(... shell=True)` with string input; `os.system(`, `Runtime.exec(` with concatenated user input; shell metacharacters (`;`, `|`, `` ` ``, `$()`) reachable from external input | Command Injection | CWE-78 | A05 Injection | Action 3 |
| R3 | Dynamic code execution via eval or equivalent | `eval(`, `new Function(`, `exec(`, `execfile(`, or `compile(` called with user-controlled content; untrusted script passed to a scripting engine | Dynamic Code Evaluation: Code Injection | CWE-94 | A05 Injection | Action 4 |


## Required Agent Actions

1. **Avoid unsafe deserialization formats** *(R1)* — prefer data-only formats (JSON, Protocol Buffers, MessagePack) over polymorphic serialization. If Java ObjectInputStream, PHP `unserialize()`, Python pickle, or .NET BinaryFormatter must be used, apply a strict type/class allowlist before deserialization begins.

2. **Use safe parsers for YAML and XML** *(R1)* — use `yaml.safe_load()` or SnakeYAML `new SafeConstructor()` instead of full-feature loaders. Full-feature YAML and XML parsers can instantiate arbitrary objects from untrusted input.

3. **Never construct shell commands from user input** *(R2)* — use parameterized subprocess APIs that accept an argument array, not a single string (e.g., `subprocess.run(["cmd", arg1])` not `subprocess.run("cmd " + arg1, shell=True)`). Allowlist permitted argument values where possible. If `shell=True` is present but the command string is fully hardcoded from server-side constants with no user input in scope, the risk is low; flag only when external input demonstrably reaches the shell command string.

4. **Restrict or sandbox dynamic code execution** *(R3)* — if `eval`, `Function()`, or `exec` is unavoidable, run in a hardened sandbox with no access to the filesystem, network, or process environment. Prefer dedicated expression evaluators with explicit safe contexts. If `eval` or `exec` is called with a fully hardcoded or server-generated string where no user input can flow in, the risk is low; flag only when user-controlled content demonstrably reaches the evaluation call.

## Completion Evidence

- [ ] *(R1)* No ObjectInputStream, pickle, unserialize, or BinaryFormatter used with untrusted data without a type allowlist; YAML and XML parsed with safe loaders only
- [ ] *(R2)* No shell command construction via string concatenation; subprocess invocations use array-based arguments without `shell=True`
- [ ] *(R3)* No `eval`, `Function()`, or `exec` called with user-controlled content outside a sandboxed context
