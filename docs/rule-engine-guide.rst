Rule Engine Guide
=================

.. _rule-engine-guide:

Introduction
------------

In industry, people often use `business rule engine <https://en.wikipedia.org/wiki/Business_rules_engine>`_ to maintain their business logics and knowledges. Rule engine can detect sepicific conditions and contradiction and execute predefined rules to fix them. Some examples of rule engines are `Jess <https://www.jessrules.com>`_ and `Drools <https://www.drools.org/>`_.

Despite the wide adoption, it is hard to run the rule engine in a publicly verifiable way (which means to run it on Blockchain). To eliminate the gap between traditional business rule engine and Blockchain technology, we decide to implement part of traditional rule engine features (to make it part of Lity). Rule engine related codes are directly compiled into EVM byte codes, so it can also be executed on Ethereum public chain.

Our rule engine's syntax and semantics are directly borrowed from Drools, so it might be a good idea to look through `Drools documentation <https://www.drools.org/learn/documentation.html>`_ before you use Lity's rule engine.

Rule Engine Components
----------------------

Facts
"""""
A struct that can be pattern matched in rule definitions. A fact must be a struct stored in storage.

Rules
"""""
An rule statement consists of three parts:

1. Rule Name: a string literal which served as the identifier of rule.
2. Filter Statements (a.k.a. *when block*): one or more statements describe which set of facts should be captured and applied actions in the *then block*.
3. Action Statements (a.k.a. *then block*): one or more statements to execute on matched objects (which are captured in *when block*.)

A contract with a rule definition looks like this:

.. code:: ts

    contract C {
        rule "ruleName" when {
            // Filter Statement
        } then {
            // Action Statement
        }
    }

Working Memory
""""""""""""""
Working memory is a containenr hides behind a contract, it stores facts that are ready to be pattern matched. To insert/remove facts to/from working memory, we can use ``factInsert`` and ``factDelete`` operators.

Rule Engine Operators
"""""""""""""""""""""

We have three operators to handle facts and working memory:

1. factInsert: add current object as a new fact to working memory.
2. factDelete: remove current object from the working memory.
3. fireAllRules: apply all rules on all facts in working memory.

factInsert
~~~~~~~~~~

This operator takes a struct with storage data location, evaluates to fact handle, which has type ``uint256``. Insert the reference to the storage struct into working memory.

For example:

.. code-block:: ts

   contract C {
     struct fact { int x; }
     fact[] facts;
     constructor() public {
        facts.push(fact(0));
        factInsert facts[facts.length-1]; // insert the fact into working memory
     }
   }

And note that the following statement won't compile:

.. code-block:: ts

   factInsert fact(0);

The reason is that ``fact(0)`` is a reference with memory data location, which is not persistant thus cannot be inserted into working memory.

For more information about data location mechanism, please refer to `solidity's documentation <https://solidity.readthedocs.io/en/v0.4.25/types.html#data-location>`_

factDelete
~~~~~~~~~~

This operator takes a fact handle (uint256) and evaluates to void. Removes the reference of the fact from working memory.

fireAllRules
~~~~~~~~~~~~

``fireAllRules`` is a special statement that launches lity rule engine execution, it works like drools' ``ksession.fireAllRules()`` API.


Rule Examples
-------------

Pay Pension
"""""""""""

Let's start with a simple example, which pays ether to old people.

.. code:: ts

   rule "payPension" when {
     p: Person(age >= 65, eligible == true);
     b: Budget(amount >= 10);
   } then {
     p.addr.transfer(10);
     p.eligible = false;
     b.amount -= 10;
   }

Above is a rule definition example which pay money to old people if the budget is still enough.
The rule name, ``"payPension"`` is the identifier of the rule declaration, and it should not have name collision with other identifiers.
``Person(age >= 65, eligible == true)`` means we want to match a person who is at least 65 years old and is eligible for receiving the pension. The ``p:`` syntax means to bind the matched person to identifier ``p``, so we can manipulate the person in then-block.

If the rule engine successfully find a person and a budget satisfies above requirements, the code in the second part will be executed, and we should modify the eligiblity of the person to prevent rule engine fire the same rule for the same person again.

.. code:: ts

    contract AgePension {
        struct Person {
            int age;
            bool eligible;
            address addr;
        }

        struct Budget {
            int amount;
        }

        mapping (address => uint256) addr2idx;
        Person[] ps;
        Budget budget;

        constructor () public {
            factInsert budget;
            budget.amount = 100;
        }

        function addPerson(int age) public {
            ps.push(Person(age, true, msg.sender));
            addr2idx[msg.sender] = factInsert ps[ps.length-1];
        }

        function deletePerson() public {
            factDelete addr2idx[msg.sender];
        }

        function () public payable { }
    }

