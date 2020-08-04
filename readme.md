# R Debugger

This extension adds debugging capabilities for the R programming language to Visual Studio Code.
For further R support see e.g. [vscode-R](https://github.com/Ikuyadeu/vscode-R).
This extension depends on the R package [vscDebugger](https://github.com/ManuelHentschel/vscDebugger).

## Using the Debugger
* Install the **R Debugger** extension in VS Code.
* Install the **vscDebugger** package in R (https://github.com/ManuelHentschel/vscDebugger).
* If your R path is neither in Windows registry nor in `PATH` environment variable, make sure to provide valid R path in `rdebugger.rterm.*`.
* Press F5 and select `R Debugger` as debugger. With the default launch configuration, the debugger will start a new R session.
* To run a file, focus the file in the editor and press F5 (or the continue button in the debug controls)
* Output will be printed to the debug console,
expressions entered into the debug console are evaluated in the currently active frame
* During debugging in the global workspace it is often necessary to click the dummy frame
in the callstack labelled 'Global Workspace' to see the variables in `.GlobalEnv`.

*For Windows users: If your R installation is from [CRAN](http://cran.r-project.org/mirrors.html) with default installation settings, especially **Save version number in registry** is enabled, then there's no need to specify `rdebugger.rterm.windows`.*

## Installation
The VS Code extension can be installed from the .vsix-file found on https://github.com/ManuelHentschel/VSCode-R-Debugger/actions?query=workflow%3Amain.
To download the correct file, filter the commits by branch (develop or master), select the latest commit,
and download the file `r-debugger.vsix` under the caption "Artifacts".


To install the latest development version of the required R-package from GitHub, run the following command in R:
```r
devtools::install_github("ManuelHentschel/vscDebugger", ref = "develop")
```
To install from the master branch, omit the argument `ref`.

**Warning:** Currently there is no proper versioning/dependency system in place, so make sure to download both packages/extensions from the same branch (Master/develop) and at the same time.


## Features

![features.png](images/features.png)

The debugger includes the following features:
1. Run and debug R Code, using one of three debug modes (details see below)
2. View scopes and variables of the currently selected stack frame.
The value of most variables can be modified in this view.
3. Add watch expressions that are evaluated in the selected stack frame on each breakpoint/step.
4. View and browse through the call stack when execution is pause.
5. Set breakpoints and break on errors.
6. Control the program flow using *step*, *step in*, *step out*, *continue*.
7. Output generated by the program is printed to the debug console (filtering out text printed by the browser itself).
8. Add a modified version of `print` and `cat` that also prints a link to the file and line where the text was printed.
9. Allow the execution of arbitrary R code in the currently selected stack frame.

## How it works
The debugger works as follows:
* An R process is started inside a child process
* The R package `vscDebugger` is loaded.
* The Debugger starts and controls R programs by sending input to stdin of the child process
* After each step, function call etc., the debugger calls functions from the package `vscDebugger` to get info about the stack/variables

The output of the R process is read and parsed as follows:
* Information sent by functions from `vscDebugger` is encoded as json and sent via a TCP socket.
These lines are parsed by the VS Code extension and not shown to the user.
* Information printed by the `browser()` function is parsed and used to update the source file/line highlighted inside VS Code.
These lines are also hidden from the user.
* Everything else is printed to the debug console

## Launch config
The behaviour of the debugger can be configured with the entry `"debugMode"`,
which can be one of the values `"function"`, `"file"`, and `"workspace"`.
The intended usecases for these modes are:

* `"workspace"`: Starts an R process in the background and sends all input into the debug console to the R process (but indirectly, through `eval()` nested in some helper functions).
R Files can be run by focussing a file and pressing `F5`.
The stack view contains a single dummy frame.
To view the variables in the global environment it is often necessary to click this frame and expand the variables view!
This method is 'abusing' the debug adapter protocol to some extent, since the protocol is apparently not designed for ongoing interactive programming in a global workspace.
* `"file"`: Is pretty much equivalent to launching the debugger with `"workspace"` and immediately calling `.vsc.debugSource()` on a file.
Is hopefully the behaviour expected by users coming from R Studio etc.
* `"function"`: The above debug modes introduce significant overhead by passing all input through `eval()` etc.
and use a custom version of `source()`, which makes changes to the R code in order to set breakpoints.
To provide a somewhat 'cleaner' method of running code, this debug mode can be used.
The specified file is executed using the default `source` command ad breakpoints are set by using R's `trace(..., tracer=browser)` function, which is more robust than the custom breakpoint mechanism.
<!-- The call to `main()` is entered directly into R's `stdin`, hence there are no additional functions on the call stack (as is the case when entering `main()` into the debug console). -->

The remaining config entries are:
* `"workingDirectory"`: An absolute path to the desired work directory. Defaults to the workspace folder.
* `"file"`: Required for debug modes `"file"` and `"function"`. The file to be debugged/sourced before calling the main function.
* `"mainFunction"`: The name of the main function to be debugged. Must be callable without arguments.
* `"allowGlobalDebugging"`: Whether to keep the R session running after debugging and evaluate expressions from the debug console.
Essential for debug moge `"workspace"`, recommended for `"file"`, usually not sensible for `"function"`.

## Debugging R Packages
In principle R packages can also be debugged using this extension.
For this to work, the proper source information must be retained during installation of the package
(check `attr(attr(FUNCTION_NAME, 'srcref'), 'srcfile')`).
I personally do not know a bullet proof way to achieve this, but the following things might help:
* The package must be installed from source code (not CRAN or `.tar.gz`)
* The flag `--with-keep.source` should be set
* Extensions containing C code seem to cause problems sometimes

In order to use the modified `print` and `cat` functions,
import the `vscDebugger` extension in your package,
assign `print <- vscDebugger::.vsc.print` and `cat <- vscDebugger::.vsc.cat`,
and deactivate the modified `print`/`cat` statements in the debugger settings.
Don't forget to remove these assignments after debugging.

## Warning
In the following cases the debugger might not work correctly/as expected:
* Calls to `trace()`, `tracingstate()`:
These are used to implement breakpoints, so usage might interfere with the debugger's breakpoints
* Calls to `browser()` without `.doTrace()`:
Usually, these will be recognized as breakpoints, but they might cause problems in some circumstances (e.g. watch expressions)
* Custom `options(error=...)`: the debugger uses its own `options(error=...)` to show stack trace etc. on error
* Any form of (interactive) user input in the terminal during runtime (e.g. `readline(stdin())`), since
the debugger passes all user input through `eval(...)`.
* Code that contains calls to `sys.calls()`, `sys.frames()`, `attr(..., 'srcref')` etc.:
Since pretty much all code is evaluated through calls to `eval(...)` these results might be wrong. <!-- This problem might be reduced by using the "functional" debug mode --> <!-- (set `debugFunction` to `true` and specify a `mainFunction` in the launch config) -->
If required, input in the debug console can be sent directly to R's `stdin` by prepending it with `###stdin`.
* Any use of graphical output/input, stdio-redirecting, `sink()`
* Extensive use of lazy evaluation, promises, side-effects:
In the general case, the debugger recognizes unevaluated promises and preserves them.
It might be possible, however, that the gathering of information about the stack/variables leads to unexpected side-effects.
Especially watch-expressions must be safe to be evaluated in any frame,
since these are passed to `eval()` in the currently viewed frame any time the debugger hits a breakpoint or steps through the code.

## Known Issues
The following topics could be improved/fixed in the future.

Variables/Stack view
* Summarize large lists (min, max, mean, ...)
* Refine display of variables (can be customized by `.vsc.addVarInfo`, default config is to be improved)

Breakpoints
* Auto adjustment of breakpoint position to next valid position
* Conditional breakponts, data breakpoints
* Setting of breakpoints during runtime (currently most of these are silently ignored)

General
* Improve error handling
* Handling graphical output etc.?
* Attach to currently open R process instead of spawning a new one?

Give user more direct access to the R session:
* Use (visible) integrated terminal instead of background process,
use `sink(..., split=TRUE)` to simultaneously show stdout to user and the debugger
* Pipe a copy of stdout to a pseudo-terminal as info for the user

If you have problems, suggestions, bug fixes etc. feel free to open an issue at
https://github.com/ManuelHentschel/VSCode-R-Debugger/issues
or submit a pull request.
Any feedback or support is welcome :)