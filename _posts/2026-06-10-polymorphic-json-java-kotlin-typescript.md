---
layout: post
title: "Discriminated Unions Across the Stack: Polymorphic JSON in Java, Kotlin and TypeScript"
---

This week I reviewed a teammate's pull request that fixed a sneaky bug: a single, well-formed request had started returning `422 Unprocessable Entity` after a dependency bump. The payload was valid. The code was unchanged. The only thing that had moved was Spring Boot 3, which dragged Jackson 2.17 along with it.

Reviewing that fix sent me down a rabbit hole, and I came out understanding polymorphic deserialization better than I had after years of using it. The culprit was exactly that: one JSON field that can hold different shapes depending on a sibling "type" field. It is a pattern every backend eventually needs, it has a name in every language, and it has sharp edges in every language too. This post is what I learned — how to model it well in **Java**, **Kotlin**, and **TypeScript / Next.js**, and the gotchas worth knowing before they bite.

After reviewing that PR I went looking for a solid treatment of the idea, and found it in Scott Wlaschin's *[Domain Modeling Made Functional](https://pragprog.com/titles/swdddf/domain-modeling-made-functional/)* (Pragmatic Bookshelf, 2018). He calls them *sum types* or *discriminated unions* and builds a whole design philosophy around them — **make illegal states unrepresentable**. On the front-end side, the [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) uses the exact same name. Different ecosystems, one idea.

> The examples below use a made-up notification domain so nothing here is tied to any real codebase.

## The problem in one example

Say you send notifications over different channels. The JSON carries a discriminator and an optional typed payload:

```json
{ "channel": "PUSH", "delivery_options": { "device_token": "abc123", "badge": 3 } }
{ "channel": "EMAIL" }
```

`PUSH` needs delivery options (a device token, a badge count). `EMAIL` and `SMS` do not. The reader has to look at `channel` to decide how to parse `delivery_options` — or whether it is even there. That "look at one field to decide the type of another" is the whole game.

## Java with Jackson

Jackson models this with `@JsonTypeInfo`. Three knobs do the work:

```java
@JsonTypeInfo(
    use      = JsonTypeInfo.Id.NAME,          // the discriminator is a logical name
    include  = JsonTypeInfo.As.PROPERTY,      // WHERE the discriminator lives
    property = "channel"                      // the field name that holds it
)
@JsonSubTypes({                               // the name -> class table
    @JsonSubTypes.Type(value = PushOptions.class, name = "PUSH")
})
abstract class DeliveryOptions { }
```

- `use` = **what** the discriminator is (a name, a class, a custom resolver).
- `include` = **where** it sits in the JSON.
- `property` = **the field name** that carries it.
- `@JsonSubTypes` = the **lookup table** mapping each name to a concrete class.

### Where the discriminator lives: `include`

This is the knob most people never think about, and it is exactly where the bug lived.

| `As.*` | JSON shape |
|---|---|
| `PROPERTY` (default) | `{ "channel": "PUSH", "device_token": "abc123" }` |
| `WRAPPER_OBJECT` | `{ "PUSH": { "device_token": "abc123" } }` |
| `WRAPPER_ARRAY` | `[ "PUSH", { "device_token": "abc123" } ]` |
| `EXTERNAL_PROPERTY` | `{ "channel": "PUSH", "delivery_options": { "device_token": "abc123" } }` |

With `PROPERTY` the type tag lives *inside* the value object. With `EXTERNAL_PROPERTY` the tag and the value are **two sibling fields**. Our example uses the external form because `channel` and `delivery_options` are siblings:

```java
@JsonProperty("channel")
Channel channel;

@JsonTypeInfo(use = Id.NAME, include = As.EXTERNAL_PROPERTY, property = "channel",
        defaultImpl = NoOpOptions.class)       // channels with no options fall back here
@JsonSubTypes({
    @JsonSubTypes.Type(value = PushOptions.class, name = "PUSH")
})
@JsonProperty("delivery_options")
DeliveryOptions deliveryOptions;
```

`defaultImpl` is the escape hatch for "this type carries no payload": `EMAIL` and `SMS` are not in the `@JsonSubTypes` table, so they resolve to `NoOpOptions` instead of failing.

### The Jackson 2.17 gotcha

Here is the trap. Because the discriminator and the value are *separate* fields in `EXTERNAL_PROPERTY`, you can legitimately receive one without the other:

```json
{ "channel": "EMAIL" }
```

No `delivery_options` — correct, an `EMAIL` notification has none. Jackson **before 2.17** mapped the absent field to `null` and moved on. Jackson **2.17** flipped a default and now throws:

```
com.fasterxml.jackson.databind.exc.MismatchedInputException:
Missing property 'delivery_options' for external type id 'channel'
```

Spring translates that into a `422`. Nothing in *your* code changed — the framework's default did. The fix is one line, restoring the old behaviour:

```properties
# application.properties
spring.jackson.deserialization.fail-on-missing-external-type-id-property=false
```

This only affects `EXTERNAL_PROPERTY`. With `PROPERTY` or the wrapper forms the tag and value are coupled in the same object, so "tag present, value absent" cannot happen and the flag is irrelevant.

> **Lesson:** if you use `EXTERNAL_PROPERTY`, guard it with a test that deserializes the *tag-only* JSON and asserts it does not throw. Frameworks change defaults across majors; a test pins the behaviour you actually depend on.