Whenever a user add themselves to ``AgePension`` with ``addPerson``,
they are also recorded to the set of fact by ``factInsert``.
The operator ``factInsert`` will return an ``uint256`` as the storage location
for the fact in the working memory.
The user will be able to remove themselves from ``AgePension`` with
``deletePerson`` by passing the storage location to ``factDelete`` operator.

.. code:: ts

   rule "payPension" when {
     p: Person(age >= 65, eligible == true);
     b: Budget(amount >= 10);
   } then {
     p.addr.transfer(10);
     p.eligible = false;
     b.amount -= 10;
   }

Next, we add a ``rule "payPension"`` that gives everyone more than age 65
one ether if they haven't received age pension yet.

.. code:: ts

    contract AgePension {
        function pay() public {
            fireAllRules;
        }
    }

The age pension is paid when ``fireAllRules`` is executed.

Complete contract source:

.. code:: ts

    contract AgePension {
        struct Person {
            int age;
            bool eligible;
            address addr;
        }

        struct Budget {
            int amount;
        }

        mapping (address => uint256) addr2idx;
        Person[] ps;
        Budget budget;

        constructor () public {
            factInsert budget;
            budget.amount = 100;
        }

        function addPerson(int age) public {
            ps.push(Person(age, true, msg.sender));
            addr2idx[msg.sender] = factInsert ps[ps.length-1];
        }

        function deletePerson() public {
            factDelete addr2idx[msg.sender];
        }

        function pay() public {
            fireAllRules;
        }

        function () public payable { }
    }

Fibonacci numbers
"""""""""""""""""

Here we demostrate how to use rule engine to calculate fibonacci numbers.

First, we define a struct to represent a fibonacci number:

.. code:: ts

  struct E {
      int256 index;
      int256 value;
  }


The struct has two members. ``index`` records the index of this fibonacci number, and ``value`` records the value of the fibonacci number. If the ``value`` is unknown, we set it to ``-1``.

We can now define a rule representing fibonacci number's recurrence relation: :math:`f_n = f_{n-1} + f_{n-2}`.

.. code:: ts

    rule "buildFibonacci" when {
        x1: E(value != -1, i1: index);
        x2: E(value != -1, index == i1+1, i2: index);
        x3: E(value == -1, index == i2+1);
    } then {
        x3.value = x1.value+x2.value;
        update x3;
    }

Note that the ``update x3;`` inside rule's RHS is essential; the update statement informs the rule engine that the value of ``x3`` has been updated, and all future rule match should not depend on the old value of it.

Let's insert initial terms and unknown fibonacci numbers into working memory

.. code:: ts

   // es is a storage array storing `E`
   es.push(E(0, 0));
   factInsert es[es.length - 1];
   es.push(E(1, 1));
   factInsert es[es.length - 1];
   for (int i = 2 ; i < 10 ; i++) {
       es.push(E(i, -1));
       factInsert es[es.length - 1];
   }

Working memory now contains :math:`f_0`, :math:`f_1`, ... , :math:`f_{10}`. And only :math:`f_0` and :math:`f_1`'s value are known. We can now use ``fireAllRules`` statement to start the rule engine, and all fibonacci numbers should be calculated accordingly.

Complete source of the contract:

.. code:: ts

  contract C {
      struct E {
          int256 index;
          int256 value;
      }

      rule "buildFibonacci" when {
          x1: E(value != -1);
          x2: E(value != -1, index == x1.index+1);
          x3: E(value == -1, index == x2.index+1);
      } then {
          x3.value = x1.value+x2.value;
          update x3;
      }

      E[] es;

      constructor() public {
          es.push(E(0, 0));
          factInsert es[es.length - 1];
          es.push(E(1, 1));
          factInsert es[es.length - 1];
          for (int i = 2 ; i < 10 ; i++) {
              es.push(E(i, -1));
              factInsert es[es.length - 1];
          }
      }

      function calc() public returns (bool) {
          fireAllRules;
          return true;
      }

      function get(uint256 x) public view returns (int256) {
          return es[x].value;
      }

      function () public payable { }
  }


Cats
""""

A cat is walking on a number line. Initially it is so hungry that it can't even move.
Fortunately, there are some cat foods scattered on the number line. And each cat food can provide some energy to the cat.
Whenever the cat's location equal to cat food's location, the cat will immediately eat all the cat foods on that location and gain energy to move forward.

First, we define our fact types:

.. code:: ts

    struct Cat {
        uint256 id;
        uint256 energy;
    }
    struct CatLocation {
        uint256 id;
        uint256 value;
    }
    struct Food {
        uint256 location;
        uint256 energy;
        bool eaten;
    }

