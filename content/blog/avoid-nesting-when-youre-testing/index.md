---
slug: 'avoid-nesting-when-youre-testing'
title: `Avoid Nesting when you're Testing`
date: '2019-07-29'
author: 'Kent C. Dodds'
description:
  '_Why using hooks like beforeEach as a mechanism for code reuse leads to
  unmaintainable tests and how to avoid it._'
categories:
  - 'testing'
keywords:
  - 'testing'
  - 'javascript'
  - 'jest'
  - 'mocha'
  - 'jasmine'
  - 'ava'
banner: './images/banner.jpg'
bannerCredit: 'Photo by [Kate Remmer](https://unsplash.com/photos/05_sUnshoaE)'
---

I want to show you something. What I'm going to show is a general testing
principle, applied to a React component test. So even though the example is a
React one, hopefully it helps communicate the concept properly.

Here's a React component that I want to test:

```jsx
// login.js
import React from 'react'

function Login({onSubmit}) {
  const [error, setError] = React.useState('')

  function handleSubmit(event) {
    event.preventDefault()
    const {
      usernameInput: {value: username},
      passwordInput: {value: password},
    } = event.target.elements

    if (!username) {
      setError('username is required')
    } else if (!password) {
      setError('password is required')
    } else {
      setError('')
      onSubmit({username, password})
    }
  }

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <div>
          <label htmlFor="usernameInput">Username</label>
          <input id="usernameInput" />
        </div>
        <div>
          <label htmlFor="passwordInput">Password</label>
          <input id="passwordInput" type="password" />
        </div>
        <button type="submit">Submit</button>
      </form>
      {error ? <div role="alert">{error}</div> : null}
    </div>
  )
}

export default Login
```

And here's what that renders (it actually works, try it):

import {Login} from './components'

<div
  style={{
    padding: 14,
    backgroundColor: 'rgba(0,0,0,0.05)',
    borderRadius: 4,
    marginBottom: 20,
  }}
>
  <Login
    onSubmit={({username, password}) =>
      alert(`Username is "${username}" and Password is "${password}"`)
    }
  />
</div>

Here's a test suite that resembles the kind of testing I've seen over the years.

```jsx
// __tests__/login.js
import React from 'react'
import {render, fireEvent} from '@testing-library/react'
import Login from '../login'

describe('Login', () => {
  let utils,
    handleSubmit,
    user,
    changeUsernameInput,
    changePasswordInput,
    clickSubmit

  beforeEach(() => {
    handleSubmit = jest.fn()
    user = {username: 'michelle', password: 'smith'}
    utils = render(<Login onSubmit={handleSubmit} />)
    changeUsernameInput = value =>
      fireEvent.change(utils.getByLabelText(/username/i), {target: {value}})
    changePasswordInput = value =>
      fireEvent.change(utils.getByLabelText(/password/i), {target: {value}})
    clickSubmit = () => fireEvent.click(utils.getByText(/submit/i))
  })

  describe('when username and password is provided', () => {
    beforeEach(() => {
      changeUsernameInput(user.username)
      changePasswordInput(user.password)
    })

    describe('when the submit button is clicked', () => {
      beforeEach(() => {
        clickSubmit()
      })

      it('should call onSubmit with the username and password', () => {
        expect(handleSubmit).toHaveBeenCalledTimes(1)
        expect(handleSubmit).toHaveBeenCalledWith(user)
      })
    })
  })

  describe('when the password is not provided', () => {
    beforeEach(() => {
      changeUsernameInput(user.username)
    })

    describe('when the submit button is clicked', () => {
      let errorMessage
      beforeEach(() => {
        clickSubmit()
        errorMessage = utils.getByRole('alert')
      })

      it('should show an error message', () => {
        expect(errorMessage).toHaveTextContent(/password is required/i)
      })
    })
  })

  describe('when the username is not provided', () => {
    beforeEach(() => {
      changePasswordInput(user.password)
    })

    describe('when the submit button is clicked', () => {
      let errorMessage
      beforeEach(() => {
        clickSubmit()
        errorMessage = utils.getByRole('alert')
      })

      it('should show an error message', () => {
        expect(errorMessage).toHaveTextContent(/username is required/i)
      })
    })
  })
})
```

