# TreeTest

_Note: at this stage, this is an alpha, proof-of-concept level code!_

## History / Explanation

The idea of TreeTest grew out from the pain of a codebase with too many
integrated tests - it took forever to run them all. One of the reasons
for the slownes was that the same scenarios were repeated across multiple
tests, but in such a way that it's impossible to structure them into
`setUp(Class)` methods. 

Consider the following tests:

    def test_logging_on_with_a_bad_password_gives_a_warning(self):
        self.as_admin().create_user(username = 'jane doe', password = 'good password', can_login = True)
        self.as_anonymous().login(username = 'jane doe', password = 'not matching password').assert_warning_is_present('Username or password doesn't match')

    def test_entering_a_bad_password_twice_in_a_row_still_gives_a_warning(self):
        ... snip ...

    def test_entering_a_bad_password_three_times_in_a_row_still_gives_a_warning(self):
        ... snip ...

    def test_after_three_bad_login_attempts_cannot_log_in_with_good_password__because_account_is_locked(self):
        self.as_admin().create_user(username = 'jane doe', password = '1234', can_login = True)

        self.as_anonymous().login(username = 'jane doe', password = 'not matching password')
        self.as_anonymous().login(username = 'jane doe', password = 'not matching password')
        self.as_anonymous().login(username = 'jane doe', password = 'not matching password')

        self.as_anonymous().login(username = 'jane doe', password = 'good password')

        self.as_admin().assert_account_is_locked(username = 'jane doe')


Each consecutive test is essentially the previous test followed by some
new step and an assertion, which is inefficient - but none of this 
duplication can be eliminated into `setUp(Class)` withot hurting the
readability of the tests (not to mention the deep inheritance trees
it could lead to). While there are ways to persist the state and
then later reload that, that's another test code organization headache
(as if that was easy to begin with), and of course, test runners
intentionally (??) have no guaranteed test order execution (though it's
deterministic AFAIK).

> TODO: ask @mfeathers the name of that state (de)serializer concept

And this is repetition inside a single testcase - it's quite unlikely we
would run into the bad password login scenarios in other testcase -
but likely we'll see the 
``as_admin().create_user(`` / ``as_anonymous().login()`` logic repeated
over and over again, with the same parameters.

Wouldn't it be great if all this code wasn't executed repeatedly?

### Visualizing the test scenarios and assertions

First, let's recap how automated tests work:

1. we start with an empty world 
1. the world is put it into a known state by invoking methods in the 
   system under test (_Arrange_)
1. some action, that is to be tested, is executed (_Act_)
1. assertions are made to confirm the above action put the world into
   the expected state. (_Assert_)
1. optionally, the world is put back into the empty state

If we would represent this as a graph, where 

* each node is a state of the world reached during test execution
* each edge is a method invoication with the given parameters
* each node's values are the assertions made against that state of the world

Since we always start with an empty world, this graph is a tree.

> TODO: visualization - a tree with each node (step) having little 
> green (red) dots (assertion results) around it

### Pairing tests to this graph

After having built such a tree from our tests, for each test we can 
find the last state node that test execution has reached as well as
can connect it to each of the assertions it has made.

### Speeding up execution

Given the above, a test runner could be written that - since it doesn't
have concern itself with organizing tests in a way that makes sense to
humans - could rewrite tests into an optimized execution order as well
as it could take and reload world-state snapshots as needed.

The concrete speedup of course would depend on the content of the tests,
but even without using snapshots we would need to execute at most as 
many scenarios as we have leaf nodes on the tree.

So, the example used above would become the equivalent of the below
sequence of (fluent) code:

```python
session('admin'
    ).as_admin().create_user(
            username = 'jane doe', password = '1234', can_login = True
).session('user'
    ).as_anonymous().login(
        username = 'jane doe', password = 'not matching password'
    ).assert_warning_is_present(
        'Invalid username or password'
    ).as_anonymous().login(
        username = 'jane doe', password = 'not matching password'
    ).assert_warning_is_present(
        'Invalid username or password'
    ).as_anonymous().login(
        username = 'jane doe', password = 'not matching password'
    ).assert_warning_is_present(
        'Invalid username or password'
).switch_to_session('admin'
    ).assert_account_is_locked(username = 'jane doe')
```



## Assumptions, constraints, and questions

The above model has been thought tested with tests using a fluent 
internal DSL of the application, though that might not be a requirement
- but they certainly make more readable tests that even non-technical 
project sponsors and stakeholders can understand.

## Additional benefits (taking it further)

## Visualization

Looking at such a tree graph effectively conveys a lot of metadata:

* which parts of the applications are under-tested (little-to-no dots)
* are the assertions evenly distributed?
* are the assertions heavy paths (developer interests) aligned with
  the paths important for the stakeholders?
* integrating the build pass/fail history into the nodes, problem spots
  become easy to find (though teams tend to know their codebase's 
  dragonlairs pretty well without any tool's help)


## Missing testcase discovery

Automated tests provide a safety net against regression bugs, but if 
you forgot to write test(s) for some scenario(s), and it wasn't caught
during pair programming or code review, it won't be discovered until
that 3AM call (why do these bugs never come up durig office hours?!).

However, when writing new tests, based on the tree we could suggest 
to you missing scenarios - clippy for testing or book recommendations,
e.g.: "tests units (testcases, modules, etc.) that traverse similar 
paths to your test also tends to visit these additional nodes too". And
of course, the same is true in the reverse - when writign a new test
that verifies a new variation of an existing feature, we could list
the other places that should assert against this new feature. Of 
course, this wouldn't be 100% accurate, but might be more than what we
already have.

## Designers dream

I really dread when our (wonderful!) designer comes over and asks me
for all possible examples for a given template...
