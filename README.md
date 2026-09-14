# SigBreaker - LLVM LIT

This repository offers a reproducible setup for evaluating SigBreaker's stability and performance across various scenarios. The archives contain:

- `original.zip`: original Windows LLVM binaries for `clang` and `lld` lit tests.
- `sigbreaker-1.0.zip`: Windows binaries scrambled with `SigBreaker 1.0`.
- `sigbreaker-linux-x86_64.tar.gz`: Linux x86-64 SigBreaker `clang-24` and `lld`, refreshed with ELF jump-table support, their driver aliases, and binary checksums. See [Linux reproduction](#linux-reproduction-llvm-2400git).

### Windows Test Results

> No debug information was used to transform these files
and all binaries are scrambled to their fullest

100% of the LLVM binaries are ran through SigBreaker and tests pass the same for both clang and lld. Your baseline test results may differ, but the the SigBreaker binaries should result in the same test results as baseline! Some of the LLVM tests fail by default as you can see the original binaries result in some failures.

#### LLD Tests

- Original, unmodified llvm binaries

```
Testing Time: 19.15s

Total Discovered Tests: 3040
  Unsupported      :   67 (2.20%)
  Passed           : 2489 (81.88%)
  Expectedly Failed:    1 (0.03%)
  Failed           :  483 (15.89%)
```

- SigBreaker-1.0

```
Testing Time: 20.77s

Total Discovered Tests: 3040
  Unsupported      :   67 (2.20%)
  Passed           : 2489 (81.88%)
  Expectedly Failed:    1 (0.03%)
  Failed           :  483 (15.89%)
```

#### Clang Tests

- Original, unmodified llvm binaries

```
Testing Time: 170.17s

Total Discovered Tests: 46200
  Skipped          :     8 (0.02%)
  Unsupported      :   318 (0.69%)
  Passed           : 45142 (97.71%)
  Expectedly Failed:    38 (0.08%)
  Failed           :   694 (1.50%)
```

- SigBreaker-1.0

```
Testing Time: 204.46s

Total Discovered Tests: 46200
  Skipped          :     8 (0.02%)
  Unsupported      :   318 (0.69%)
  Passed           : 45142 (97.71%)
  Expectedly Failed:    38 (0.08%)
  Failed           :   694 (1.50%)
```

## Windows Setup

Requirements:

- Git
- Visual Studios 2022
- cmake
- Python 3.x (> 3.8)

### Run Tests

```sh
# Run this in a powershell!
git clone --recursive -b llvmorg-20.1.0 https://github.com/llvm/llvm-project.git
cd llvm-project
cmake -S llvm -B build \
    -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;lldb;polly;bolt;mlir;openmp" \
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt" \
    -DCMAKE_BUILD_TYPE=Release

# Unzip the original binaries into the build folder
Expand-Archive -Path "../original.zip" -DestinationPath "./build/Release/bin/" -Force
cd "./build/Release/bin/"

#
# Run the original binaries to get baseline test results
#

# Run clang lit tests
python llvm-lit.py ../../../clang/test/ > original-clang-lit-results.txt
# Run lld lit tests
python llvm-lit.py ../../../lld/test/ > original-lld-lit-results.txt

#
# Run any of the other tests
#

Expand-Archive -Path "../../../../[zip file name here].zip" -DestinationPath . -Force

# Run clang lit tests
python llvm-lit.py ../../../clang/test/ > altered-clang-lit-results.txt
# Run lld lit tests
python llvm-lit.py ../../../lld/test/ > altered-lld-lit-results.txt
```

## Linux reproduction (LLVM 24.0.0git)

The current WSL build uses **Clang 24.0.0git / LLD 24.0.0**, from LLVM commit
[`4bf4bc65d198e54add11ce8e0bbae7ad51cbb0f8`](https://github.com/llvm/llvm-project/commit/4bf4bc65d198e54add11ce8e0bbae7ad51cbb0f8).
This is a development revision, separate from the LLVM 20.1.0 Windows setup above.
The binaries were refreshed on **September 13, 2026** with **CodeDefender 1.2.4**
and include ELF jump-table support.
The Linux binaries were built and tested on **Ubuntu 24.04.1 LTS, x86-64, under WSL**,
with GCC 13.3.0 and glibc 2.39. Use Ubuntu 24.04 x86-64, either native or in WSL;
the archive dynamically links against its system libraries.

The archive contains the two tested SigBreaker binaries, including `ElfDebugStrip`,
generated with seed `12648430` (`0xC0FFEE`). Their relative symlinks provide `clang`,
`clang++`, `clang-cpp`, `clang-cl`, `clang-dxc`, `ld.lld`, `lld-link`, `ld64.lld`, and
`wasm-ld`. LLVM support tools, resource headers, and unit-test executables are built
locally using the pinned revision below.

### Get the archive and build LLVM

Run these commands in **Bash on Ubuntu 24.04**. The build needs several GB of disk
space; the WSL run used 24 CPUs and approximately 30 GiB of RAM. Reduce `-j16` if needed.

```bash
sudo apt-get update
sudo apt-get install -y build-essential binutils cmake ninja-build git git-lfs \
    python3 python3-psutil zlib1g-dev libzstd-dev libxml2-dev libedit-dev

# Download only the Linux archive from Git LFS.
git lfs install
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/codedefender-io/sigbreaker.git
cd sigbreaker
git lfs pull --include="sigbreaker-linux-x86_64.tar.gz"
repo_dir="$PWD"
sha256sum -c sigbreaker-linux-x86_64.tar.gz.sha256

# Keep the LLVM build and test outputs outside the repository.
work_dir="$repo_dir/../sigbreaker-linux-work"
mkdir -p "$work_dir"
tar -xzf "$repo_dir/sigbreaker-linux-x86_64.tar.gz" -C "$work_dir"
(cd "$work_dir/sigbreaker-linux-x86_64/bin" && sha256sum -c SHA256SUMS)
cd "$work_dir"

git clone --filter=blob:none --no-checkout https://github.com/llvm/llvm-project.git
git -C llvm-project checkout --detach 4bf4bc65d198e54add11ce8e0bbae7ad51cbb0f8
cmake -G Ninja -S llvm-project/llvm -B llvm-project/build-linux \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_ENABLE_PROJECTS="clang;lld" \
    -DLLVM_TARGETS_TO_BUILD=X86 \
    -DLLVM_ENABLE_ASSERTIONS=ON \
    -DLLVM_INCLUDE_TESTS=ON \
    -DLLVM_INCLUDE_BENCHMARKS=OFF \
    -DLLVM_INCLUDE_EXAMPLES=OFF \
    -DLLVM_USE_LINKER=gold \
    -DLLVM_PARALLEL_LINK_JOBS=2 \
    -DLLVM_ENABLE_ZLIB=ON \
    -DLLVM_ENABLE_ZSTD=ON \
    -DLLVM_ENABLE_LIBXML2=ON \
    -DLLVM_ENABLE_LIBEDIT=ON \
    -DLLVM_ENABLE_CURL=OFF \
    -DLLVM_ENABLE_HTTPLIB=OFF \
    -DCLANG_ENABLE_OBJC_REWRITER=OFF
cmake --build llvm-project/build-linux --parallel 16 \
    --target clang-test-depends lld-test-depends
```

### Run baseline and SigBreaker tests

Continue in the same Bash session. Both runs use the same build directory and
support tools. The subshell saves and restores the original compiler and linker,
including when a command fails. Do not rebuild LLVM while the SigBreaker binaries
are installed: a build can replace them with the originals.

```bash
(
    set -euo pipefail
    cd "$work_dir"
    build_dir="$PWD/llvm-project/build-linux"
    results_dir="$PWD/results"
    mkdir -p "$results_dir"
    saved_dir=$(mktemp -d "$results_dir/original-binaries.XXXXXX")
    cp "$build_dir/bin/clang-24" "$saved_dir/clang-24"
    cp "$build_dir/bin/lld" "$saved_dir/lld"
    restore_binaries() {
        cp "$saved_dir/clang-24" "$build_dir/bin/clang-24"
        cp "$saved_dir/lld" "$build_dir/bin/lld"
    }
    trap restore_binaries EXIT

    run_lit() {
        local variant=$1 suite=$2 jobs=$3 status=0
        rm -f "$results_dir/$variant-$suite.json"
        python3 "$build_dir/bin/llvm-lit" -sv -j"$jobs" --timeout=1200 \
            -o "$results_dir/$variant-$suite.json" \
            "$build_dir/tools/$suite/test" \
            2>&1 | tee "$results_dir/$variant-$suite.log" || status=$?
        # Exit 1 can mean test failures; retain them for the comparison.
        if (( status > 1 )); then return "$status"; fi
        test -s "$results_dir/$variant-$suite.json"
    }

    run_lit baseline clang 16
    run_lit baseline lld 6

    # These are regular files; existing driver symlinks continue to point here.
    cp sigbreaker-linux-x86_64/bin/clang-24 "$build_dir/bin/clang-24"
    cp sigbreaker-linux-x86_64/bin/lld "$build_dir/bin/lld"
    (cd "$build_dir/bin" && sha256sum -c "$work_dir/sigbreaker-linux-x86_64/bin/SHA256SUMS")
    "$build_dir/bin/clang" --version
    "$build_dir/bin/ld.lld" --version

    run_lit sigbreaker clang 16
    run_lit sigbreaker lld 6
)
```

Compare every recorded test name and outcome, including expected failures and
unsupported cases. An identical failure in both runs still counts as parity;
inspect the logs for baseline failures on your system.

```bash
python3 - "$work_dir/results" <<'PY'
import json
from pathlib import Path
import sys

root = Path(sys.argv[1])
different = False
for suite in ("clang", "lld"):
    def outcomes(variant):
        tests = json.loads((root / f"{variant}-{suite}.json").read_text())["tests"]
        result = {test["name"]: test["code"] for test in tests}
        assert result and len(result) == len(tests), "Empty or duplicate test inventory"
        return result

    baseline, sigbreaker = outcomes("baseline"), outcomes("sigbreaker")
    changed = [name for name in sorted(baseline.keys() | sigbreaker.keys())
               if baseline.get(name) != sigbreaker.get(name)]
    print(f"{suite}: {len(baseline)} baseline / {len(sigbreaker)} SigBreaker tests; "
          f"{len(changed)} differences")
    for name in changed:
        print(f"  {name}: {baseline.get(name, 'MISSING')} -> {sigbreaker.get(name, 'MISSING')}")
    different |= bool(changed)
sys.exit(1 if different else 0)
PY
```

### Recorded Linux results

Fresh WSL runs of the September 13 refresh had identical test inventories and
outcomes, with no unexpected failures. Regression counts were the same for baseline
and SigBreaker:

| Suite | Passed | Expected failures | Unsupported |
|---|---:|---:|---:|
| Clang | 19,401 | 26 | 6,006 |
| LLD | 2,070 | 0 | 1,182 |

Separate, unchanged unit-test executables added 29,238 Clang passes and four LLD
passes in each run. Clang also reported six skipped unit tests, omitted from lit's
JSON. Counts can differ with host features; this build enables only the X86 backend.

The Linux transformation selected every decomposable function. The refreshed
decomposer rejected 20,150 Clang functions and 11,007 LLD functions, which retained
their original implementations. The pre-jump-table archive rejected 22,615 and
12,313 respectively, so the refresh leaves 2,465 fewer Clang functions and 1,306
fewer LLD functions untransformed. Support tools and unit-test executables were
unchanged.

## LLVM-LIT Tests

Each LLVM project comes with its own set of lit tests, designed to verify complex functionality and maintain backward compatibility. This is ideal for our needs, as these tests cover a vast portion of the executable paths within our obfuscated binaries. A huge thanks to LLVM for providing such extensive test coverage!

You can checkout the docs for [llvm-lit here](https://llvm.org/docs/CommandGuide/lit.html)

- https://github.com/llvm/llvm-project/tree/llvmorg-20.1.0/llvm/test
- https://github.com/llvm/llvm-project/tree/llvmorg-20.1.0/clang/test
- https://github.com/llvm/llvm-project/tree/llvmorg-20.1.0/lld/test

> There are tons of other tests and subtests for each project. An associate of Back Engineering Labs has worked on the llvm linker (lld) and expressed how complex their lit tests are. You can explore the ELF tests here: https://github.com/llvm/llvm-project/tree/llvmorg-20.1.0/lld/test/ELF
