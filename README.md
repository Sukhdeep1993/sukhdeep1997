
from __future__ import annotations

import os
import shutil
import subprocess
import sys
from pathlib import Path


def load_dotenv_file(repo_root: Path) -> dict[str, str]:
    env_path = repo_root / ".env"
    loaded: dict[str, str] = {}
    if not env_path.exists():
        return loaded

    for raw_line in env_path.read_text(encoding="utf-8", errors="ignore").splitlines():
        line = raw_line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        key, value = line.split("=", 1)
        key = key.strip()
        value = value.strip().strip('"').strip("'")
        if key:
            loaded[key] = value
    return loaded


def build_local_php_ini(repo_root: Path, php_bin: str) -> Path:
    php_exe = Path(php_bin).resolve()
    php_dir = php_exe.parent
    source_ini = php_dir / "php.ini"
    runtime_dir = repo_root / ".runtime"
    runtime_dir.mkdir(exist_ok=True)
    local_ini = runtime_dir / "php.dev.ini"

    if source_ini.exists():
        content = source_ini.read_text(encoding="utf-8", errors="ignore")
    else:
        content = "[PHP]\n"

    lines = content.splitlines()
    out_lines: list[str] = []
    saw_ext_dir = False
    saw_pdo_mysql = False

    for line in lines:
        stripped = line.strip()
        lower = stripped.lower().replace(" ", "")

        if lower.startswith("extension_dir="):
            out_lines.append(f'extension_dir="{(php_dir / "ext").as_posix()}"')
            saw_ext_dir = True
            continue

        if lower in {"extension=pdo_mysql", ";extension=pdo_mysql"}:
            out_lines.append("extension=pdo_mysql")
            saw_pdo_mysql = True
            continue

        out_lines.append(line)

    if not saw_ext_dir:
        out_lines.append(f'extension_dir="{(php_dir / "ext").as_posix()}"')
    if not saw_pdo_mysql:
        out_lines.append("extension=pdo_mysql")

    local_ini.write_text("\n".join(out_lines) + "\n", encoding="utf-8")
    return local_ini


def main() -> int:
    repo_root = Path(__file__).resolve().parent
    public_dir = repo_root / "public"

    php_bin = shutil.which("php")
    if php_bin is None:
        print("Error: php is not installed or not on PATH.", file=sys.stderr)
        return 1

    env = os.environ.copy()
    for key, value in load_dotenv_file(repo_root).items():
        env.setdefault(key, value)

    env.setdefault("JONESAUTO_DB_DSN", "mysql:host=127.0.0.1;port=3307;charset=utf8mb4")
    env.setdefault("JONESAUTO_DB_USER", "jonesauto")
    env.setdefault("JONESAUTO_DB_PASS", "jonesauto")
    env.setdefault("APP_HOST", "localhost")
    env.setdefault("APP_PORT", "8000")
    local_ini = build_local_php_ini(repo_root, php_bin)

    host = env["APP_HOST"]
    port = env["APP_PORT"]
    url = f"http://{host}:{port}/"

    print(f"Starting Jones Auto DBMS on {url}")
    print(f"Public dir: {public_dir}")
    print(f"Using php.ini: {local_ini}")
    print("Press Ctrl+C to stop.")

    cmd = [php_bin, "-c", str(local_ini), "-S", f"{host}:{port}", "-t", str(public_dir)]
    try:
        return subprocess.call(cmd, cwd=str(repo_root), env=env)
    except KeyboardInterrupt:
        print("\nServer stopped.")
        return 0


if __name__ == "__main__":
    raise SystemExit(main())
