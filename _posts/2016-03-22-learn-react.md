# Tips from react tutorial

1. `props` are immutable: they are passed from the parent and are "owned" by the parent.
2. `props` and `state` are two important concepts.

### Let's go through each one and figure out which one is state. Simply ask three questions about each piece of data:

1. Is it passed in from a parent via props? If so, it probably isn't state.
2. Does it change over time? If not, it probably isn't state.
3. Can you compute it based on any other state or props in your component? If so, it's not state. 



