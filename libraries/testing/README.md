TaskCluster-Lib-Testing
=======================

Support for testing TaskCluster components.

This module contains a number of utilities that facilitate testing TaskCluster
components.  It is typically installed as a devDependency, so it is not used in
production code.

See the source for detailed documentation.

Sticky Loader
-------------

A sticky loader is a thin wrapper around `taskcluster-lib-loader` to support
dependency injection. It "remembers" each value it has returned and will return
it again on the next call; it can also have a dependency injected.  Use it like
this in `helper.js`:

```javascript
const {stickyLoader} = require('taskcluster-lib-testing');
const load = require('../src/server');

exports.load = stickyLoader(load);

setup(function() {
  exports.load.reset();
  exports.load.inject('profile', 'test');
  exports.load.inject('process', 'test');
});
```

then, in test scripts:

```javascript
const {load} = require('./helper');

setup(async function() {
  load.inject('cfg', { /* fake config */});
  const SomeTable = await load('SomeTable');
  await SomeTable.create({ /* ... */ });
});

test(async function() {
  const component = await load('some-component');
  // component will be loaded with the fake config and with
  // the same instance of SomeTable that we set up above
});
```

Secrets
-------

This class handles getting secrets for tests, and easily determining what
secrets are available.  It integrates with `typed-env-config`.  Set it up by
in `test/helper.js`:

```javascript
const {Secrets} = require('taskcluster-lib-testing');

exports.secrets = new Secrets({
  secretName: 'project/taskcluster/testing/taskcluster-lib-foo',
  secrets: {
   pulse: [
     // env - the environment variable by which this secret is set in the config (if any)
     // cfg - dotted path to the config value containing this secret (if any)
     // name - name for the secret (used for programmatic access in tests; defaults to env)
     {env: 'PULSE_USERNAME', cfg: 'pulse.username', name: 'username'},
     {env: 'PULSE_PASSWORD', cfg: 'pulse.password', name: 'password'},
   ],
   aws: [
     {env: 'AWS_ACCESS_KEY_ID', cfg: 'aws.accessKeyId'},
     {env: 'AWS_SECRET_ACCESS_KEY', cfg: 'aws.secretAccessKey'},
   ],
  },
);
```

Then, in a global, async `setup` function, set it up (using a sticky loader):

```javascript
suiteSetup(async function() {
  const cfg = await load('cfg');
  await exports.secrets.setup({cfg});
});
```

This will fetch secrets, if necessary, and modify `cfg` in-place, if it is given.
If the system you are testing does not use `typed-env-config`, simply omit the `cfg` option, and do not specify the `cfg` properties to the constructor.

The secrets object has a few useful methods, all of which can only be called *after* `setup`:

* `secrets.have(name)` -- true if the given secret is available
* `secrets.get(name)` -- returns an object containing the secret values by name, or throws an error if not avaialble
* `secrets.mockSuite(title, [secrets], async function(mock) { .. })` -- run the given suite of tests both with and without mocks, skipping the real tests if all given secrets are not available.
  The `mock` parameter is true for the mock version, and false for the real version.
  If `$NO_SKIP_TESTS` is set, then this will throw an error for each test when secrets are not available.

If a secret is defined in the loaded configuration, that value will be used even if the `env` key is also set.
Secrets should not have any value set in `config.yml`, or this class will not function properly.

## Usage

```javascript
// helper.js
const {Secrets, stickyLoader} = require('taskcluster-lib-testing');

const secrets = new Secrets({
  secretName: 'project/taskcluster/testing/taskcluster-ping',
  secrets: {
    pingdom: [
      {name: 'apiKey', env: 'PINGDOM_API_KEY', cfg: 'app.pingdom.apiKey'},
    ],
  });
const load = stickyLoader(require('../src/main'));

suiteSetup(async function() {
  secrets.setup(load('cfg'));
});

exports.secrets = secrets;
exports.load = load;
```

```javascript
// some_test.js
const {secrets, load} = require('./helper');

secrets.mockSuite('pingdom updates', ['pingdom'], function(mock) {
  const pingdomUpdater = new PingdomUpdater({apiKey: mock ? 'pretendKey' : secrets.get('pingdom').apiKey});
  if (mock) {
    suiteSetup(function() {
      nock('https://pingdom.com:443', ..); // mock out Pingdom API
    });
    suiteTeardown(function() {
      nock.clearAll();
    });
  }

  test('updates once', function() { .. });
});
```

The test output will contain something like

```
  pingdom updates (mock)
    ✓ updates once
  pingdom updates (real)
    - updates once
```


PulseTestReceiver
-----------------

A utility for tests written in mocha, that makes it very easy to wait for a
specific pulse message.  This uses real pulse messages, so pulse credentials
will be required.

**Example:**
```js
suite("MyTests", function() {
  let credentials = {
    username:     '...',  // Pulse username
    password:     '...'   // Pulse password
  };
  let receiver = new testing.PulseTestReceiver(credentials, mocha)

  test("create task message arrives", async function() {
    var taskId = slugid.v4();

    // Start listening for a message with the above taskId, giving
    // it a local name (here, `my-create-task-message`)
    await receiver.listenFor(
      'my-create-task-message',
      queueEvents.taskCreated({taskId: taskId})
    );

    // We are now listen for a message with the taskId
    // So let's create a task with it
    await queue.createTask(taskId, {...});

    // Now we wait for the message to arrive
    let message = await receiver.waitFor('my-create-task-message');
  });
});
```

The `receiver` object will setup an PulseConnection before all tests and close
the PulseConnection after all tests. This should make tests run faster.  All
internal state, ie. the names given to `listenFor` and `waitFor` will be reset
between all tests.

schemas
-------

Test schemas with a positive and negative test cases.

The method should be called within a `suite`, as it will call the mocha `test`
function to define a test for each schema case.

 * `validator` - {}  // options to pass to the [taskcluster-lib-validate](https://github.com/taskcluster/taskcluster-lib-validate) constructor
 * `cases` - array of test cases
 * `basePath` -  base path for relative pathnames in test cases (default `path.join(__dirname, 'validate')`)
 * `schemaPrefix` - prefix used to resolve schema references; usually `http://schemas.taskcluster.net`

Each test case looks like this:

```js
{
  schema:   'svc/v7/frobnicate-foo.json', // JSON schema identifier to test against (appended to schemaPrefix)
  path:     'test-file.json',             // Path to test file (relative to basePath)
  success:  true || false                 // true if validation should succeed; false if it should fail
}
```

fakeauth
--------

A fake for the auth service to support testing APIs without requiring
production credentials, using Nock.

This object intercepts requests to the auth service's `authenticateHawk` method
and return a response based on the given `clients`, instead. Note that
accessTokens are not checked -- the fake simply controls access based on
clientId or the scopes in a temporary credential or supplied with
authorizedScopes.

To start the mock, call `testing.fakeauth.start(clients)` in your suite's
`setup` method. Clients has the form

```js
{
 "clientId1": ["scope1", "scope2"],
 "clientId2": ["scope1", "scope3"],
}
```

Call `testing.fakeauth.stop()` in your test suite's `teardown` method to stop the HTTP interceptor.

Utilities
---------

### Sleep

The `sleep` function returns a promise that resolves after a delay.

**NOTE** tests that depend on timing are notoriously unreliable, and suggest
poorly-isolated tests. Consider writing the tests to use a "fake" clock or to
poll for the expected state.

### Poll

The `poll` function will repeatedly call a function that returns a promise
until the promise is resolved without errors.