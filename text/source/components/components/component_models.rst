.. _model-comps:

Component Models
----------------

In this section, we'll summarize how component models are different
from the previous models we've created.  This discussion will be
broken into two parts.  The first part will focus on acausal modeling
and how it provides a framework for schematic-based,
component-oriented modeling where conservation equations are
automatically generated and enforced.  The second part will provide an
overview of how the topics in this chapter impact, mostly
syntactically, the definition of component models.

However, before we dive into that discussion, it is worth taking some
time to talk about terminology.  In this chapter, we've created two
different types of models.  The first type represent individual
effects (*e.g.,* resistors, capacitors, springs, dampers).  The other
type represent more complex assemblies (*e.g.,* circuits, mechanisms).

Before we discuss some of the differences between these different
types of models, let's introduce some terminology so we can refer to
them precisely.  A *component model* is a model that is used to
encapsulate equations into a reusable form.  By creating such a model,
an instance of the component can be used in place of the equations it
contains.  A *subsystem model* is a model that is composed of
components or other subsystems.  In other words, it doesn't
(generally) include equations.  Instead, it represents an assembly of
other components.  Typically, these subsystem models are created by
dragging, dropping and connecting component and other subsystem models
schematically.  While component models are "flat" (they don't contain
other components or subsystems, only equations), subsystem models are
hierarchical.

We'll often refer to a subsystem model as a *system model*.  A system
model is a model that we expect to simulate.  In simulating it, the
Modelica compiler will traverse the hierarchy of the model and note
all the variables and equations present throughout the hierarchy.
These are the variables and equations that will be used in the
simulation.  Of course, in order for there to be a unique solution,
the system model (like any non-``partial`` model), must be
:ref:`balanced <balanced-components>`.

Note that a subsystem model *can* include equations.  There is no rule
against it in Modelica.  But most of the time models tend to be
composed either of equations or other components/subsystems.  It is
actually a good idea to avoid putting equations in models containing
subcomponents or subsystems because doing so means that some information
about the model will be "invisible" when looking at a diagram of the
subsystem.  One possible exception to this could be the use of
``initial equation`` sections in subsystems.

With that discussion of terminology out of the way, let's dive
into discussions about component models.

.. _acausal-modeling:

Acausal Modeling
^^^^^^^^^^^^^^^^

We'll start with a discussion about acausal modeling.  We touched on
this topic very briefly in the chapter on :ref:`connectors`.  Here we
will provide a more comprehensive discussion about acausal modeling.

Composability
~~~~~~~~~~~~~

There are two very big advantages to acausal modeling.  The first is
composability.  In this context, composability means the ability to
drag, drop and connect component instances in nearly any configuration
we wish without having to concern ourselves with "compatibility".
This is because acausal connectors are designed around the idea of
physical compatibility, not causal compatibility.  This is possible
because acausal connector definitions focus on physical information
exchanged, not the direction that information flows. The result is
that we can create component models around the idea of physical
interactions without requiring any *a priori* knowledge about the
nature (*i.e.,* directionality) of the information exchange.

But there are other implications to this composability.  Not only can
we easily create systems by dragging, dropping and connecting
components, but we can also easily reconfigure them.  Replacing a
voltage source in an electrical circuit with a current source can have
a profound impact on the mathematical representation of that system
(*e.g.,* if the system is represented as a block diagram).  But such a
change has no significant impact when using an acausal approach.
Although the underlying mathematical representation still changes,
sometimes profoundly, there is no impact on the user, because that
representation is generated automatically as part of the compilation
process.

.. todo:: I really need to include one of the typical examples here.

Finally, another aspect of composability is in the support for
multi-domain systems.  In fact, Modelica not only supports different
engineering domains (electrical, thermal, hydraulic), it supports
multiple modeling formalisms.  Model developers have created libraries
for block diagrams, state charts, petri nets, *etc.* Instead of
requiring special tools or editors in each case, all of these
different domains and formalisms can be freely combined in Modelica as
appropriate.

.. _flow-signs:

Accounting
~~~~~~~~~~

.. index:: connect; flow

Connectors
++++++++++

The other advantage of acausal modeling is the amount of automatic
"accounting" performed with this approach.  To understand exactly what
accounting is performed, let's consider the following rotational
``connector`` definitions from the Modelica Standard Library:

.. code-block:: modelica

    connector Flange_a "1-dim. rotational flange of a shaft (filled square icon)"
      Modelica.SIunits.Angle phi "Absolute rotation angle of flange";
      flow Modelica.SIunits.Torque tau "Cut torque in the flange";
      annotation(Icon(/* Filled gray circle */));
    end Flange_a;

    connector Flange_b "1-dim. rotational flange of a shaft (filled square icon)"
      Modelica.SIunits.Angle phi "Absolute rotation angle of flange";
      flow Modelica.SIunits.Torque tau "Cut torque in the flange";
      annotation(Icon(/* Gray circular outline */));
    end Flange_b;

