---
title: Coding's Unexpected Journeys
categories:
  - Concepts
tags:
  - Coding Practices
---

A few years back, I had the pleasure of interacting with APIs built by the three credit bureaus in the US, which was pure joy.
Following their cryptic documentation was fun, but once we navigated through that, the good times really started rolling.
Turned out, the actual payloads we received from them were often very different from their docs, which broke our integration.

This issue comes up a lot in software development; we write code that works with the "happy path", but breaks otherwise. Whether it's changing APIs, or just edge cases that were not considered.
It takes a seasoned developer to write code that is fault-tolerant and future-proof. While errors and edge cases can be the source of frustration, they also offer moments of enlightenment and professional growth.

Let's explore some examples of code and how we can think of possible issues with it, and how those can be improved.
The code examples are in TypeScript, but the concepts should be universal.

### The Mysterious Case of the API Call

The most common source of unexpected input is third party integration. Between breaking changes in API, bad documentation, or network errors, you are in for a wild ride.
Even with our beloved static type safety tools in place, those API payloads just don't care.

Consider the following function:

```typescript
const sendWelcomeEmailAndUpdateStatus = async (
  userId: string
): Promise<User> => {
  const { email } = userRepository.findOne(userId);
  await mailer.sendWelcomeEmail(email);
  await userRepository.setStatus(userId, UserStatus.onboarded);
};
```

Seems generally fine, right? You should be hearing a lot of alarm bells in your head right now.
What if:

- The request to send the email fails?
- The database update or retrieval fails?
- The `userId` is not in the database?
- The email is an empty string? Or invalid?

OK, let's address some of these concerns:

```typescript
const sendWelcomeEmailAndUpdateStatus = async (
  userId: string
): Promise<User> => {
  const user = await userRepository.findOne(userId);
  if (!user) {
    throw new SendWelcomeEmailUserNotFoundError(userId);
  }
  if (user.status != UserStatus.needsOnboarding) {
    throw new SendWelcomeEmailInvalidUserError(userId);
  }
  if (invalidEmail(user.email)) {
    throw new SendWelcomeEmailBadArgsError(user.email);
  }
  const response = await mailer.sendWelcomeEmail(user.email);
  if (!response.ok) {
    throw new EmailRequestFailedError(response);
  }
  await userRepository.setStatus(userId, UserStatus.onboarded);
};
```

That's a bit better. However, what happens if the API or repo calls error unexpectedly?
Also, this function can throw a lot of errors, which can possibly be unexpected behaviour to the consumer (since it is not explicit anywhere).

