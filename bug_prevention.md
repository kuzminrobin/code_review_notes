[About the author and ways to get in touch](https://docs.google.com/document/d/1TmA8cXRxb_9lDnCREfYuIeGpJy9sMsXSpFVMK-0PwNo/edit).  
# Bug Prevention and Code Reviewing vs. Testing With Bug Fixing.

From time to time I hear something like this:
* "Let’s not spend too much time for code-reviewing, let’s focus on bug fixing". 
* "Migrate to a newer compiler? Not this year, we have too much bugs to fix by the year end".
* "Adding (in addition to Debug and Release) the extra build configurations (e.g. Maximum optimization for speed, Maximum optimization for size), and running the tests against them, is a huge overkill. That’s not what we want and/or have time for. What we need is to write more and better tests".
* "Adding extra compilers? Spending time for changing our code to compile successfully on other compilers? Just for the sake of extra compiling (and never use the generated binaries)? We definitely have no resources for that, did you see how many bugs are on our list?"

Even more sadly, at times I see in the people’s eyes (and never hear that explicitly): "We don’t want _new_ bugs to be found/disclosed/filed, we want to get rid of the _known_ ones (and to make an impression that very few bugs are found and we fix them efficiently, and that the code is fine/safe). Please stop your bug search. Please let's conclude the code review and move on. Please don’t remove the suppression of the warnings. Please don’t ask us to turn that static analyzer on".

Here I make a deep sigh.

On the one hand I feel a common misconception about what is more important and efficient for the project and how it affects the bugs on the people’s list.  
On the other hand (and this is a larger and more serious issue) I see that sometimes people are awfully afraid of bugs. Mostly they have strong objective reasons for their fear such as  
the annual bonus for a large team depends on the number of bugs opened and resolved,  
the audit and certification can fail or get delayed for the large company or product because of the extra bugs found,  
a number of clients are using the product and rely on it, new bugs reported can result in a loss of clients, reputation damage.

Here I’m trying to settle the misconception and to dissipate the fear. For that let’s consider 

## The Life Cycle of a Bug

The bug typically appears when the programmer writes the code. Many bugs die when the programmer re-reads one’s own code, and when the programmer drives the code through the preprocessor + compiler + linker, followed by the debugger (the programmer makes sure that the code works as expected). Then optionally either the same programmer or someone else may write and run the test for the added change (in the Test-Driven Development the tests are written _before_ changing the code).  
When the programmer is satisfied with the change he or she passes the code to the code review. In general the team of code-reviewers possesses broader knowledge than a single programmer and the team helps the programmer to sweep out another group of bugs.  
In parallel to the code review, typically a build and test-run for multiple configurations takes place on the build servers as a part of the continuous integration process, which helps to eliminate another group of bugs.  

Once the code review is over and the build succeeds, the change is merged to the code repository. However some bugs still survive. Moreover, a new group of bugs can appear here. As has been heard from [Thomas McGuire](https://cppcon2017.sched.com/thomas.mcguire2) at [one of the trainings](https://cppcon2017.sched.com/event/BhdM/debugging-and-profiling-c-code-on-linux): "The most sophisticated bugs are a result of the joined efforts". Two correct independent changes (made by two independent programmers), when they meet in the code repository, can contradict to each other and create new bugs, often rather complex.  

Finally there are bugs in the repository.  
Not all of them get control immediately. Not all of them after getting control show themselves right away. Often they stay inactive ("kind"), at times they form the category of what I call _risks_ rather than bugs, e.g. the code works fine as long as certain data type has certain size (for instance `sizeof(wchar_t)` is `2`). And once the code is migrated to a newer compiler (or is ported to a different platform, etc.) the bug causes the difference in behavior, and that difference can be detected much later during the run, and result in a long investigation. Or the bug can cause an undefined behavior, which can result in many different weird things. In the best case it is a reliable crash. In more complicated case it is different behavior during different runs (of the same binary), on different machines, with different compiler arguments, with different optimization levels, with different compilers. In the most complicated and severe cases it is a corruption or loss of data upon coincidence of a number of rare events, e.g. a corruption (and full loss) of a database file once in 6 months, plus-minus 4 months (and nothing crashes, justs starts failing).

Eventually the bugs that get control and show themselves, get noticed/detected by the test infrastructure. They are chased down and fixed. However the fix at times is very expensive. Because e.g. it is difficult to reproduce the bug (in order to start working on it), it can be difficult to chase the bug down during the debugging (especially in multi-threaded/multi-process/multi-machine distributed environment), it can be difficult to not break the other things with the fix, the fix can require running it through the debugger, writing extra tests, running the tests locally, code review, continuous integration build, another round of the full test run (followed by a detection of a minor typo, followed by a start over).

Finally the _reported_ bugs get fixed. And the bugs' life cycle is over.  
The key word here is "*reported*". This means that only the bugs that _get the control_, and only those that _show themselves_ (the "angry" ones), and only those that are _caught_ by the test infrastructure, get fixed. My experience tells me that it is a small amount compared to the number of bugs that _stay_ in the code repository after the run through the test infrastructure. And those that are fixed are fixed at a very high price, because of the reasons listed above, and, more importantly, because the quality of the bug reports by the test infrastructure is very low. The test infrastructure mostly reports the bugs like this: "At the moment M we expected to see A but instead we saw B". Nearly never it reports the bugs like this: "In the source file F, line L, we see `Printers` but there should be `Printer&` (the last character should be `&` instead of `s`)".  
The test infrastructure fights only the active ("angry") bugs. It fights with the bugs in the second half of the bug's life cycle. The bugs are always ahead of the test infrastructure, and the infrastructure always catches them up.

As opposed to the test infrastructure, it is order of magnitude more efficient  
to fight all the bugs at once, including those that currently don't get control, or don't show themselves,  
to fight with the bugs in the beginning of their life cycle, starting with the first compilation,  
to get the bug reports that provide the exact file path, line number, and the matter of the bug.  
And that's what is done by what I call

## The Bug Prevention
With _Bug Prevention_ I mostly mean creating an infrastructure for and running the code through  
* as many [static](https://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis) and [dynamic](https://en.wikipedia.org/wiki/Dynamic_program_analysis#Memory_error_detection) analyzers as reasonable,  
* as many different compilers as reasonable,  
* as _new_ compilers as possible ([Mateusz Pusz](https://cppnow2019.sched.com/speaker/mateusz.pusz) said that if the code is old and requires the older standard of the language, then the new compiler can be used in the compatibility mode with the old standard, but the new compiler will still report more issues compared to the old compiler, the new compiler will partially act as a static code analyzer),  
* as high warning level as possible,  
* as many extreme configurations as reasonable (no optimization, highest optimization for speed, highest optimization for size, etc.),  

and adding all that to the continuous integration process.

After or in parallel to all those automated tools of bug prevention, the code still needs to be run through a _thorough code review_. The code review catches the bugs that no other tool and infrastructure can catch. And in my personal case reviewing the code is more efficient bug fight than working as a part of the test infrastructure (writing and running tests, fixing the bugs reported by the infrastructure). This can be opposite for other people, depends on the passion and prior experience.

## Summary  
Here I would like to remind a phrase from the beginning of this post. But this time I will reword a little bit: 

> "Let’s not spend too much time for code-reviewing and bug prevention, let’s focus on bug fixing".

This is one of the most absurd phrases that I continue to hear regularly. It sounds to me similar to this: "(The side of our ship is leaking) Let's not spend too much time for patching the side, let's focus on getting the water out of the ship (with buckets and bathtubs)".  
I apologize if my opinion hurts you.

## What to Remember
* Having equal resources, { the bug prevention and code reviewing } _alone_ is order of magnitude more efficient than the { whole test infrastructure and bug fixing } _alone_. Ideally both should be present. But if the team has limited resources then the bug prevention and code reviewing should have higher priority.
* Some work environments induce the code owners to dislike the bug preventers and productive code reviewers. It is inefficient or even risky if the code owner manages the bug preventer or manages a more efficient code reviewer.
* Bug prevention is not certain type of testing or debugging. It is a separate activity.
