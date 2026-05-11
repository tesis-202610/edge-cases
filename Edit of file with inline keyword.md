Let me help you modify that file. Here are the commands to find and replace all `static inline void` with `static void`:

## Step 1: Backup the original file (optional but recommended)

```bash
cp /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c \
   /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c.backup
```

## Step 2: Replace `static inline void` with `static void`

Use `sed` to do an in-place replacement:

```bash
sed -i 's/static inline void/static void/g' \
  /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c
```

## Step 3: Verify the changes

Check how many replacements were made:

```bash
grep -c "static void" /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c
```

And verify no `static inline void` remains:

```bash
grep "static inline void" /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c
```

(This should return nothing if the replacement was successful)

---

## Step 4: Create the test corpus and re-extract

```bash
# Create a minimal compdb with just this file's compilation command
python3 - <<'PY'
import json, os

src = os.path.join(os.environ["FUCHSIA_SRCDIR"], "out/default/compile_commands.json")
with open(src) as f:
    all_cmds = json.load(f)

# Filter for the u_format_s3tc.c file
filtered = [cmd for cmd in all_cmds if "u_format_s3tc.c" in cmd.get("file", "")]

dst = "/tmp/test_u_format_s3tc.json"
with open(dst, "w") as f:
    json.dump(filtered, f)

print(f"Wrote {len(filtered)} command(s) to {dst}")
if len(filtered) == 0:
    print("WARNING: No commands found! Check if the file path in compdb matches.")
PY
```

## Step 5: Extract the modified corpus

```bash
export CORPUS_TEST="$HOME/corpus_test_u_format_s3tc"
rm -rf "$CORPUS_TEST"

cd "$MLGO_DIR"
extract_ir \
  --cmd_filter="^-Os|-Oz$" \
  --input=/tmp/test_u_format_s3tc.json \
  --input_type=json \
  --llvm_objcopy_path="$LLVM_INSTALLDIR/bin/llvm-objcopy" \
  --output_dir="$CORPUS_TEST"

echo "Extraction complete. Module count:"
python3 - <<'PY'
import json, os
p = os.path.join(os.environ["CORPUS_TEST"], "corpus_description.json")
if os.path.exists(p):
    d = json.load(open(p))
    print("Modules:", len(d.get("modules", [])))
else:
    print("corpus_description.json not found!")
PY
```

## Step 6: Evaluate the modified file against your trained policy

```bash
export TEST_REPORT="$HOME/test_u_format_s3tc_report.csv"

cd "$MLGO_DIR"
PYTHONPATH=$PYTHONPATH:. python3 \
  compiler_opt/tools/generate_default_trace.py \
  --data_path="$CORPUS_TEST" \
  --policy_path="$OUTPUT3/saved_policy" \
  --output_performance_path="$TEST_REPORT" \
  --gin_files=compiler_opt/rl/inlining/gin_configs/common.gin \
  --gin_bindings=clang_path="'$LLVM_INSTALLDIR/bin/clang'" \
  --gin_bindings=llvm_size_path="'$LLVM_INSTALLDIR/bin/llvm-size'" \
  --sampling_rate=1.0

echo "Test report:"
cat "$TEST_REPORT"
```

## Step 7: Compare before and after

Your original result for this file was:

```
module_name: x64-shared/obj/third_party/gfxstream/src/guest/mesa/src/util/format/format.u_format_s3tc.c.o
size under heuristic: 29803
size under trained policy: 16902
```

After running Step 6, check the new report to see how removing `inline` affects inlining decisions. The sizes should differ from the original.

---

## Quick Cheat Sheet (All at once)

If you want to run everything in sequence:

```bash
# 1. Modify file
sed -i 's/static inline void/static void/g' \
  /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c

# 2. Verify
echo "Replacements made:"
grep -c "static void" /home/maurive2014/fuchsia/third_party/mesa-migrating/src/src/util/format/u_format_s3tc.c

# 3. Create test compdb
python3 - <<'PY'
import json, os
src = os.path.join(os.environ["FUCHSIA_SRCDIR"], "out/default/compile_commands.json")
with open(src) as f:
    all_cmds = json.load(f)
filtered = [cmd for cmd in all_cmds if "u_format_s3tc.c" in cmd.get("file", "")]
with open("/tmp/test_u_format_s3tc.json", "w") as f:
    json.dump(filtered, f)
print(f"Found {len(filtered)} command(s)")
PY

# 4. Extract corpus
export CORPUS_TEST="$HOME/corpus_test_u_format_s3tc"
rm -rf "$CORPUS_TEST"
cd "$MLGO_DIR"
extract_ir \
  --cmd_filter="^-Os|-Oz$" \
  --input=/tmp/test_u_format_s3tc.json \
  --input_type=json \
  --llvm_objcopy_path="$LLVM_INSTALLDIR/bin/llvm-objcopy" \
  --output_dir="$CORPUS_TEST"

# 5. Evaluate
export TEST_REPORT="$HOME/test_u_format_s3tc_report.csv"
cd "$MLGO_DIR"
PYTHONPATH=$PYTHONPATH:. python3 \
  compiler_opt/tools/generate_default_trace.py \
  --data_path="$CORPUS_TEST" \
  --policy_path="$OUTPUT3/saved_policy" \
  --output_performance_path="$TEST_REPORT" \
  --gin_files=compiler_opt/rl/inlining/gin_configs/common.gin \
  --gin_bindings=clang_path="'$LLVM_INSTALLDIR/bin/clang'" \
  --gin_bindings=llvm_size_path="'$LLVM_INSTALLDIR/bin/llvm-size'" \
  --sampling_rate=1.0

# 6. See results
cat "$TEST_REPORT"
```

This will give you a direct before/after comparison showing how the trained policy handles the code with `inline` removed!