Let's adopt a pattern from rust and return errors within a [result](https://doc.rust-lang.org/std/result/) instead of throwing them.
This pattern facilitates an agreed upon contract between the function and the consumer about what the possible outcomes of the function call can be.
This pattern has been [adopted quite nicely](https://github.com/vultix/ts-results) in TypeScript as well.
Our function might look like this now:

```typescript
const sendWelcomeEmailAndUpdateStatus = async (
  userId: string
): Promise<Result<User, SendWelcomeEmailError>> => {
  try {
    const userResult = await userRepository.findOne(userId);
    if (!userResult.ok) {
      return userResult.mapErr((err) => SendWelcomeEmailError.from(err));
    }
    const user = userResult.safeUnwrap();

    if (!user) {
      return Err(SendWelcomeEmailUserNotFoundError(userId));
    }
    if (user.status != UserStatus.needsOnboarding) {
      return Err(SendWelcomeEmailInvalidUserError(userId));
    }
    if (invalidEmail(user.email)) {
      return Err(SendWelcomeEmailBadArgsError(user.email));
    }
    const response = await mailer.sendWelcomeEmail(user.email);
    if (!response.ok) {
      Err(EmailRequestFailedError(response));
    }
    const result = await userRepository.setStatus(userId, UserStatus.onboarded);
    return result.mapErr((err) => SendWelcomeEmailError.from(err));
  } catch (err) {
    return Err(SendWelcomeEmailUnexpectedError(err));
  }
};
```

On top of being explicit about what a consumer can expect from using this function, we can also quite nicely handle an unexpected error case.
The example above assumes the repository function is also returning a result, so we just need to map that error result to one expected by this function.
This is done here by using a `from` method that we assume is implemented on our `SendWelcomeEmailError` class.

We're still not quite there. There's a quite glaring issue.
What happens if the database update fails? Would the user potentially get a bunch of welcome emails?
We could solve this with good old database transactions, which might be abstracted away and look like this:

```typescript
const sendWelcomeEmailAndUpdateStatus = async (
  userId: string
): Promise<Result<User, SendWelcomeEmailError>> => {
  try {
    const userResult = await userRepository.findOne(userId);
    if (!userResult.ok) {
      return userResult.mapErr((err) => SendWelcomeEmailError.from(err));
    }
    const user = userResult.safeUnwrap();

    if (!user) {
      return Err(new SendWelcomeEmailUserNotFoundError(userId));
    }
    if (user.status != UserStatus.needsOnboarding) {
      return Err(new SendWelcomeEmailInvalidUserError(userId));
    }
    if (invalidEmail(user.email)) {
      return Err(new SendWelcomeEmailBadArgsError(user.email));
    }
    const result = await userRepository.setStatus(
      userId,
      UserStatus.onboarded,
      ({ email }) => {
        const response = await mailer.sendWelcomeEmail(email);
        if (!response.ok) {
          Err(new EmailRequestFailedError(response));
        }
      }
    );
    return result.mapErr((err) => SendWelcomeEmailError.from(err));
  } catch (err) {
    return Err(new SendWelcomeEmailUnexpectedError(err));
  }
};
```

The idea being that the database update can be rolled back if the call to send the email fails, and that the API call to send the email won't happen if the database update fails.

I know, this is a lot of code for the same amount of core functionality. However, you can sleep better at night with this implementation knowing your children are safe in case the email API goes down.

### The Wild Cards of Coding

The edge cases of your code are often not very obvious, especially when you're trying to ship fast.
Practice makes perfect though, and conditioning your brain to always think about those edge cases is very rewarding.

Consider a scenario in a financial application that calculates a discount based on the number of items purchased.
The application applies discounts on bulk purchases: buying 10 or more items grants a 10% discount, and buying 20 or more items grants a 20% discount:

```typescript
function calculateDiscount(items: number): number {
  if (items >= 10) {
    return 10;
  }
  if (items >= 20) {
    return 20;
  }

  return 0;
}
```

What's not right here?

Did we handle edge cases like a negative amount of items? What about 100,000 items? Should that be considered?

To address those issues, let's use our Result pattern again:

```typescript
function calculateDiscount(items: number): Result<number, CalculationError> {
  if (items < 0) {
    return Err(new CalculationNegativeItemCountError(items));
  } else if (items > 1000) {
    return Err(new CalculationTooManyItemsError(items));
  }

  if (items >= 10) {
    return Ok(10);
  }
  if (items >= 20) {
    return Ok(20);
  }

  return Ok(0);
}
```

### The Enigma of User Input

Another fun and unexpected thing we can deal with is the user's inputs. _Your users will always surprise you with their inputs and the ways they will use your application_.
A good way to simulate your users' interactions with your application is putting your cat on your keyboard and seeing what happens.

Let's look at a simple function that accepts a string representing a user's age, coming directly from a form without any sanitization:

```typescript
function updateAge(userInput: string): string {
  const age = parseInt(userInput);
  return `You are ${age} years old`;
}
```

The above is, of course, pure madness. Consider the following inputs:

```typescript
updateAge("-9");
updateAge("100000");
updateAge("old");
updateAge("");
```

Let's improve it:

```typescript
function updateAge(userInput: string): Result<string, UpdateAgeError> {
  const age = parseInt(ageInput);
  if (isNaN(age)) {
    return Err(new UpdateAgeNotNumberError(age));
  } else if (age <= 0) {
    return Err(new UpdateAgeTooYoungError(age));
  } else if (age > 120) {
    return Err(new UpdateAgeTooOldError(age));
  }
  return Ok(`You are ${age} years old.`);
}
```

## Embrace the Chaos

Thinking about those error and edge cases makes us better developers, and helps us avoid those calls to put out fires when things go wrong.
Always keep a curious and experimental mindset, and look for those pitfalls.

Using the Result pattern is a huge asset to you and your team, and will elevate your error handling throughout your whole stack.

Tools such as static type checking and linters are amazing, but they are no replacement for critical thinking.
Before shipping your code, I advise you to stop, take a broad look at what you're doing, and ask yourself: "what could possibly go wrong here?".
Writing meaningful tests can often shed light on those edge cases you've missed.
