---
layout: post
title:  "Error handling in styxe"
date:   2019-09-30 09:00:00 +1000
categories: C++ styxe error-handling
---
Some details about the error handling approach as implemented in libstyxe.

# Intro
[libstyxe][libstyxe-git] is a somewhat simple library - its only job is to parse 9P messages out of a user-provided bytes. And write such messages into a byte stream.
In this case, the error handling strategy is to report the error back to the caller informing them of the invalid input.
That is if the library itself has no implementation issues. Will unit testing should help find later. So let us take a closer look at implementation details
or error reporting.

While refactoring [libstyxe][libstyxe-git] I also realized that the way I have implemented encoders and decoders does not land it for reuse of extensibility.
Having member function such as `Encoder::encode()` and `Decoder::decode` overloaded for value types is easy to develop. All of the overloads have the same
return type of `Solace::Result<>`. However, if I were to add a new data type (for example to support 9P2000.L) - I will need to 'extend' encoders and decoders.
Since designing `libstyxe` - I didn't want to use inheritance to extend classes - I been thinking how else we can do it a way that preserves error handling.

## Problem
To find a way for Encoders and Decoder to be extensible and reusable while keeping error reporting in the familiar form of:
{% highlight C++ %}
  return encoder.encode(value1)
               .then([&]() { return encoder.encode(value2); });
{% endhighlight %}

That is to say, Encoder can take different input types to decode. And encode operations are chainable such that
if one operation failed, the following is not performed.

Oh, do that without using inheritance!


# Options
Initial implementation of `styxe::Encoder` and `styxe::Decoder` were designed to return `Solace::Result` for each operation:

{% highlight C++ %}
styxe::Encoder encoder{...};

auto result = encoder.encode(my_value);
if (!result)
  return result.getError();

result = encoder.encode(my_other_value);
if (!result)
  return result.getError();

{% endhighlight %}

This is all fine and familiar, but my personal experience suggests that [error handling
is not optional][optional-failure] (all puns intended). That is if error handling can be ignored chances are it will be ignored.
Most likely when it is needed. In code terms it means nothing stopping someone from writing code like this:

{% highlight C++ %}
  encoder.encode(my_value);
  encoder.encode(my_other_value);
{% endhighlight %}

It is obviously shorter to read. It also has less obvious issues: if the first `encode` operation fails, second should not be called.
That is to say, we want to have robust error handling. Throwing an exception to signal an error is an option. Except it does not
make it any clearer what error is expected. One would need to decide if it is an exceptional situation.
But more importantly, the problem of error handling remains. It is not enough to throw an error - the error must be handled in a meaningful way.
Working on a support library [libsolace][libsolace-git] I decided to experiment with a different approach to error handling.
That is to return a `Result<>` type that can either be a value or an error. This is an approach implemented by [Rust for example][rust-error-handling].
It is also a topic of various error handling technique has been discussed [many times][error-proposal]. My favorite take on it is using [monads][escaping-hell-monads].

For the purpose of libstyxe design, while the discussion is still raging on, I wanted something easy to use and reusable. Maybe even composable?  
Wouldn't it be nice to write:

{% highlight C++ %}

Result<void, Error> writeSomething(...) {
  ...
  return encoder << my_value
                 << my_other_value;  
}
{% endhighlight %}

This operation chaining does look familiar to C++ developers. What is missing is conversion to a `Result<>`

# Solution
As another experiment, I chose to add a different set of `operator<<` overloads that take Result as an argument in addition to the classical
`operator<<`:

{% highlight C++ %}
Solace::Result<Decoder&, Error>
operator>> (Solace::Result<Decoder&, Error>&& decoder, Solace::uint64& dest);
{% endhighlight %}

This allows me to have operation chains and to return a result. Exactly as a target solution.

The only challenge was to add support to [libsolace][libsolace-git] Result for returning Reference types.
Glad that one is done so now `Result<Value&, Error>` is a valid type.

# Conclusion

For a message parser and writer, the best error handling strategy seems to return an error to the caller. Safe for coding issues - there should be no places for `panic`. That is no need to throw exceptions. After all, if all your function does is converting from user input to value - it is reasonable to expect that a user-provided input can be invalid. The only challenge is to keep a nice interface to allow operation chaining. Using good old free-standing
`operator<<` and `opeartor>>` fits the bill perfectly if these operators are allowed to return Result. That is each IO operation can fail.
This requires a set of overloads that accepts `Result<Encoder&, E> ` and `Result<Decoder&, E>`.
This may not be the most elegant solution but it is extensible at the cost of an extra overload for each new type conversion added. `
I will write another post with the results of using this approach after a while.


[libstyxe-git]: https://github.com/abbyssoul/libstyxe
[libsolace-git]: https://github.com/abbyssoul/libsolace
[optional-failure]: https://isocpp.org/blog/2019/02/optional-is-not-a-failure-phil-nash-meeting-cpp-2018
[rust-error-handling]: https://doc.rust-lang.org/book/ch09-00-error-handling.html
[escaping-hell-monads]: https://philipnilsson.github.io/Badness10k/posts/2017-05-07-escaping-hell-with-monads.html
[error-proposal]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0709r0.pdf
