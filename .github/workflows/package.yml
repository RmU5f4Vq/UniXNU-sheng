#!/usr/bin/env bash
# Device packaging script, identical in every device repository. Do not edit it: a board changes board.toml
# ([package] and the sections it names), never this file. Steps (run by "unixnu release"): deps, preflight,
# patches, build, pack, verify. Documentation: docs/porting/packaging.md in the UniXNU repository.
# usage: scripts/package.sh [--os 17|26|27|PROFILE] [--no-deps] [--check] [any other unixnu option]
#
# Local deviation (see "fix_mesa_patches" below): the pinned UniXNU revision ships a malformed Mesa patch,
# patches/mesa/turnip-frame-diagnostics.patch. Its new-file section carries "diff --git ..." and
# "--- /dev/null" but no "new file mode", so "git apply" treats it as a rename of dev/null and fails with
#   error: dev/null: No such file or directory
# upstream in build-boot-runtime.py. The file is repaired in the UniXNU working tree before "unixnu release"
# runs. The repair is idempotent, costs nothing when the patch is already correct, and is harmless when the
# patch, the profile or the patch list is absent. Delete this block once upstream ships the fix.
set -euo pipefail
DEVICE=$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)
UNIXNU=$(python3 - "$DEVICE/board.toml" <<'PY'
import sys, tomllib
package = tomllib.load(open(sys.argv[1], 'rb')).get('package', {})
if 'unixnu' not in package:
    sys.exit('board.toml needs [package] unixnu = "<path of the UniXNU checkout, relative to this repository>"')
print(package['unixnu'])
PY
)
case "$UNIXNU" in /*) ROOT=$UNIXNU ;; *) ROOT=$(cd "$DEVICE/$UNIXNU" && pwd) ;; esac

# Repair Mesa patches that create files without a "new file mode" header. git apply misreads such a section
# as a rename of dev/null; the same defect also makes the reverse-applied patch-identity check pass silently,
# so it only surfaces as a build failure. Runs before "unixnu release" because the build applies these patches
# from the UniXNU checkout (tree/patches is a symlink into it). Never fails a build: an unexpected error here
# leaves the source untouched and lets the build report the defect itself.
fix_mesa_patches() {
    python3 - "$ROOT" <<'PY'
import sys, pathlib

root = pathlib.Path(sys.argv[1])
targets = sorted(root.glob('patches/mesa/*.patch'))
# A previous failed run can leave an assembled tree behind; repair its copies too.
targets += sorted(root.glob('out/*/*/*/tree/patches/mesa/*.patch'))
repaired = []

for path in targets:
    try:
        text = path.read_text()
    except (OSError, UnicodeDecodeError):
        continue
    # A file can mix repaired and unrepaired sections, so decide per section below. This only skips files
    # that create nothing.
    if '--- /dev/null' not in text:
        continue

    lines = text.splitlines(keepends=True)
    out = []
    changed = False
    for index, line in enumerate(lines):
        out.append(line)
        if not line.startswith('diff --git '):
            continue
        # Collect the extended header lines that follow "diff --git" and precede the first hunk.
        header = []
        probe = index + 1
        while probe < len(lines) and not lines[probe].startswith('@@'):
            header.append(lines[probe])
            probe += 1
        creates = any(item.startswith('--- /dev/null') for item in header)
        declares = any(item.startswith('new file mode') for item in header)
        if creates and not declares:
            out.append('new file mode 100644\n')
            changed = True
    if changed:
        try:
            path.write_text(''.join(out))
        except OSError:
            continue
        repaired.append(str(path))

if repaired:
    print('package.sh: repaired Mesa patch header in ' + ', '.join(repaired), file=sys.stderr)
PY
}
fix_mesa_patches || true

exec python3 "$ROOT/unixnu" release -D "$DEVICE" "$@"