That should give us 100% confidence that this component works and will continue
to work as designed. And it does. But here are the things I don't like about
that test:

## Over-abstraction

I feel like the utilities like `changeUsernameInput` and `clickSubmit` can be
nice, but the tests are simple enough that duplicating that code instead could
simplify our test code a bit. It's just that the abstraction of the function
doesn't really give us a whole lot of benefit for this small set of tests, and
we incur the cost for maintainers to have to look around the file for where
those functions are defined.

## Nesting

The tests above are written with [Jest](https://jestjs.io/) APIs, but you'll
find similar APIs in all major JavaScript frameworks. I'm talking specifically
about `describe` which is used for grouping tests, `beforeEach` for common
setup/actions, and `it` for the actual assertions.

I have a strong dislike for nesting like this. I've written and maintained
thousands of tests that were written like this and I can tell you that as
painful as it is for these three simple tests, it's way worse when you have
thousands of lines of tests and wind up nesting even further.

What makes it so complex? Take this bit for example:

```javascript
it('should call onSubmit with the username and password', () => {
  expect(handleSubmit).toHaveBeenCalledTimes(1)
  expect(handleSubmit).toHaveBeenCalledWith(user)
})
```

Where is `handleSubmit` coming from and what's its value? Where is `user` coming
from? What's _its_ value? Oh sure, you can go find where it's defined:

```javascript {3-4}
describe('Login', () => {
  let utils,
    handleSubmit,
    user,
    changeUsernameInput,
    changePasswordInput,
    clickSubmit
  // ...
})
```

But then you _also_ have to figure out where it's assigned:

```javascript {2-3}
beforeEach(() => {
  handleSubmit = jest.fn()
  user = {username: 'michelle', password: 'smith'}
  // ...
})
```

And then, you have to make sure that it's not actually being assigned to
something else in a further nested `beforeEach`. Tracing through the code to
keep track of the variables and their values over time is the number one reason
I strongly recommend against nested tests. The more you have to hold in your
head for menial things like that, the less room there is for accomplishing the
important task at hand.

You can argue that variable reassignment is an "anti-pattern" and should be
avoided, and I would agree with you, but adding more linting rules to your suite
of possibly already overbearing linting rules is not an awesome solution. What
if there were a way to share this common setup without having to worry about
variable reassignment at all?

## Inline it!

For this simple component, I think the best solution is to just remove as much
abstraction as possible. Check this out:

```jsx
// __tests__/login.js
import React from 'react'
import {render, fireEvent} from '@testing-library/react'
import Login from '../login'

test('calls onSubmit with the username and password when submit is clicked', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText, getByText} = render(<Login onSubmit={handleSubmit} />)
  const user = {username: 'michelle', password: 'smith'}

  fireEvent.change(getByLabelText(/username/i), {
    target: {value: user.username},
  })
  fireEvent.change(getByLabelText(/password/i), {
    target: {value: user.password},
  })
  fireEvent.click(getByText(/submit/i))

  expect(handleSubmit).toHaveBeenCalledTimes(1)
  expect(handleSubmit).toHaveBeenCalledWith(user)
})

test('shows an error message when submit is clicked and no username is provided', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText, getByText, getByRole} = render(
    <Login onSubmit={handleSubmit} />,
  )

  fireEvent.change(getByLabelText(/password/i), {target: {value: 'anything'}})
  fireEvent.click(getByText(/submit/i))

  const errorMessage = getByRole('alert')
  expect(errorMessage).toHaveTextContent(/username is required/i)
  expect(handleSubmit).not.toHaveBeenCalled()
})

test('shows an error message when submit is clicked and no password is provided', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText, getByText, getByRole} = render(
    <Login onSubmit={handleSubmit} />,
  )

  fireEvent.change(getByLabelText(/username/i), {target: {value: 'anything'}})
  fireEvent.click(getByText(/submit/i))

  const errorMessage = getByRole('alert')
  expect(errorMessage).toHaveTextContent(/password is required/i)
  expect(handleSubmit).not.toHaveBeenCalled()
})
```

> Note: `test` is an alias for `it` and I just prefer using `test` when I'm not
> nested in a `describe`.

You'll notice that there is a bit of duplication there (we'll get to that), but
look at how clear these tests are. With the exception of some test utilities and
the Login component itself, the entire test is self-contained. This
significantly improves the ability for us to understand what's going on in each
test without having to do any scrolling around. If this component had a few
dozen more tests, the benefits would be even more potent.

