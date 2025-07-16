# Development guide for re engine

This guide contains development guidelines.

## General notes

Renaming classes that we migrate with prefix `Re` (old prefix is `RB`).
This way we can track what has been migrated.
But please note, not all `Re` classes are fully migrated, usually this means we did a first pass and initial cleanings.

On refactoring menu items we have a practice to put:
- (R) for refactorings so users know this will preserve behavior
- (T) for transformations, similarly so they know this will not necessarily preserve behavior

## Refactorings

Generally, we first need to migrate refactorings to new architecture to start working on drivers. This is because drivers rely on the new API of refactorings and fine-grained access to refactorings.

Migrating refactorings includes:

- Creating and extracting the `prepareForExecution` step. This step is almost always present and is implicit and scattered among preconditions and execution. This step should prepare everything for precondition checking and transformation. This includes: retrieving objects from the model (refactoring namespace), parsing methods, doing some initial transformations, etc.
- Identifying preconditions that are performed during the transformation step. This usually boils down to some checks that raise refactoring warning or error. These should be moved to the precondition step.
- Reifying preconditions. For each precondition that is not a standalone object we should create one. Check the classes in the hierarchy of ReCondition. These objects are really useful for driver and having good error messages.
- Enforcing reuse of existing refactorings. Next step is to inspect the transformation steps and investigate if some of them could be replaced with existing refactorings OR if some of them can be encapsulated to create new refactorings. If we identify some steps that can be refactored to use existing transformation/refactoring, it is good practice to search for similar method usage in other refactorings and further increase code reuse and safety.
- Try to make the transformation step declarative. To do this we leverage ReCompositeRefactoring or ReCompositeTransformation classes and their approach to compose refactoring steps into new refactorings. Check references to ReCompositeRefactoring. 
- Make refactorings decorators of transformations. For refactorings that have several behavior preserving preconditions and/or additional steps they perform to ensure behavior will be preserved, we make them decorators of transformation. This is very useful for UX since then we will be able to offer to user to use either refactoring or transformation depending on the usecase. 

## UI/UX

Improving experience: 
- Sane default values
- Default selection of accept/cancel buttons.
— the widgets we use should be in Spec2 (`Sp` prefix)
— We should make sure that the refactoring logic does not depend on Spec. 
- Make sure to package the UI layer in a Refactoring-UI package

For the flow of the refactorings through driver. This is the case that it is right now:

1. Driver is created and invoked through a command. A command supplies it with scopes and a minimum set of inputs. It is the drivers responsibility to gather all other required inputs from the user.
2. Driver should create a new instance of refactoring that it will execute later. (Note: some drivers may operate with a few refactorings, in those cases, pick a default one here). Sometimes the refactoring is partially instantiated (it is not provided all the information at once) but first created then further more information is provided. For example, for renameMethod, the refactorings is created before the new name is asked to the user, so that the driver can ask the refactoring if the new name is correct. This is the job of the refactoring to do that - we should not duplicate this logic in the driver.
3. Next, the driver should check applicability preconditions that are not related to user input. (For example if we want to rename a method, we should first check if the method to be renamed exists before asking the user for a new name).
4. Following that, the driver should gather all required inputs from the user:
  - a checkbox for each configuration that are related to the refactoring being performed. This is where we show the user defaults that are carefully chosen so that user doesn't need to change the majority of the time. For example, for Pull Up Method refactoring here we put a checkbox that says "Remove duplicate methods from sibling classes as well" which is marked by default,
  - any additional info that might be required (like pick a target class or pick a new name, etc.).
5. The driver then passes the inputs to the refactorings and checks preconditions that are related to user input (for rename method, we will check if the name is valid, if it already exists, etc.)
6. If the name is not valid, we return to step 4. and show the user error message of why the input was not valid and ask it for a new name. If the user cancels we should abort the refactoring. We are in this loop until all inputs are validated.
7. After validating user inputs, driver then checks breaking change preconditions. Based on failed preconditions driver creates a selection dialog.

   7.1. If the conditions do not hold the dialog contains messages of the failed breaking change preconditions and a list of choices. Each choice is an action that can be performed on the system. Take a look at RenameClass driver and its `handleBreakingChanges` method.

   7.2. Based on error messages user picks an action which he wants to perform, which can either go to step 8. or open some other flow, depending on the selection (these flows include: opening browser of references/senders/etc.).
   
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

In the future, we can think of merging these, but I think we should start somewhere and I think this is a good start. And of course we improve in the future. (The suggestion for merging is to have options 2 and 3 merged, where we have default choice (default refactoring) that is previewed by default as well, and user can switch choices and thus change preview of the changes as well. If we manage to do that, we could even move configuration to the same window and have only two steps: 1. gather input 2. preview changes with default configuration and default choice action which user can tweak. This way the process feels more alive and is something we should strive for.
