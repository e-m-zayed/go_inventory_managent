# Project
- inventory managenent app.
- refrence:
    - https://github.com/inventree/InvenTree
    - it should be used only as a refrence for features and structure
    - the language and the stack are diffrent.
- InvenTree was just one refrence you should search for more refrences.
- the domain is inventory and warehouse management.
- we want our app to be thorough and cover the most widespread use cases regaredless of their scale. -> ease of use is a priority

# Stack
- golang -> we are using the standard library for the backend 
- we will minimize the use of external libraries as much as possible.
- htmx: https://github.com/donseba/go-htmx
- daisyui: https://daisyui.com/htmx-component-library/?lang=en
- we are using Ent as an orm: https://github.com/ent/ent , https://entgo.io/docs/tutorial-setup/.


# Localization

- The system will be multi language -> all languages are first class.
- we are starting with English and Arabic as a beginning and adding more later
- we are using i18n: https://github.com/nicksnyder/go-i18n .
- each language localization should be in toml file.
- for a start the app will be only english -> we will add more later.
- the ui logic should be bi-directional from the start -> neither RTL and LTR should be harcoded.



# building

- all the run, build, test and similar processes should be handled by magefile build system.
- after every phase the agent should do a clean build.
- we are doing a local binary no docker nor complicated containers.

# git discipline
- every minor change should be tracked with git closely.
- write commit messages in Mitchellh's style
- 




# Pitfalls
- Never suggest other librairies and packages that are not part of the stl outside the ones we are using.
- Don't botch up the bi-directional ui and the loclaization.
- Don't do a change without a thorough check of the code and doing a web research.