Notice also that we aren't nesting everything in a `describe` block, because
it's really not necessary. Everything in the file is clearly testing the `login`
component, and including even a single level of nesting is pointless.

## Apply AHA (Avoid Hasty Abstractions)

The [AHA principle](/blog/aha-programming) states that you should:

> prefer duplication over the wrong abstraction and optimize for change first.

For our simple Login component here, I'd probably leave the test as-is, but
let's imagine that it's a bit more complicated and we're starting to see some
problems with code duplication and we'd like to reduce it. Should we reach for
`beforeEach` for that? I mean, that's what it's there for right?

Well, we could, but then we have to start worrying about mutable variable
assignments again and we'd like to avoid that. How else could we share code
between our tests? AHA! We could use functions!

```jsx
import React from 'react'
import {render, fireEvent} from '@testing-library/react'
import Login from '../login'

// here we have a bunch of setup functions that compose together for our test cases
// I only recommend doing this when you have a lot of tests that do the same thing.
// I'm including it here only as an example. These tests don't necessitate this
// much abstraction. Read more: https://kcd.im/aha-testing
function setup() {
  const handleSubmit = jest.fn()
  const utils = render(<Login onSubmit={handleSubmit} />)
  const user = {username: 'michelle', password: 'smith'}
  const changeUsernameInput = value =>
    fireEvent.change(utils.getByLabelText(/username/i), {target: {value}})
  const changePasswordInput = value =>
    fireEvent.change(utils.getByLabelText(/password/i), {target: {value}})
  const clickSubmit = () => fireEvent.click(utils.getByText(/submit/i))
  return {
    ...utils,
    handleSubmit,
    user,
    changeUsernameInput,
    changePasswordInput,
    clickSubmit,
  }
}

function setupSuccessCase() {
  const utils = setup()
  utils.changeUsernameInput(utils.user.username)
  utils.changePasswordInput(utils.user.password)
  utils.clickSubmit()
  return utils
}

function setupWithNoPassword() {
  const utils = setup()
  utils.changeUsernameInput(utils.user.username)
  utils.clickSubmit()
  const errorMessage = utils.getByRole('alert')
  return {...utils, errorMessage}
}

function setupWithNoUsername() {
  const utils = setup()
  utils.changePasswordInput(utils.user.password)
  utils.clickSubmit()
  const errorMessage = utils.getByRole('alert')
  return {...utils, errorMessage}
}

test('calls onSubmit with the username and password', () => {
  const {handleSubmit, user} = setupSuccessCase()
  expect(handleSubmit).toHaveBeenCalledTimes(1)
  expect(handleSubmit).toHaveBeenCalledWith(user)
})

test('shows an error message when submit is clicked and no username is provided', () => {
  const {handleSubmit, errorMessage} = setupWithNoUsername()
  expect(errorMessage).toHaveTextContent(/username is required/i)
  expect(handleSubmit).not.toHaveBeenCalled()
})

test('shows an error message when password is not provided', () => {
  const {handleSubmit, errorMessage} = setupWithNoPassword()
  expect(errorMessage).toHaveTextContent(/password is required/i)
  expect(handleSubmit).not.toHaveBeenCalled()
})
```

Now we could have dozens of tests that use these simple `setup` functions, and
notice also that they can be composed together to give us a similar behavior as
the nested `beforeEach` that we had before if that makes sense. But we avoid
having mutable variables that we have to worry about keeping track of in our
mind.

You can learn more about the benefits of AHA with testing from
[AHA Testing](/blog/aha-testing).

## What about grouping tests?

The `describe` function is intended to group related tests together and can
provide for a nice way to visually separate different tests, especially when the
test file gets big. But I don't like it when the test file gets big. So instead
of grouping tests by `describe` blocks, I group them by file. So if there's a
logical grouping of different tests for the same "unit" of code, I'll separate
them by putting them in completely different files. And if there's some code
that really needs to be shared between them, then I'll create a
`__tests__/helpers/login.js` file which has the shared code.

This comes with the benefit of logically grouping tests, completely separating
any setup that's unique for them, reducing the cognitive load of working on a
particular part of the unit of code I'm working on, and if your testing
framework can run tests in parallel, then my tests will probably run faster as
well.