## Kotlin

Kotlin makes the model nicer with **sealed classes** — the compiler knows the full set of subtypes, so a `when` over them is exhaustive with no `else`.

```kotlin
sealed class DeliveryOptions

data class PushOptions(val deviceToken: String, val badge: Int) : DeliveryOptions()
object NoOpOptions : DeliveryOptions()
```

With Jackson (plus `jackson-module-kotlin`) the annotations are the same as Java:

```kotlin
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.EXTERNAL_PROPERTY,
    property = "channel", defaultImpl = NoOpOptions::class)
@JsonSubTypes(
    JsonSubTypes.Type(value = PushOptions::class, name = "PUSH")
)
val deliveryOptions: DeliveryOptions?
```

The exhaustiveness pays off when you consume the value:

```kotlin
val summary = when (options) {
    is PushOptions -> "push to ${options.deviceToken} (badge ${options.badge})"
    NoOpOptions    -> "no options"
    // no else needed — add a new subtype and this stops compiling until you handle it
}
```

If you control both ends and want JSON without runtime reflection, **kotlinx.serialization** is the idiomatic alternative. It encodes the discriminator for you:

```kotlin
@Serializable
sealed class DeliveryOptions {
    @Serializable @SerialName("PUSH")
    data class Push(val deviceToken: String, val badge: Int) : DeliveryOptions()
}
```

It emits a `"type"` field by default (configurable via `classDiscriminator`). The trade-off: it is great for service-to-service contracts you own, less forgiving when you must match an existing external JSON shape — which is where Jackson's flexibility wins.

## TypeScript and Next.js

TypeScript has the same concept under a different name: **discriminated unions** (a.k.a. tagged unions). A shared literal field — the *discriminant* — lets the compiler narrow the type:

```typescript
type Notification =
  | { channel: "PUSH"; device_token: string; badge: number }
  | { channel: "EMAIL" }
  | { channel: "SMS" };

function summary(n: Notification): string {
  switch (n.channel) {
    case "PUSH":
      return `push to ${n.device_token} (badge ${n.badge})`; // n is narrowed here
    case "EMAIL":
    case "SMS":
      return "no options";
  }
}
```

Exhaustiveness, like Kotlin's sealed classes, is enforceable. Add a `never` sink and any unhandled case fails the build:

```typescript
default: {
  const _exhaustive: never = n;
  return _exhaustive;
}
```

### The catch that bites Next.js API routes

Here is the part that trips people up coming from a typed backend: **TypeScript types vanish at runtime.** A `Notification` annotation gives you zero protection against a malformed request body — `req.json()` returns `any`-shaped data and TypeScript happily trusts it. The discriminated union is a *compile-time* tool; the wire is still untyped.

So in a Next.js route handler you validate at the boundary, and a runtime validator like **zod** mirrors the union exactly:

```typescript
import { z } from "zod";

const Notification = z.discriminatedUnion("channel", [
  z.object({ channel: z.literal("PUSH"), device_token: z.string(), badge: z.number() }),
  z.object({ channel: z.literal("EMAIL") }),
  z.object({ channel: z.literal("SMS") }),
]);

// app/api/notifications/route.ts
export async function POST(req: Request) {
  const parsed = Notification.safeParse(await req.json());
  if (!parsed.success) {
    return Response.json({ errors: parsed.error.issues }, { status: 422 });
  }
  // parsed.data is fully typed AND validated here
  return Response.json({ ok: true });
}
```

`z.discriminatedUnion` reads the tag first and validates against the matching branch only, so the error messages stay precise instead of dumping every branch's failures. And `z.infer<typeof Notification>` gives you the static type for free — one source of truth for both the compiler and the runtime.

> **Lesson:** in a typed backend the framework parses *and* validates at the edge. In Next.js those are two separate jobs — TypeScript handles the first, you own the second. Skipping runtime validation is the most common way a "type-safe" API ships a 500.

## The same idea, three flavours

| | Java | Kotlin | TypeScript |
|---|---|---|---|
| Construct | `@JsonTypeInfo` + `@JsonSubTypes` | sealed class + Jackson / kotlinx | discriminated union |
| Discriminator | any `As.*` placement | same as Java | a literal field |
| Exhaustiveness | runtime only | compile-time (`when`) | compile-time (`never`) |
| Runtime validation | built into Jackson | built into the serializer | **bring your own** (zod) |
| Main gotcha | `EXTERNAL_PROPERTY` + framework default flips | reflection vs kotlinx trade-off | types are erased — validate the wire |

## Takeaways

1. **Polymorphic JSON is universal** — the same modelling problem shows up in every language; only the syntax changes.
2. **Know where your discriminator lives.** `EXTERNAL_PROPERTY` is the most flexible Jackson form and the only one with the "tag without value" failure mode.
3. **Pin framework behaviour with tests.** That `422` came from a default flipping across a major bump, not from anyone's code — and it was caught in review, not by a test. A test on the tag-only payload would have caught it in CI.
4. **In Next.js, types are not validation.** Narrowing is a compile-time gift; the request body is still untrusted. Validate at the boundary with zod and infer the type from the schema.

The next time a "valid" request starts failing after a dependency bump, check whether something polymorphic is being deserialized — and whether a default moved underneath you.
