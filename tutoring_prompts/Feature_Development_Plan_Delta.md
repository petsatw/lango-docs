# CONTEXT
Since this is a major refactor, please read in MY_CB and assess which of the existing clasess and configurations are relevant to this plan. Where new classes, packages or modules are being introduced make sure they align with the current architecture. Be very specific as this needs to provide the development team a high level of context in order to align to our success metrics of 90% first time compile rate and 80% first time build rate for this refactor. So thoroughness and high alignment with conventions, dependencies and architecture are essential for integrating this new State Machine architecture into the core of our existing project. 

# OUT OF SCOPE 
As per the architectural review, the classes that are OUT OF SCOPE are | StartSessionUseCase, GenerateDialogueUseCase, ProcessTurnUseCase, EndSessionUseCase, CoachOrchestrator - these and all associated tests will be eliminated

WHY
All session flow decisions now live in the YAML spec + interpreter.

# IN SCOPE
any helper that builds the **LLM prompt** or manages the queues and learning items; it isn’t duplicated by the FSM. 

# TASK 
Take the development sequence that you just provided and add the relevant context to each section so that the development team knows how to align the development task to the existing codebase. Be detailed and be thorough. 