## What about cleanup?

This blog post isn't an attack on utilities like `beforeEach`/`afterEach`/etc.
It's more of a caution against mutable variables in tests, and being mindful of
your abstractions.

For cleanup, sometimes you're stuck with a situation where the thing you're
testing makes some changes to the global environment and you need to cleanup
after it. If you try to put that code inline within your test, then a test
failure would result in your cleanup not running which could then lead to other
tests failing, ultimately resulting in a lot of error output that is harder to
debug.

For example, React Testing Library will insert your component into the document,
and if you don't cleanup after each test, then your tests can run over
themselves:

```jsx
import React from 'react'
import {render, fireEvent} from '@testing-library/react'
import Login from '../login'

test('example 1', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  fireEvent.change(getByLabelText(/password/i), {
    target: {value: 'little lamb'},
  })
  // more test here
})

test('example 2', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  // 💣 this will blow up because the `getByLabelText` is actually querying the
  // entire document, and because we didn't cleanup after the previous test
  // we'll get an error indicating that RTL found more than one field with the
  // label "username"
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  // more test here
})
```

Fixing this is pretty simple, you need to execute the `cleanup` method from
`@testing-library/react` after each test.

```jsx {2,13,21}
import React from 'react'
import {render, fireEvent, cleanup} from '@testing-library/react'
import Login from '../login'

test('example 1', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  fireEvent.change(getByLabelText(/password/i), {
    target: {value: 'little lamb'},
  })
  // more test here
  cleanup()
})

test('example 2', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  // more test here
  cleanup()
})
```

However, if you _don't_ use `afterEach` to do this then if a test fails your
cleanup wont run, like this:

```jsx {5-9}
test('example 1', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  // 💣 the following typo will result in a error thrown:
  //   "no field with the label matching passssword"
  fireEvent.change(getByLabelText(/passssword/i), {
    target: {value: 'little lamb'},
  })
  // more test here
  cleanup()
})
```

Because of this, the `cleanup` function in "example 1" will not run and then
"example 2" wont run properly, so instead of only seeing 1 test failure, you'll
see that all the tests failed and it'll make it much harder to debug.

So instead, you should use `afterEach` and that will ensure that even if your
test fails, you can cleanup:

```jsx
import React from 'react'
import {render, fireEvent, cleanup} from '@testing-library/react'
import Login from '../login'

afterEach(() => cleanup())

test('example 1', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  fireEvent.change(getByLabelText(/password/i), {
    target: {value: 'little lamb'},
  })
  // more test here
})

test('example 2', () => {
  const handleSubmit = jest.fn()
  const {getByLabelText} = render(<Login onSubmit={handleSubmit} />)
  fireEvent.change(getByLabelText(/username/i), {target: {value: 'mary'}})
  // more test here
})
```

> Even better, with React Testing Library, you can import
> `@testing-library/react/cleanup-after-each` and it'll do this for you
> automatically. Learn more
> [in the docs](https://testing-library.com/docs/react-testing-library/setup#cleanup)

In addition, sometimes there are definitely good use cases for `before*`, but
they're normally matched with a cleanup that's necessary in an `after*`. Like
starting and stopping a server:

```javascript
let server
beforeAll(async () => {
  server = await startServer()
})
afterAll(() => server.close())
```

There's not really any other reliable way to do this. Another use case I can
think of that I've used these hooks is for testing `console.error` calls:

```javascript
beforeAll(() => {
  jest.spyOn(console, 'error').mockImplementation(() => {})
})

afterEach(() => {
  console.error.mockClear()
})

afterAll(() => {
  console.error.mockRestore()
})
```

**So there are definitely use cases for those kinds of hooks. I just don't
recommend them as a mechanism for code reuse. We have functions for that.**

## Conclusion

I hope this helps clarify what I meant in this tweet:

https://twitter.com/kentcdodds/status/1154468901121482753

I've written tens of thousands of tests with different frameworks and styles and
in my experience, reducing the amount of variable mutation has resulted in
vastly simpler test maintenance. Good luck!

P.S. If you'd like to play with the examples,
[here's a codesandbox](https://codesandbox.io/s/react-codesandbox-ni9fk)