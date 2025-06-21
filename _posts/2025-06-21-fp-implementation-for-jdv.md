---
layout: post
title: Implementing JSON Data Validator with functional programming
---

After release of the [blog post about the JSON Data Validation](https://jesinski.github.io/2025/06/14/json-data-validation.html){:target="\_blank"}, the user on X [@sseraphini](https://x.com/sseraphini){:target="\_blank"} suggested to implement the framework using functional programming style.

One of the issues I described with the OOP implementation was the huge quantity of files necessary to implement a full validation flow, therefore his suggestion was more than welcome to try to solve this problem.

Functional programming helps in this situation, as we can have multiple validations in the same folder, with as much validation logic and as little boiler plate code around that comes with the OOP solution.

By grouping related function in the same file, and having one main function creating the validation structure / defining the validation flow, we can have less files without loosing the readability, and most importantly keeping each validation logic clear to the developer.

```ts
export async function UserValidator(payload: any): Promise<ValidationResult> {
  const validator = Composite(
    Sequence(
      EmailShouldBeDefined,
      EmailHasDollarSign,
      EmailShouldHaveAtLeast5Characters
    ),
    NameShouldHaveAtLeast3Characters
  );

  return validator(payload);
}

function NameShouldHaveAtLeast3Characters(payload: any): ValidationResult {
  const name = payload.name;
  if (name.length < 3) {
    return { valid: false, messages: ["Name is too short"] };
  } else {
    return { valid: true, messages: [] };
  }
}

function EmailShouldHaveAtLeast5Characters(payload: any): ValidationResult {
  const email = payload.email;
  if (email.length < 5) {
    return {
      valid: false,
      messages: ["Email must be at least 5 characters long"],
    };
  } else {
    return { valid: true, messages: [] };
  }
}

async function EmailShouldBeDefined(payload: any): Promise<ValidationResult> {
  const email = payload.email;
  if (!email) {
    return { valid: false, messages: ["Email is required"] };
  } else {
    return { valid: true, messages: [] };
  }
}

function EmailHasDollarSign(payload: any): ValidationResult {
  const { email } = payload;

  if (email.includes("$")) {
    return { valid: true, messages: ["Email has $ symbol."] };
  }

  return { valid: true, messages: [] };
}
```

To avoid having different contexts mixed in the same files, as the validations quantity grows, we could separate the function into files by their context, for example, email validations, name validations, age validations and so on.

## Functions implementation

The interfaces are the same used in OOP style. Again, we have two main helper functions: Sequence (implementing the Chain of Responsibility) and Composite (which implements the Composite).

Both of them receives N amount of functions to be executed - that must follow our signature contract `(payload: any) => Promise<ValidationResult> | ValidationResult`.

### Chain / Sequence of validations

```ts
export function Sequence(
  ...args: Array<Validator["validate"]>
): Validator["validate"] {
  return async (payload: any): Promise<ValidationResult> => {
    const messages: string[] = [];
    for (const validator of args) {
      const result = await validator(payload);
      if (!result.valid) {
        return { valid: false, messages: [...messages, ...result.messages] };
      }
      messages.push(...result.messages);
    }

    return { valid: true, messages };
  };
}
```

We receive the functions and return another function. Once the returned function is called with a payload, the validation starts.

The execution will stop as soon as we encounter our first invalid validation and it will return the messages generated until that point.

### Composite / Group of validations

```ts
export const Composite = (
  ...args: Array<Validator["validate"]>
): Validator["validate"] => {
  return async (payload: any): Promise<ValidationResult> => {
    const result = await Promise.all(args.map((fn) => fn(payload)));
    return {
      valid: result.every((res) => res.valid),
      messages: result.flatMap((res) => res.messages),
    };
  };
};
```

Again we receive N quantity of validations to be executed, and returns a function to be called.

Composite differs from Sequence, as the functions received through parameters will be executed even if one fails. At the end, we aggregate the result and return it.

## Follow up question

What would happen if, on the Sequence implementation, we instantiate the messages array outside the returned function?

```ts
export function Sequence(
  ...args: Array<Validator["validate"]>
): Validator["validate"] {
  const messages: string[] = []; // Messages is outside the returned function
  return async (payload: any): Promise<ValidationResult> => {
    for (const validator of args) {
      const result = await validator(payload);
      if (!result.valid) {
        return { valid: false, messages: [...messages, ...result.messages] };
      }
      messages.push(...result.messages);
    }

    return { valid: true, messages };
  };
}
```

This might cause a "message leakage" depending how we use the Sequence function because the messages are now bound to the Sequence function and not the Sequence's returned function.

This may duplicate messages, and scenarios like the test below would fail. The second call to chainValidator would have "garbage" messages from the first run.

```ts
it("should not leak messages", async () => {
  const chainValidator = Sequence(
    () => ({ valid: true, messages: ["Valid"] }),
    () => ({ valid: false, messages: ["Invalid"] }),
    () => ({ valid: true, messages: [] })
  );

  const result1 = await chainValidator({});
  assert.deepEqual(result1, {
    valid: false,
    messages: ["Valid", "Invalid"],
  });

  const result2 = await chainValidator({});
  assert.deepEqual(result2, {
    valid: false,
    messages: ["Valid", "Invalid"], // Would wrongly return ["Valid", "Valid", "Invalid"]
  });
});
```

You might wonder why I added such test.

Gustavo Jesinski
