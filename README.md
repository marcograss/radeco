# radeco

Radeco is the radare decompiler tool using the [radeco-lib](https://github.com/radare/radeco-lib) rust crate.

[![Build Status](https://travis-ci.org/radare/radeco.svg)](https://travis-ci.org/radare/radeco)

## Usage

To get up and running, make sure you have a working rust compiler. Building is
fairly simple using cargo.

`cargo build`

radeco provides a small help menu

```bash
radeco. The radare2 decompiler.

Usage:
  radeco <file>
  radeco [options]
  radeco run [options] [<file>]
  radeco --shell <file>
  radeco --output=<output> <file>
  radeco --version

Options:
  --help                 Show this screen.
  --version              Show version.
  --shell                Run interactive prompt.
  --output=<mode>        Select output mode.
  --from-json            Run radeco based on config and information
                         from input json file. Needs an input file.
  --json-builder         Interactive shell used to build the config
                         json for radeco. When used with run, the
                         config generated is automatically used to
                         run radeco rather than dumping it to a file.
```

radeco can be run on binaries using json as input using: 

`radeco run --from-json <path/to/json/file>`

`radeco --build-json` provides an interactive utility to generate the json.
Using the same command with run however, generates the json and uses it to run
radeco.

```bash
# Build json and run radeco
radeco run --build-json

# Build json and save to file
radeco --build-json

# Run radeco using a prebuild json
radeco run --from-json <json>
```

Example of json used to power radeco:

```json
{
  "bin_name": "/bin/ls",
  "esil": null,
  "addr": "sym.main",
  "name": "radeco_simple2",
  "outpath": ".",
  "stages": [
    0,
    1,
    2,
    3,
    4,
    5
  ],
  "verbose": true
}
```

Explanation of the above fields:
* `bin_name`: Relative path to the binary to be analysed
* `esil`: radeco allows users to load raw esil [WIP]
* `addr`: Address from where the instructions to be analysed must be loaded.
  This field can take an address such as `0xbadcode` or any symbols identified
  by r2 such as `sym.main`
* `name`: Name of the analysis. This name is used to name the generated output
  files as well as the input json if saved.
* `outpath`: Path to the directory where radeco will output the results of the
  analysis. This can be either dot files for graphs such as SSA and CFG, or
  text files for esil and instructions.
* `stages`: list of stages in the radeco pipeline. Output from one stage is
  automatically fed as input for the next stage of the analysis. More
  information about the pipeline is described in the pipeline section of this
  document.
* `verbose`: Prints out current stage and other debug information.

## Pipeline

radeco describes the various stages of the analysis in terms of a pipeline.
The output of one stage of the pipeline is directly fed into the next stage.
For this reason, the user must ensure that the output of the previous stage of
the pipeline is compatible with the stage directly after it. Here we briefly
describe the stages of the pipeline as it is now. Note that these are very
likely to change in the future as more analysis are currently being added to
expand radeco's arsenal.

1. __R2__: This is the read stage where radeco spawns an instance of radare2 and
   reads instructions in form of esil from it. This stage can be skipped if
   the esil input is directly given by the user.
2. __Parse esil__: This is the stage where the esil from the previous stage is
   passed and converted into an intermediate form. Note that this is not the
   ssa representation and is never used in any of the analysis. This  forms
   the base of the ssa form and can help debug the ssa construction when
   needed. Also, it is slightly more readable that the SSA graph. This stage
   requires that the Pipeout is initialised with the esil instructions to be
   used by this stage.
3. __Build CFG__: radeco takes the IR generated from the previous stage and breaks
   it up into Basic Blocks and constructs a control flow graph of the same.
   This stage is required for the SSA construction which is usually the next
   phase. This phase can be helpful to analyse the control flow structuring
   the program without paying much attention to the instructions actually
   present inside the Basic Blocks.
4. __Build SSA__: The CFG generated by radeco is used to construct the Static
   Single Assignment (SSA) form. This is radeco's primary Intermediate
   Representation and is used for all subsequent analysis. The output for this
   stage is a dot file which can be used with a graphing utility like graphviz
   to generate a human readable SSA graph. All analysis, Dead Code Elimination
   and Verify expect SSA to be in the Pipeout.
5. __Constant Propagation__: radeco currently supports constant propagation to
   propagate constants and generate a better SSA graph. It is recommended to
   run Dead Code Elimination after this step to ensure all the unreachable
   nodes are removed and cleaned up.
6. __Dead Code Elimination__: This pass expects the previous stage of the pipeline
   to output an SSA. Provided with an SSA, DCE cleans up all the nodes that
   are detected to be unreachable or are removed.
7. __Verify__: This pass is essentially to ensure the integrity of the SSA form.
   Therefore it expects SSA to be the output of the previous stage of the
   pipeline. Note that this might make radeco panic! and abort. In such cases,
   please file a bug report on this repository.

## License

(TODO)