Here we model the problem in a way similiar to entity-relationship model. ``Cat`` and ``CatLocation`` has an one-to-one relationship.
Food represents a cat food on the number line, ``location`` represents its location, ``energy`` represents how much energy it can provide to Cat. Each unit of energy provides power for the cat to move one unit forward.

Now we can define 2 rules to solve the problem (Note that the order of definition is important!)

.. code:: ts

    rule "catEatFood"
    when {
        c1: Cat();
        cl1: CatLocation(id == c1.id);
        f1: Food(location == cl1.value, !eaten);
    } then {
        c1.energy += f1.energy;
        update c1;
        f1.eaten = true;
        update f1;
    }

In the above rule, we first match ``Cat`` and ``CatLocation`` using ``id``, then match all not yet eaten food that have the same location.
If we successfully found a cat whose location equal to the food's location, we let the cat eat the food and tell rule engine that ``c1`` and ``f1``'s value have been modified, so that no food will be eaten twice, for example.

The second rule:

.. code:: ts

    rule "catMoves"
    when {
        c1: Cat(energy > 0);
        cl1: CatLocation(id == c1.id);
    } then {
        c1.energy--;
        update c1;
        cl1.value++;
        update cl1;
    }

This rule states that if the cat have positive energy, it can move one unit forward.

The order of rules is important because we want the cat eat the food whenever its location overlaps with food's location. If the order is reversed, the cat will keep moving forward and ignore the food, which is not what we want.


Complete source code of the contract:

.. code:: ts

    contract C {
        struct Cat {
            uint256 id;
            uint256 energy;
        }
        struct CatLocation {
            uint256 id;
            uint256 value;
        }
        struct Food {
            uint256 location;
            uint256 energy;
            bool eaten;
        }

        // Note that rules appear first have higher priority,
        // so cats won't go through a food without eating it.
        rule "catEatFood"
        when {
            c1: Cat();
            cl1: CatLocation(id == c1.id);
            f1: Food(location == cl1.value, !eaten);
        } then {
            c1.energy += f1.energy;
            update c1;
            f1.eaten = true;
            update f1;
        }

        rule "catMoves"
        when {
            c1: Cat(energy > 0);
            cl1: CatLocation(id == c1.id);
        } then {
            c1.energy--;
            update c1;
            cl1.value++;
            update cl1;
        }

        Cat[] cats;
        CatLocation[] catLocations;
        uint256[] factIDs;
        Food[] foods;

        function addCat(uint256 initialLocation) public returns (bool) {
            uint256 newId = cats.length;
            cats.push(Cat(newId, 0));
            catLocations.push(CatLocation(newId, initialLocation));
            factIDs.push(factInsert cats[newId]);
            factIDs.push(factInsert catLocations[newId]);
            return true;
        }

        function addFood(uint256 location, uint256 energy) public returns (bool) {
            foods.push(Food(location, energy, false));
            factIDs.push(factInsert foods[foods.length-1]);
            return true;
        }

        function queryCatCoord(uint256 catId) public view returns (uint256) {
            assert(catLocations[catId].id == catId);
            return catLocations[catId].value;
        }

        function run() public returns (bool) {
            fireAllRules;
            return true;
        }

        function reset() public returns (bool) {
            for (uint256 i = 0; i < factIDs.length; i++)
                factDelete factIDs[i];
            delete cats;
            delete catLocations;
            delete factIDs;
            return true;
        }

        function () public payable { }
    }

Specs
-----

Grammar
"""""""

Grammar of current rule definition:

.. code-block:: abnf
   :linenos:

   Rule = 'rule' StringLiteral 'when' '{' RuleLHS '}' 'then' '{' RuleRHS '}'
   RuleLHS = ( ( Identifier ':' )? FactMatchExpr ';' )
   FactMatchExpr = Identifier '(' ( FieldExpr (',' FieldExpr)* )? ')'
   FieldExpr = ( Identifier ':' Identifier ) | Expression
   RuleRHS = Statement*

Note that some nonterminal symbols are defined in solidity's grammar, including ``StringLiteral``, ``Identifier``, ``Expression``, and ``Statement``.

Rete Network Generation
"""""""""""""""""""""""

* Each ``FieldExpr`` involve more than 1 facts creates a beta node. Otherwise, it creates an alpha node.
* Each nodes corresponding to a dynamic memory array (a data structure which supports lity rule engine runtime execution), these dynamic memory array contains matched fact sets of each node.
* All dynamic memory arrays are reevaluated when ``fireAllRules`` is called.

