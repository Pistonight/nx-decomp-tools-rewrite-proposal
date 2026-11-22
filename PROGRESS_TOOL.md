# Proposal for `progress.py` rewrite

The idea is to unify all 3 versions:
- The old progress.py that works with the CSV: https://github.com/open-ead/nx-decomp-tools/blob/master/progress.py
- The progress.py in SMO fork that works with the new YAML format: https://github.com/LynxDev2/nx-decomp-tools/blob/progress-tool/progress.py
- The new progress tool in the SMO fork: https://github.com/LynxDev2/nx-decomp-tools/blob/progress-tool/viking/src/tools/progress.rs

The new unified tool will be in the new viking repo: https://github.com/open-ead/viking

## Comparison of Feature Sets

This shows the current feature sets of each of the 3 versions of the progress tool

Y = Yes

| Feature | Old progress.py | New progress.py | New progress.rs | Unified |
|-|-|-|-|-|
| CSV Format | Y | - | - | Y |
| YAML Format | - | Y | Y | Y |
| Print total number of functions | Y | Y | Y | Y |
| Print total size of code | Y | Y | Y | Y |
| Print number of functions by status | Y | Y | Y | Y |
| Print size of code by status | Y | Y | Y | Y |
| Flags for filtering by status | Y | Y | - | Y |
| Print progress per namespace (top only, like ksys,uking...) | - | - | Y | Y |
| Print progress per object | - | - | - | Y |
| Compare the progress of 2 revisions | - | - | Y | Y |
| When comparing, only show objects that changed | - | - | Y | Y |
| Output Format = CSV Line (comma separated) | Y | Y | - | Y |
| Output Format = "STATUS=PERCENTAGE" | - | - | Y | Y |

## CLI

Based on the features above, here's the proposed CLI flags: (help displayed with --help)

```
Show decompilation progress

Usage: progress [OPTIONS]

Options:
      --version <VERSION>
          Specify the version to use

  -s, --status <STATUS_SET>
          Filter the output by status
          
          The value can either be a full status name like `NotDecompiled`, or a string of status codes like `OMmUL`
          
          [default: OMm]

  -d, --diff [<REV>]
          Show difference of the progress in the current worktree's file list against another Git revision; requires Git

  -g, --group-by <GROUP>
          Show per-group progress

          Possible values:
          - namespace: Group by top namespace
          - object:    Group by object as listed in the file list

  -a, --all
          When diffing per-group, show unchanged groups

  -f, --format <FORMAT>
          Specify output format

          Possible values:
          - pretty: Human-readable, color-coded format
          - json:   JSON
          
          [default: pretty]

  -h, --help
          Print help (see a summary with '-h')
```

## Specification

### File List Reading

The new tool will reuse the new viking's `config` and `fl` modules to read the file list based on the repo's config.
The `fl` module already handles CSV and YAML format transparently.

The file list is already organized into `fl::Object`, each stores a `text_section` of `fl::Info`.

### Stat Collector

The stat collector visits functions and collect their stats. If `--group-by`, the stat is collected per-group. Otherwise, one stat collector collects the stat of all functions.

The collector keeps track of:
- total number of functions collected
- total size of code collected
- number of functions by status (map of status -> number)
- size of code by status (map of status -> size)

### Display

> TODO: when a prototype is ready, the examples here will be replaced by real output/screenshots

If `--group-by`, it will display the information by group first, then at the end the overall progress. Without `--group-by`, it will just display the overall progress.

Each group will be displayed in two lines, one for count and one for size, the columns will be aligned. The group name is left-truncated if it does not fit the terminal width.

```
GROUP NAME (40)      | O: 12 (30%)    | M: 1 (2.5%) | m: 0 (0%)   | U: 27 (67.5%) |  L: 0 (0%)
           (4.0kB)   |    1.2kB (30%) | ... 
```

The overall status is displayed at the end:
```
Decompiled:         xxx, xxx.xxMB
NonMatchingMajor:
NonMatchingMinor:
NotDecompiled:
Wip:
Library:

Total: 12000 functions, xxx.xxMB
```

If --diff, each number `A` is printed as `A -> B (delta)`, like `12 -> 13 (+1)  (30% -> 32.5% (+2.5%))`

### Output Format

comma and key-value format are kept since they exist in some of the versions now. We should decide if we want to keep them, or use JSON as the only machine-readable format (in which case it can also be just a `--json` flag)

- pretty: human-readable, aligned, color coded format (if piped to non-terminal, it will be non color coded and not aligned)
- json: the whole report is going to be represented as JSON, including the stat for each group and diff delta
- key-value: only the overall stat fields are displayed, as KEY=VALUE lines
- comma: only the overall stat fields are displayed, as VALUE1,VALUE2,VALUE3,... (just one line, no header line)