As we've discussed previously, an acausal connector includes two
different types of variables, across variables and through variables.
The through variable is indicated by the presence of the ``flow``
qualifier.  In the case of the ``Rotational`` connector, the across
variable is ``phi``, the angular position, and the through variable is
``tau``, the torque.

Sign Conventions
++++++++++++++++

Also recall from our previous discussion that Modelica models should
observe the following convention: a positive value for the ``flow``
variable on a connector represents the flow of that quantity **into**
the component that the connector is connected to.  This is an
important sign convention not only because it make sure all the
accounting is correct, but it also helps with composability as well by
allowing (inherently symmetric) components like springs, dampers,
*etc.* to be flipped over and still function identically.

.. index:: connection set

.. _connection-sets:

Connection Sets
+++++++++++++++

Before we can get into the details of the accounting performed by the
compiler, we need to introduce the concept of a *connection set*.  To
demonstrate what a connection set is, consider the following
schematic:

.. image:: /ModelicaByExample/Components/Rotational/Examples/SMD.*
   :width: 100%
   :align: center
   :alt:

Note that there are 8 connections in this model:

.. code-block:: modelica

    equation
      connect(ground.flange_a, damper2.flange_b);
      connect(ground.flange_a, spring2.flange_b);
      connect(damper2.flange_a, inertia2.flange_b);
      connect(spring2.flange_a, inertia2.flange_b);
      connect(inertia2.flange_a, damper1.flange_b);
      connect(inertia2.flange_a, spring1.flange_b);
      connect(damper1.flange_a, inertia1.flange_b);
      connect(spring1.flange_a, inertia1.flange_b);

If two connect statements have one connector in common, **they belong
to the same connection set**.  If a connector is not connected to any
other connectors, then it belongs to a connection set that includes
only itself.  Using this rule, we can organize the connectors into
connection sets as follows:

  * Connection Set #1

    * ``ground.flange_a``
    * ``damper2.flange_b``
    * ``spring2.flange_b``

  * Connection Set #2

    * ``damper2.flange_a``
    * ``spring2.flange_a``
    * ``inertia2.flange_b``

  * Connection Set #3

    * ``inertia2.flange_a``
    * ``damper1.flange_b``
    * ``spring1.flange_b``

  * Connection Set #4

    * ``inertia1.flange_b``
    * ``damper1.flange_a``
    * ``spring1.flange_a``

  * Connection Set #5

    * ``inertia1.flange_a``

Note that these connection sets appear from right to left in the
diagram.  It may be useful to take the time to match the connectors in
the diagram with those listed in the connection sets to understand
what a connection set intuitively is.  Note that the ``flange_a``
connectors are filled circles whereas the ``flange_b`` ones are only
outlined.

Generated Equations
+++++++++++++++++++

This is where the "accounting" starts.  For each connection **set**,
special equations are automatically generated.  The first set of
automatic equations are related to the across variables.  We need to
impose the constraint, mathematically speaking, that all across
variables must have the same value.  Furthermore, we also introduce an
equation that states that the sum of all through variables in the
connection set must sum to zero.

In the case of the connection sets above, the following equations will
be automatically generated:

.. code-block:: modelica

    // Connection Set #1
    //   Equality Equations:
    ground.flange_a.phi = damper2.flange_b;
    damper2.flange_b.phi = spring2.flange_b;
    //   Conservation Equation:
    ground.flange_a.tau + damper2.flange_b.tau + spring2.flange_b.tau = 0;

    // Connection Set #2
    //   Equality Equations:
    damper2.flange_a.phi = spring2.flange_a.phi;
    spring2.flange_a.phi = inertia2.flange_b.phi;
    //   Conservation Equation:
    damper2.flange_a.tau + spring2.flange_a.tau + inertia2.flange_b.tau = 0;

    // Connection Set #3
    //   Equality Equations:
    inertia2.flange_a.phi = damper1.flange_b.phi;
    damper1.flange_b.phi = spring1.flange_b.phi;
    //   Conservation Equation:
    inertia2.flange_a.tau + damper1.flange_b.tau + spring1.flange_b.tau = 0;

    // Connection Set #4
    //   Equality Equations:
    inertia1.flange_b.phi = damper1.flange_a.phi;
    damper1.flange_a.phi = spring1.flange_a.phi;
    //   Conservation Equation:
    inertia1.flange_b.tau + damper1.flange_a.tau + spring1.flange_a.tau = 0;

    // Connection Set #5
    //   Equality Equations: NONE
    //   Conservation Equation:
    inertia1.flange_a.tau = 0;

