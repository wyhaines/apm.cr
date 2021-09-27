# Building APM (for the Crystal Language)

Several months ago, I wrote about the [basics of how an APM system works](https://www.therelicans.com/wyhaines/the-basicest-basics-of-how-an-apm-system-works-847).

That article was an overview of what APM is, and it presented a very minimal, proof-of-concept APM library for software written in the Crystal language. Since that time, I have been working on a number of different libraries, with the singular goal in mind of building a complete APM system for the Crystal language. There is still a lot of work to do, but it is time to unveil both what has been done so far, how it works, and where it is going.

An APM system is more than just an instrumentation library. A good APM system should be simple to integrate into your own project. It should not require that the developer manually write any instrumentation code in order to start collecting useful information; they should be able to confidently insert the APM system into their software, and immediately realize a benefit. It should, however, permit a developer to instrument parts of their application which may not have received automatic instrumentation.

At the same time, integration of an APM system _must not_ ever change how any given code within your project works. It's behavior should introduce no side effects, and if the APM system is turned off, it should function as closely as a NOP as possible.

So, without further preamble, let's look at each of the elements that have been built so far, and how they come together to form an APM system.

## Automatic Code Injection

One of the major returns on investment with an APM system is that it can be used to automatically instrument code which is not otherwise instrumented. With a very dynamic language, like Ruby, doing this is fairly straightforward. Ruby provides a couple different approaches for this sort of dynamic code injection, *alias_method* and *Module#prepend*. Katherine Wu [wrote a blog post](https://newrelic.com/blog/best-practices/ruby-agent-module-prepend-alias-method-chains) some years ago that lays out details about how each of these work, if you want to better understand the Ruby way.

The Crystal language, unlike Ruby, is compiled, statically type checked, and is not dynamic in the same way that Ruby is. All is not lost, though. Crystal does provide some powerful metaprogramming facilities in the form of its [macro language](https://crystal-lang.org/reference/syntax_and_semantics/macros/index.html), which can be leveraged to inject instrumentation into arbitrary code.

Crystal provides both `include` and `extend`, like Ruby, but it does not currently provide a `prepend`. It also does not provide a conventient syntax for [creating method aliases](https://github.com/wyhaines/alias_method.cr). However, Crystal classes can be reopened, and when a method is redefined, there is a `previous_def` which can be called in order to access the previous definition of the method.

If you were to use this manually to inject some code into a method, it might look something like this:

```crystal
# This is the code that will be instrumented. We will pretend that it
# isn't defined in the same place as the instrumentation code which follows.
class MyClass
  def my_method
    puts "Hello, world!"
  end
end

# Now let's inject some instrumentation into `MyClass#my_method`.
class MyClass
  def my_method
    start = Time.monotonic
    previous_def
  ensure
    finish = Time.monotonic
    puts "Time elapsed: #{finish - start.not_nil!} seconds"
  end
end

puts
mc = MyClass.new
mc.my_method
```

When this code is ran:

```
❯ crystal run --release v.cr

Hello, world!
Time elapsed: 00:00:00.000002100 seconds
```

That is useful. Given a method, one can write code which replaces the original method, wrapping it in your own code. However, an APM system has to instrument a wide variety of methods across many different libraries. It is a significant maintenance burden to explicitly write and maintain instrumentation code for every method that is instrumented, particularly since most of the code is boilerplate that is likely to be very similar from one method to the next.

### [tracer.cr](https://github.com/wyhaines/tracer.cr/tree/main)

Crystal macros allow access to a wide variety of information about the program code, including the argument signatures and the source code for any given method. With this information, it is possible to build a generic, flexible method tracing mechanism for Crystal.

The `tracer.cr` library is intentionally very low level. It only concerns itself with making it easy to wrap any given method in tracing code. All implementations of that tracing code are left to other implemetations. A simple one could be:

```crystal
class MyTracer
  Traces = Hash(String, Array(Time::Span)).new {|h, k| h[k] = [] of Time::Span}

  def self.do_trace(method_name, phase, method_identifier)
    MyTracer::Traces[method_identifier] << Time.monotonic

    if phase == :after
      puts "Elapsed Time: #{MyTracer::Traces[method_identifier][1] - MyTracer::Traces[method_identifier][0]}"
    end
  end
end
```

With that, the previous example of manually wrapping a method in tracing code could be rewritten as:

```crystal
require "tracer"

# This is the code that will be instrumented. We will pretend that it
# isn't defined in the same place as the instrumentation code which follows.
class MyClass
  VERSION = "0.1.0"
  def my_method
    puts "Hello, world!"
  end
end

# Now let's inject some instrumentation into `MyClass#my_method`.
class MyClass
  trace("my_method") {|method_name, phase, method_identifier| MyTracer.do_trace(method_name, phase, method_identifier)}
end

puts
mc = MyClass.new
mc.my_method

```

When this version of the code is ran:

```
❯ crystal run --release src/v.cr

Hello, world!
Elapsed Time: 00:00:00.000003700
```

There is a slight, measurable approach to the execution speed, given the simplicity of the method that is being wrapped, and the approach that is being used. It is possible that the tracing code might see implementation improvements in the future which can reduce some of this, but in the context of real methods (which tend to do considerably more work than outputting "Hello, world!"), the incremental cost of tracing is negligible.

It is still likely that some library methods will require hand curated code injection, but this facility will work for many of the pieces that the APM system will want to measure.

If you would like to know more about how the tracing library works, it is [fully documented](https://wyhaines.github.io/tracer.cr/) within its [GitHub repository](https://github.com/wyhaines/tracer.cr).

With that piece taken care of, the other major component involved in building an APM system is the telemetry framework itself.

## Telemetry Framework

New Relic has agents for a variety of different programming languages