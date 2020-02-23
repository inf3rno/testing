# testing

[![Build Status](https://travis-ci.org/inf3rno/testing.svg?branch=master)](https://travis-ci.org/inf3rno/testing)
my testing library

# todo

- [ ] everything :P

# documentation

**usage**

We can start our tests from a module, something like this:
```js
import {Selection, FileNameFilter, ESMObjectLoader, ParallelExecutor, ConsoleReporter} from "@inf3rno/testing";

var selection = new Selection();
selection.include(new FileNameFilter("./test/**/spec.*.js"));
var loader = new ESMObjectLoader();
var plan = loader.createExecutionPlan(selection);
var executor = new ParallelExecutor();
executor.execute(plan, reporter, {dbPass: "..."});
```

An alternative way to use the library is using the execute function, which creates the upper objects automagically:

```js
import execute from "@inf3rno/testing";
execute("./test/**/spec.*.js", {dbPass: "..."});
```

We can do the same from CLI, which uses a CLI specific reporter.

```js
testing --include "./test/**/spec.*.js" --context "{dbPass: '...'}"
```

The tests look like this:

test/math/spec.mySum.js
```js
import mySum from "../math.js";
import assert from "assert";

export default {
    name: "my summary implementation",
    shouldReturnTheSumOfThePassedNumbers(context){
        assert.equal(mySum(1,2), 3);
        assert.equal(mySum(4,5,1), 10);
    }
}
```

**Selection**

We use selections to store the filters for selecting actual tests. After the selection is made we use the loaders to load all tests matching the selection. We can include and exclude tests using the selection. The selection object is mutable.

```js
var selection = new Selection();
selection.include(new FileNameFilter("./test/**/*.js"));
selection.exclude(new FileNameFilter("./test/utility/*"));
selection.exclude(new FileNameFilter("./test/helper/*"));
```

**Filter**

We use the filters to define inclusion or exclusion rules for the selection. We can filter by file name, by test or group name, by test or group object if imported into the filter and by test dependencies.

We can use glob and regex patterns or exact match:
```js
new FileNameFilter("./test/**/spec.*.js");
new FileNameFilter(/^\.\/test\/(?:|.+\/)spec\.[^\/]+\.js$/g);
new FileNameFilter("./test/spec.loader.js");
```

We can use different filter types:
```js
new FileNameFilter("./test/**/spec.*.js");
new TestFilter(/loader/);
new TestFilter(loaderTest);
new TestDependencyFilter(/loader/);
new TestDependencyFilter(loaderTest);
```

**Loader**

We use loaders to load the tests described in the selections and make an execution plan for them. The relevance of loaders is supporting different types of test definitions.

We can define the tests according to the loader we use, for example:
```js
export default function testFeature(){}
```
```js
export default new Test("test feature", function (){});
```
```js
module.exports = new Test("test feature", function (){});
```
```js
module.exports = {
    testFeature1(){},
    testFeature2(){}
}
```

We have a default loader that supports ESM modules with objects literals as test groups. It uses the file name for naming the test group and camelCase to space separated transformation for naming the tests if necessary.
```js
export default {
    testFeature1(){},
    testFeature2(){},
    "test feature 3": function (){},
    "test feature 4": async function (){}
};
```

It is possible to describe dependencies too. In this example `testFeature1` depends on `testFeature2`. Dependencies are important because they affect execution order.
```js
export default {
    testFeature1: [function (){}, "testFeature2"],
    testFeature2(){},
};
```

**Test**

The test type is for defining tests. Test definitions contains a name, a function and dependencies. The functions are just normal js functions that can throw errors by failure or do nothing by success. We can use async functions too. The tests get parameters passed to the executor.

```js
var test1 = new Test("some feature", function (context){
    assert.equal(sum(1,2), 3);
}, [test2, test3]);
```

**Group**

Test groups are collection of tests. They can have dependencies and tests can depend on entire tests groups too.

```js
var test1 = new Test("some feature", function (){});
var test2 = new Test("other feature", function (){});

var group12 = new Group("group 12", [test1, test2]);

var test3 = new Test("dependent feature, function (){}, group12);
```

**ExecutionPlan**

The execution plan is created based on the loaded tests and their dependencies. The plan will be executed by the executor.

```js
var selection = new Selection();
selection.include(new FileNameFilter("./test/**/spec.*.js"));
var loader = new ESMLoader();
var plan = loader.createExecutionPlan(selection);
```

**Executor**

The executor executes the execution plan, can pass parameters to tests and publishes the events to the reporters.

Our default executor executes the tests async parallel and uses series only by dependencies. It can pass a single context object to each of the tests, which can be used for configuration.

```js
var executor = new ParallelExecutor();
var reporter = new ConsoleReporter();
var context = {db: databaseConnection};
executor.execute(plan, reporter, context);
```

We can set reporters and context on the executor if we want to execute multiple plans.

```js
executor.subscribe(reporter);
executor.context(context);
await executor.execute(plan1);
await executor.execute(plan2, additionalReporters, additionalContext);
await executor.execute(plan3);
executor.clearContext(context);
executor.unsubscribe(reporter);
```

We can execute multiple plans parallel too if the reporter supports it.

```js
executor.subscribe(parallelReporter);
executor.context(context);
await Promise.all([
    executor.execute(plan1, plan3),
    executor.execute(plan2, additionalReporters, additionalContext),
]);
executor.clearContext(context);
executor.unsubscribe(parallelReporter);
```

**Event**

The executor creates events by execution, which can be used for reporting. Currently we have these events:

- `PlanStarted{plan}` - starting the execution of the plan, can be used the count the tests for example
- `PlanExecuted{plan, duration, failures, skips}` - the execution of the plan is done, can be used to display summary
- `PlanAborted{plan, duration, failures, skips, abortions}` - execution can be aborted depending on the executor type e.g. abortion at the first failure

- `GroupStarted{plan, group, testCount}` - starting the execution of the group
- `GroupExecuted{plan, group, duration}` - the execution of the group is done, it was a success
- `GroupFailed{plan, group, duration, reasons}` - the group failed
- `GroupSkipped{plan, group, reasons}` - the group was skipped because a dependency failed
- `GroupAborted{plan, group, reason}` - the executor aborted the test

- `TestStarted{plan, group, test}` - starting the execution of the test
- `TestExecuted{plan, group, test, duration}` - the execution of the test is done, it was a success
- `TestFailed{plan, group, test, duration, reason}` - the test failed with the following reason
- `TestSkipped{plan, group, test, reason}` - the test was skipped because a dependency failed
- `TestAborted{plan, group, test, reason}` - the executor aborted the test

**Reporter**

The reporter reports about the execution to the user of the library. There is many-many relationship between executors and reporters, so we can use multiple reporters for the same executor and a single reporter for multiple executors. The communication goes by publishing events the reporters can subscribe to.

By default we have a console reporter, which reports only the failures to the console.

```js
var reporter = new ConsoleReporter();
```

We can define a custom reporter relative easily.

```js
class MyReporter {
    handle(event){
        // handle any type of event
    }
    handlePlanStarted(planStarted){
        // handle a specific type of event
    }
    handlePlanExecuted(planExecuted){
        // handle a specific type of event
    }
    // other handlers
}
```