Note that for an empty connection set (*i.e.,* Connection Set #5),
there is only one across variable in the set, so no equality equations
are generated.  The conservation equation is still generated but it
contains only one term.  So it amounts to a statement that nothing can
flow out of an unconnected connector.  This makes intuitive physical
sense as well.

What does all this mean physically?  In the case of an electrical
connection, this implies that each connection can be treated as a
"perfect short" between the connectors.  In the case of a mechanical
system, connections are treated as perfectly rigid shafts with zero
inertia.  The bottom line is that a connection means that the across
variables on each connector will be equal and that any conserved
quantity that leaves one component must enter another one.  Nothing
can get lost or stored between components.

Conservation
++++++++++++

There are two important consequences to these equations.  The first is
that the ``flow`` variable is automatically conserved.  Typical
``flow`` variables are current, torque, mass flow rate, etc.  Since
these are all the time derivative of a conserved quantity (*i.e.,*
charge, angular momentum and mass, respectively), such equations are
automatically conserving these quantities.

But something else is being implicitly conserved as well.
Specifically, **we can ensure that energy is conserved** as well.  For
all of these domains, the power flow through a connector can be
represented by the product of the through variable and either the
across variable or a derivative of the across variable.  As a result,
for each domain we can easily derive a power conservation equation
from the equations automatically generated for the connection set.
From our example above, we know that in the first connection set we
have the following equations:

.. code-block:: modelica

    ground.flange_a.phi = damper2.flange_b;
    damper2.flange_b.phi = spring2.flange_b;
    ground.flange_a.tau + damper2.flange_b.tau + spring2.flange_b.tau = 0;

If we multiply the last equation by ``der(ground.flange_a.phi)``, the
angular velocity of the ``ground.flange_a`` connector, we get:

.. code-block:: modelica

    der(ground.flange_a.phi)*ground.flange_a.tau
    + der(ground.flange_a.phi)*damper2.flange_b.tau
    + der(ground.flange_a.phi)*spring2.flange_b.tau = 0;

However, we also know that all the across variables in the connection
set are equal.  As a result, their derivatives must also be equal.
This means that we can substitute any one of them for another.  Making
two such substitutions yields:

.. code-block:: modelica

    der(ground.flange_a.phi)*ground.flange_a.tau
    + der(damper2.flange_b.phi)*damper2.flange_b.tau
    + der(spring2.flange_b.phi)*spring2.flange_b.tau = 0;

The first term in the equation above is the power flowing into the
``ground`` component through ``flange_a``.  The second term is the
power flowing into the ``damper2`` component through ``flange_b``.
The last term is the power flowing into the ``spring2`` component
through ``flange_b``.  Since these represent the power flowing through
all connectors in the connection set, this implies that power is
conserved by that connection set (*i.e.,* all power that flows out of
one component must flow into another, nothing is lost or stored).

.. _balanced-components:

Balanced Components
+++++++++++++++++++

.. index:: balanced models
.. index:: models; balanced
.. index:: models; equations; number

If we look carefully at the previous discussion on equations generated
involving acausal variables in connection sets, we'll see something
very interesting.  But to see it, we first need to review a few things
we've learned about connectors and connector sets:

  1. A connection can only belong to one connection set.
  2. As we learned in our previous discussion on :ref:`acausal-vars`,
     for every through variable in a connector (*i.e.,* a variable
     declared with the ``flow`` qualifier), there must be a matching
     across variable (*i.e.,* a variable without any qualifier).
  3. The number of equations generated in a connection set is equal to
     the number of connectors in the connection set multiplied by the
     number of through-across pairs in the connector.

Remember that acausal variables come in pairs.  Equations for half of
those variables (one per pair) will be generated automatically via
connections.  That means the remaining half of the equations must come
from the component models themselves.

Keep in mind that this discussion is focused only on acausal variables
in connectors.  We also need to take into account two other cases:

  1. Variables declared within a component model (as opposed to on a
     connector).
  2. Causal variables on connectors (*i.e.,* those qualified by either
     ``input`` or ``output``).

Modelica requires that any non-``partial`` model be balanced.  But
what does that mean?  It means that the component should provide the
proper number of equations (no more and no less than necessary).  The question is how to compute the number of equations
required?

We already have a start based on our discussion about acausal
variables.  Since half of the equations needed for acausal variables
come from generated equations, the other half must come from within
component models containing these connectors.  Specifically, the
component must provide one equation for every through-across pair in
each of its connectors.  In addition, it should also provide one
equation for every variable on its connectors that has the ``output``
qualifier (note, the component does not have to provide equations for
any variables on its connectors with the ``input`` qualifier).  The
rationale here is that a component can assume that all ``input``
signals are known (specified externally) and that it is responsible
for computing any ``output`` signals it advertises.  Finally, any
(non-``parameter``) variable declared within the component must also
have an equation.

In summary, the number of equations that a component must provide is
the sum of:

  1. The number of through-across pairs across all its connectors
  2. The number of non-``parameter`` variables declared in the
     component model.
  3. The number of ``output`` variables across all its connectors.

Note that these equations can (and frequently do) originate in a
``partial`` model that is inherited.

If the number of equations provided by a component equals the number
of equations required, then the component model is said to be
**balanced**.

Component Definitions
^^^^^^^^^^^^^^^^^^^^^

In this chapter we've discussed how to create component models.
Fundamentally, nothing has changed since we first discussed what a
:ref:`model-definition` should include.  But it is worth emphasizing a
few things about component models.

Blocks
~~~~~~

.. index:: block

First, in the discussion on :ref:`block-components` we introduced the
idea of a ``block``.  A ``block`` is a special kind of ``model`` where
the connectors contain only ``input`` and ``output`` signals.

Conditional Variables/Connectors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another thing we saw in our discussion of the
:ref:`optional-ground-connector` was the ability to make a declaration
conditional.  The expression on which the conditional declaration
depends cannot change as a function of time (*i.e.,* the variable
cannot appear and disappear during the simulation).  Instead, it must
be a function of parameters and constants so that the compiler or
simulation runtime can determine whether the variable should be
present prior to simulation.  As we saw, the syntax for such a
declaration is:

.. code-block:: modelica

    VariableType variableName(/* modifications /*) if conditional_expression;

In other words, by including the ``if`` keyword and a conditional
expression immediately after the name of the variable (and any
modifications that are applied to the variable), we can make the
declaration of that variable conditional.  When the conditional
expression is ``true``, the conditional variable will be present.
When it is ``false``, it will not be present.

Model Limitations
~~~~~~~~~~~~~~~~~

.. _assertions:

``assert``
++++++++++

.. index:: ! assert

To understand how to enforce model limitations, we must first explain
the ``assert`` function.  The syntax of a call to the ``assert``
function is:

.. code-block:: modelica

    assert(conditional_expression, "Explanation of failure", assertLevel);

where ``conditional_expression`` is an expression that yields either
``true`` or ``false``.  A value of ``false`` indicates a failure of
the assertion.  We'll discuss the consequences of that momentarily.
The second argument must be a ``String`` that describes the reason
that the assertion failed.  The last argument, ``assertLevel``, is of
type ``AssertionLevel`` (which was defined in our previous discussion
on ``enumerations``).  This last argument is **optional** and has the
default value of ``AssertionalLevel.error``.

Now that we know how to use the ``assert`` function, let's examine the
consequences of assertions during simulation to understand why they
are important.

Defining Model Limitations
++++++++++++++++++++++++++

.. index:: model limitations
.. index:: assert
.. index:: assertions

When creating a component ``model`` (or any ``model``, for that
matter), it is useful to incorporate any limitations on the equations
in a model by including them directly in the model.  This is done by
adding ``assert`` calls in either the ``equation`` or ``algorithm``
section.  As their name implies, these assertions assert that certain
conditions must always be true.

If the equations within a model are only accurate or applicable under
certain conditions, it is essential that these conditions be included
in the model via assertions.  Otherwise, the model may silently yield
an incorrect solution.  If not uncovered, this could lead to bad
decisions based on model solutions.  If it is uncovered, it will
undermine the trust people have in the models.  So always try to
capture such model limitations.

.. index:: candidate solutions

It is worth taking a moment to understand what impact such an
assertion has during simulation.  Part of the simulation process is
the generation of so-called *candidate solutions*.  These solutions
may, or may not, end up being actual solutions.  They are usually
generated as the underlying solvers propose solutions and then check
to make sure that the solutions are accurate to within some numerical
tolerance.  Those candidate solutions that are found to be inaccurate
are typically refined in some way until a sufficiently accurate
solution is found.

If a candidate solution violates an assertion, then it is
automatically considered to be inaccurate.  The violated assertion
will automatically trigger the refinement process in an attempt to
find a solution that is more accurate and, hopefully, doesn't violate
the assertion.  However, if these refinement processes lead to a
solution that is sufficiently accurate (*i.e.,* satisfies the accuracy
requirements to within the acceptable tolerance), but that solution
still violates any assertions in the system, then the simulation
environment will do one of two things.  If the ``level`` argument in
the ``assert`` call is ``AssertionLevel.error`` then the simulation is
terminated.  If, on the other hand, the ``level`` argument is
``AssertionLevel.warning``, then the assertion description will be
used to generate a warning message to the user.  How this message is
delivered is specific to each simulation environment.  Recall that the
default value for the ``level`` argument (if none is provided in the
call to ``assert``) is ``AssertionLevel.error``.
