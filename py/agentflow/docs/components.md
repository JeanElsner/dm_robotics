# AgentFlow Core Components

[TOC]

## Policy

A Policy is the base-class for all agents in AgentFlow. Its primary method is:

```python
def step(self, timestep: dm_env.TimeStep) -> np.ndarray:
  ...
```

All other methods on `Policy` are for book-keeping and visualization.

## Option

An `Option` is a `Policy` which can also:

*   Decide if it's eligible to stop (`pterm`)
*   Consume a runtime-argument (E.g. a pose to reach towards)
*   Produce a result (`OptionResult`)

These enable it to be used within an AgentFlow graph by other `MetaOptions`,
which all know about the basic life-cycle semantics of options. E.g. `af.While`
will repeat a given option until the condition is `False`, or the option's
`pterm` samples to `True`.

*What are `runtime-args` and why do we need them?*

In the original options literature, an `Option` is an atomic, temporally
extended action. I.e. it's not-parameterized. A higher-level agent can invoke an
option, but it can't tell it what to do. In the robotics use-case that's often
not general enough, so we designed in the ability for an `Option` to be
parameterized by (i.e. conditioned-on) an arbitrary additional variable. Args
should therefore be thought of as a communication mechanism for agentflow nodes.

**Protocol:** Args are similar to information in the observation (and are
actually passed via timestep.observation), but instead of being generated by the
environment, it comes from a higher-level `Option` or `SubTask`.

If an `Option` wants to receive args it defines an `arg_spec` with the
appropriate spec. This signifies that at runtime it will expect an observation
with this shape under the (auto-generated) `arg_key` for the option. It is the
parent node's responsibility to check for arg_specs on it's children and
provide them where required.

Beyond this basic API, AgentFlow doesn't enforce how args are used. An `Option`
can choose to use the arg only during `on_selected`, or it can condition on it
every step. It is the user's responsibility to ensure the parent passes args
when necessary (the easiest thing is to stick to the API and pass it every step
if `arg_spec` is defined).

**Example:** A concrete example is cartesian reaching: if you want to define an
af node that reaches, but have the target be passed in at runtime by a
higher-level (possibly learned) source, you'd define the appropriate `arg_spec`
on this option to signify that it accepts 6D vectors.

## SubTask

`SubTask` is the AgentFlow mechanism for training an option policy *in-situ*,
i.e. in the context of a higher-level graph. A `SubTask` and `Policy` are
composed via `SubTaskOption` into an option that can be used like a regular
Option in an AgentFlow graph (see below).

The `SubTask` defines everything that an option needs to run, except the policy
itself. This decoupling of option-methods from policy-methods allows a subtask
to train an arbitrary RL agent, which doesn't have to know it's actually living
in an AgentFlow graph. From the agent's perspective, it just wakes up in a
particular state and runs an episode until seeing a LAST timestep signifying the
end of episode.

**Specs.** Like a regular RL-task, `SubTask` must also define observation and
action specs that define the task for the agent that it runs. The `arg_spec` on
the SubTask is different: this is equivalent to `Option.arg_spec`, and
signifies how the resulting `SubTaskOption` can be controlled **by the parent**.

If the child `Policy` is actually an `Option` (which is a perfectly valid thing
to do), then it's the SubTask job to pack an arg in the timestep corresponding
to `option.arg_spec`.

Yes this can be confusing, but it is the price of allowing deep hierarchy and
parameterized options. The simple rule is: SubTasks *tell* action and
observation spaces to their children, *ask* arg spaces from their children, and
*tell* arg spaces to their parent.

**Timestep transformation.** Rather than having to generate observations from
scratch, a `SubTask` receives the timestep from the parent (or base
environment), and modifies it via `parent_to_agent_timestep` to suit the needs
of the child `Policy` it is training.

After stepping the policy with this timestep, it post-processes the policy's
action with `agent_to_parent_action` to comply with the action_spec of the
context in which the `SubTask` is defined.

A SubTask can be learned or engineered, or a mixture of both. E.g. in the
[DPGfD](https://sites.google.com/corp/view/dpgfd-insertion/home) paper we used a
SubTask that had a learned CNN-based reward classifier, injected bottle-neck
activations from this model for visual features (both via
`TimestepPreprocessors`), and a hand-defined cartesian-velocity action space.

**Action Spaces**. A `SubTask` also allows the action space of the sub-task to
be different than the action space of the parent task. This is achieved by
mapping the action an agent returns to it to an action that the larger task
(environment) can consume.

In AgentFlow the role of action-mapping in a SubTask is handled by an
`ActionSpace`. This class has a `project` method which works as follows:

`parent_action = space.project(child_action)`.

## SubTaskOption

A `SubTaskOption` is an `Option` which composes a `SubTask` and a `Policy`. It
simulataneously defines the life-cycle methods to run in an AgentFlow graph, as
well as an RL-environment for the Policy it contains. See above for details.
User's shouldn't have to make new SubTaskOptions, only SubTasks and Policies to
solve those subtasks.

## MetaOption

A MetaOption is an option that contains other options, AgentFlow comes with

*   `Sequence`: Runs a list of options one after the other, each option is run
    until it signals termination from `pterm`.

## Options Core Library

### Basic Options

*   `FixedOp`: Returns a fixed action for N steps.
*   `ConcurrentOption`: Steps N `Options` in sequence at each step. I.e. each
    option is given the same step the `ConcurrentOption` is given, this is the
    sense in which they are run concurrently.
*   `LambdaOption`: An option that is parameterizable by functions, these
    functions can run when the option is selected, stepped or terminates. The
    return value of the function can be sent to other options and the action
    this option returns comes from another option it delegates to.
*   `SubTaskOption`: Combines a `SubTask` and a policy to make an `Option`, see
    the section on [SubTask](#subtask) for more information.
*   `DelegateOption`: A implementation base class that delegates its method
    implementations to another `Option` instance.
*   `PadOption`: A delegating option that applies an `ActionSpace` to the action
    of the delegate option. This can be used to project the actions of two
    options into the same action space before merging the actions together.
*   `RandomOption`: Emits uniform-random actions.

### Control-Flow Options

*   `Cond`: A conditional. This uses a function to decide which of two options
    to select, this choice can happen on option selection or at every step.
*   `Repeat`: Repeats another option a given number of times.
*   `While`: Repeats another option while a user-supplied function returns True.

### Action Spaces

An action space is the semantics attached to the dimensions of actions.
AgentFlow was developed in the context of robotics use cases, in which it's
common to have multiple actions spaces, such as cartesian and joint action
spaces.

Action spaces can also be used to decompose parts of an environments action
dimensions into parts which individual options can deal with separately.

*   `CastActionSpace`: Used to change numpy array dtype.
*   `FixedActionSpace`: Creates a reduced-dimensionality action space, by
    yielding a constant value for some dimensions of the larger action space.
*   `prefix_slicer`: An action space that consists of some dimensions of another
    action space, according to their names.
*   `SequentialActionSpace`: An action space that consists of a sequence of
    other action spaces, this allows the composition of action spaces.
*   `ShrinkToFitActionSpace`: Scales action values towards zero until they fit
    in a given action specification. This assumes the action spec's min <= 0 <=
    max
*   `IdentityActionSpace`: Performs no modification to actions it is given.