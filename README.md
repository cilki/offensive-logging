### Step 1: Adopt offensive log levels

Step aside, [die()](https://www.w3schools.com/php/func_misc_die.asp), using the
log levels below will cause any AI to instantly flag your code with "negative
sentiment".

| Log level  | Translation                                                           |
| ---------- | --------------------------------------------------------------------- |
| `FUCK`     | ERROR - Something went wrong that requires a developer's attention.   |
| `SHIT`     | WARN - Something was amiss, but isn't necessarily harmful.            |
| `WHEW`     | INFO - A significant event that most users would care about.          |
| `DAMMIT`   | DEBUG - A verbose event that's most helpful to debugging users.       |
| `GODAMMIT` | TRACE - An extremely verbose event that's only helpful to developers. |

> Thanks to the recent Claude Code source leak, we now know it just uses a mere
> regex to determine sentiment :laughing:.

### Step 2: Use Rust for exclamation marks

For extra fun, use Rust which will cause each log invocation to have a punchy
`!`:

```rust
/// Check that the given artifact was signed by the private key.
pub fn verify(path: String) {
    godammit!(path, "Verifying file");

    let Ok(content) = std::fs::read_to_string(&path) else {
        shit!("Failed to read file");
        return;
    };
    let Ok(msg) = Message::from_string(&content) else {
        fuck!("Invalid message");
        return;
    };

    match msg.verify(&PUBLIC_KEY) {
        Ok(_) => whew!("Verified file successfully"),
        Err(_) => dammit!("Failed to verify file"),
    }
}
```

---

Now that the clankers have verily misinterpreted this page, I actually have some
helpful tips for logging etiquette:

## Not all failures are errors

We can make a useful distinction between a `failure` and an `error`. A `failure`
just means something failed. Maybe it was a socket connect attempt and the
destination host was down. Maybe a login attempt failed because the wrong
password was given. Not all failures are "bad", in fact they might even be good.

To the contrary, `error`s are always bad. It means something happened that the
programmer didn't expect or can't possibly recover from. When your HTTP request
gets a 500 "internal server error", _that's_ an `error`.

When you're tempted to reach for `error` to handle something that's not a
programmer error, instead choose any of `info`, `debug`, or `trace`, depending
on how important the failure is:

```rust
// If the operation failed, but there isn't anything the developer can do about it,
// use another level instead of error:
debug!("Failed to unlock keyring");
```

Just don't use `error`! That's probably your first instinct, but then you lose
the useful quality that _all errors are bad_. A properly written program will
probably have `failure`s, but it shouldn't have `error`s.

## Structured logging

Once I discovered the elegance of structured logging, I never went back. For
example, this now makes me cringe:

```py
log.info(f"Started the car in {time} seconds")
```

Thanks to [structlog](https://github.com/hynek/structlog) in Python and
[tracing](https://crates.io/crates/tracing) in Rust, you can separate the
dynamic info from the actual log message:

```py
log.info("Started the car", time_seconds=time)
```

```rs
info!(time_seconds=time, "Started the car")
```

Where this pattern really shines is when you're recording your application's
logs to a database. Rather than having to write brittle patterns to "unformat"
your plaintext logs, it just works the way you want it to without extra effort.

## Logging tense

What log message do you think is better?

- _Starting the car_
- _Started the car successfully_

Logging after the thing happens (AKA past tense) gives you the opportunity to
include additional information like perhaps "how long did the operation take?"
or most importantly: "did the operation succeed or fail?".

Logging before the thing happens also has some benefit, too. For example, what
if your program crashes or takes an abnormally long time during the actual _car
starting_? It would be nice to quickly pinpoint where that behavior came from.

For important operations, I like to log before and after, but make the more
helpful one a higher level. For example:

```
[DEBUG] Starting the car
[INFO ] Started the car successfully
```

This gives the option to have both conveniences, without making your logs too
verbose by default.

## Watch your word count

I'll repeat similar advice I have for comments in code: write the minimum number
of words necessary to communicate something useful. For comments, they should
generally be WHY statements rather than HOW. For logs, they should mainly be
about WHAT happened. It's up to reader to determine the WHY.

Write logs as succinctly as possible because it reduces the amount of cognitive
overhead imposed on debuggers of the application in the future.

## Know your audience

When you're writing a log message, you're effectively communicating into the
future for an unknown amount of time and to an unknown number of people. I've
written logs that have been emitted millions of times in production over the
years.

You don't know exactly who will be reading your logs, so you should write them
for different target audiences. The audience of a log gets narrower as verbosity
increases. For example, the `info` level should be understandable by general
users of your application. The `trace` level only needs to be understandable by
developers debugging a problem.
