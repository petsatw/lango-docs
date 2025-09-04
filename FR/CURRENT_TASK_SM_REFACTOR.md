### Step 4: Implement Persistence for Mastered Items  
**Goal:** Persist only at mastery, per crash‑resume rules. 
Inject LearningRepository into DefaultActionRegistry and pass it into AddWordToLearnedPoolAction; call repository.saveQueues(ctx.queues) at the end of that action.
 
1. **Use** `LearningRepository` from `RepositoryModule` (already in code).  
2. **Inject** `LearningRepository` into the action registry and pass it to `AddWordToLearnedPoolAction`.  
3. In `AddWordToLearnedPoolAction.invoke(ctx)`, after moving the item, call `learningRepository.saveQueues(ctx.queues).getOrThrow()`  
4. On app init (`MainViewModel.start()`), keep loading via `learningRepository.loadQueues()` and `runner.initialize(queues)`

> **See SM_REFACTOR_SPEC §1** for `schemaVersion` and JSON shape  
> **See SM_REFACTOR_SPEC §5** “Persistence Integration”

Test Plan: assert saveQueues called once with updated Queues