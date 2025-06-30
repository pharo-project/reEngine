# Development guide for re engine

This guide contains development guidelines.

## General notes

Renaming classes that we migrate with prefix `Re` (old prefix is `RB`).
This way we can track what have been migrated.
But please note, not all `Re` classes are fully migrated, usually this means we did a first pass and initial cleanings.

## UI/UX

Improving experience: 
- sane default values
- default selection of accept/cancel buttons.
— the widgets we use should be in Spec2 (`Sp` prefix)
— We should make sure that the refactoring logic does not depend on spec. 
- make sure to package the UI layer in a Refactoring-UI package

For the flow of the refactorings through driver. This is the case that it is right now:

1. Driver is created and invoked through command. Command supplies it with scopes and a minimum set of inputs. It is drivers responsibility to gather all other required inputs from the user.
2. Driver should create a new instance of refactoring that it will execute later. (Note: some drivers may operate with a few refactorings, in those cases pick a default one here). Sometimes the refactoring is partially instantiated (it is not provided all the information at once) but first created then further more information is provided. For example, for renameMethod, the refactorings is created before the new name is asked to the user, so that the driver can ask the refactoring if the new name is correct. This is the job of the refactoring to do that - we should not duplicate this logic in the driver.
3. Next, driver should check applicability preconditions that are not related to user input. (For example if we want to rename a method, we should first check if the method to be renamed exist before asking user for a new name).
4. Following that, driver should gather all required inputs from the user:
  - a checkbox per each configuration that are related to the refactoring begin performed. This is where we show the user defaults that are carefully chosen so that user doesn't need to change them majority of the time. For example, for Pull Up Method refactoring here we put a checkbox that says "Remove duplicate methods from sibling classes as well" which is marked by default,
  - any additional info that might be required (like pick a target class or pick a new name, etc.).
5. Driver then passes the inputs to the refactorings and check preconditions that are related to user input (for rename method, we will check if the name is valid, if it already exists, etc.)
6. If the name is not valid, we return to step 4. and show the user error message of why the input was not valid and ask it for a new name. If the user cancels we should abort the refactoring. We are in this loop until all inputs are validated.
7. After validating user inputs, driver then checks breaking change preconditions. Based on failed preconditions driver creates a selection dialog.

   7.1. If the conditions do not hold the dialog contains messages of the failed breaking change preconditions and a list of choices. Each choice is an action that can be performed on the system. Take a look at Rename Class driver and its handleBrekaingChanges method.

   7.2. Based on error messages user picks action which he wants to perform, which can either go to step 8. or open some other flow depending on the selection (these flows include: opening browser of references/senders/etc.).
8. Driver executes the refactoring and previews the changes to the user (this is by default executed if the preconditions from the step 8. hold).
9. User selects which changes to apply. When confirming the selection, selected changes are performed on the image.

We also need to pay attention to create least possible amount of UI interaction.
This means that if we have to collect target class, and some configuration, we do it in one go.
Multiple dialogs are very boring and break the flow, make the process feel slow, etc.
This implies that each refactoring that gathers some kind of input and configuration will need a custom dialog.
My thinking is that we could have a generic input dialog that can have configuration + inputs section, where subclasses fill configuration options and overwrite input section.

One additional thing that is very important, we need to make sure that happy path is very fast to execute. So we should put focus on all buttons that go with happy path so that developers can act quickly and use shortcuts to perform refactorings.

Based on this we can see that each driver should have 3 types of UI interaction at maximum. Those are:

1. Gather inputs and configuration,
2. Handling breaking changes,
3. Previewing changes.

In the future we can think of merging these, but I think we should start somewhere and I think this is a good start. And of course we improve in the future. (The suggestion for merging is to have options 2 and 3 merged, where we have default choice (default refactoring) that is previewed by default as well, and user can switch choices and thus change preview of the changes as well. If we manage to do that, we could even move configuration to the same window and have only two steps: 1. gather input 2. preview changes with default configuration and default choice action which user can tweak. This way the process feels more alive and is something we should strive for.
