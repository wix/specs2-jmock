# specs2-jmock 
[![Build Status](https://github.com/wix/specs2-jmock/actions/workflows/sbt-test.yml/badge.svg)](https://github.com/wix/specs2-jmock/actions/workflows/sbt-test.yml?query=branch%3Amaster)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.wix/specs2-jmock_2.12/badge.svg?style=flat)](http://mvnrepository.com/artifact/com.wix/specs2-jmock_2.12)

This is a specs2 adapter + DSL for using the popular mocking framework JMock

# Installation
### Maven
```xml
<dependency>
    <groupId>com.wix</groupId>
    <artifactId>specs2-jmock_${scala.version}</artifactId>
    <version>1.6.3</version>
    <scope>test</scope>
</dependency>
```
### SBT
```sbt
libraryDependencies += "com.wix" %% "specs2-jmock" % "1.6.3"
```

* for latest version check [releases](https://github.com/wix/specs2-jmock/releases)

# Usage
Mixin the `JMock` trait to your Specification class, and use the DSL.

# Example
```Scala
import com.wixpress.common.specs2.JMock
import org.specs2.mutable.Specification

trait FooTrait {
  def bar: String
  def baz: Unit
  def foo(i: Int, s: String): Int
}

class JMockTest extends Specification with JMock {
  "My JMock test" should {
    "Do something" in {
      val mockFoo = mock[FooTrait]
      checking {
        allowing(mockFoo).bar.willReturn("foo")
        oneOf(mockFoo).baz
        // New DSL flavor - allows mixing values and matchers, and more!
        expect.allowing(mockFoo)(_.foo(42, having(any[String]))) willReturn 43
      }
      val result = mockFoo.bar
      mockFoo.baz
      result must be equalTo "foo"
      
      mockFoo.foo(42, "blah") === 43
    }
    //...
  }
}
```

You can see more examples in the [tests](/src/test/scala/com/wixpress/common/specs2).


# Support for advanced Scala features
## Methods with default argument values
Out of the box `JMock` doesn't support calling methods with default argument values 
(without explicitly providing all argument values)

```scala
import org.jmock.AbstractExpectations.returnValue
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: Int, s: String = "foo") }
class JMockTest extends Specification {
  "test" in {
    val mockery = new Mockery
    val mockFoo = mockery.mock(classOf[FooTrait])
    mockery.checking(new Expectations() {{
      allowing(mockFoo).bar(1);
      will(returnValue(42))
    }})
    mockFoo.bar(1) === 42
  }
}
```
The above will fail with:
```
unexpected invocation: fooTrait.bar$default$2()
```
The reason is that the Scala compiler generates the following additional method in the interface representing `FooTrait`
```java
public interface FooTrait {
   // ... 
   public String bar$default$2() {
       return "foo";
   }
}
```
And at the call site, `foo.bar(1)` is converted to `foo.bar(1, foo.bar$default$2())`

`specs2-jmock` fixes this, by not passing the calls to such generated methods to JMock interceptor, but rather calling the generated implementations instead.\
For backward compatibility, by default, when capturing expectations, default value methods (like `foo.bar$default$2()`) return `null`/`0`/`false`... etc.,
instead of the actual default values, as defined on the mocked trait/class.
So by default:

```scala
import com.wixpress.common.specs2.JMock
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: Int, b: Boolean = true) }
class Test extends Specification with JMock {
  "test" in {
    val mockFoo = mock[FooTrait]
    checking {
      // captures expectation of bar(3, false) for comaptibility
      allowing(mockFoo).bar(3) willReturn 42
      // expects bar(2, true) - as this is a new DSL, no compatibility required 
      expect.allowing(mockFoo)(_.bar(2)) willReturn 43
    }
    mockFoo.bar(3, false) === 42
    mockFoo.bar(2) === 43  
    mockFoo.bar(3) === 42 // unexpected invocation: expected bar(2, false)
    
  }  
}
```
To disable this compatibility behavior call `disableDefaultArgsCompatibility()` at the beginning of the test.
You can also wrap your expectation definition in `withoutDefaultArgsCompatibility {}`.
This compatibility behavior is disabled when using the new DSL flavor, see bellow


*Note*: for mocked traits this doesn't work under Scala 2.11, as 2.11 compiler doesn't use Java 8 default interface methods.
Mocked classes (when using `useClassImposterizer()`) are supported. 

### Using matchers in expectations
If you want to set up an expectation using a matcher instead of a concrete value the following won't work:

```scala
import com.wixpress.common.specs2.JMock
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: Int, s: String = "foo") }
class Test extends Specification with JMock {
  "test" in {
    val mockFoo = mock[FooTrait]
    checking {
      allowing(mockFoo).bar(having(beGreaterThan(1))) willReturn 42
    }
    mockFoo.bar(2) === 42    
  }  
}
```
The above will fail with:
```
java.lang.IllegalArgumentException: not all parameters were given explicit matchers: either all parameters must be specified by matchers or all must be specified by values, you cannot mix matchers and values
```
This because `.bar(having(beGreaterThan(1)))` is actually equivalent to `.bar(having(beGreaterThan(1)), "foo")` and JMock doesn't allow mixing values and matchers when defining expectations.

This can be worked around by passing all the method arguments as matchers:
```scala
allowing(mockFoo).bar(having(beGreaterThan(1)), having(beEqualTo("foo"))) willReturn 42
```
The downside, is that now your test code has to be aware of the default values of the arguments, and if they change your test will need to change as well.

Another way is to use a **New JMock DSL flavor** - accessed using `expect.` prefix - that actually allows mixing values and matchers in method arguments.\
The methods under `expect` (`allowing`, `oneOf`, etc.) are macros, that generate code where values are wrapped in `having(beEqualTo())`, if one of the argumets is a matcher.
The macros also support named arguments passed out of order. 

```scala
import com.wixpress.common.specs2.JMock
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: Int, s: String = "foo", boolean: Boolean = false) }
class Test extends Specification with JMock {
  "test" in {
    val mockFoo = mock[FooTrait]
      checking {
        expect.allowing(mockFoo)(_.bar(having(beGreaterThan(1)), b = true)) willReturn 42
        //// expands to:
        // val capturing = allowing(mocked);
        // val p1 = having(beGreaterThan(1));
        // val p2 = having(beEqualTo(capturing.doSomething$default$2));
        // val p3 = having(beEqualTo(true));
        // capturing.doSomething(p1, p2, p3) willReturn 42
      }
    mockFoo.bar(2) === 42    
  }  
}
```

## By-Name method arguments
Out of the box in JMock you can't set expectations on method arguments that are declared by name (for example `def bar(i: => Int)`)
```scala
import org.jmock.AbstractExpectations.returnValue
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: => Int) }
class JMockTest extends Specification {
  "test" in {
    val mockery = new Mockery
    val mockFoo = mockery.mock(classOf[FooTrait])
    mockery.checking(new Expectations() {{
      allowing(mockFoo).bar(2);
      will(returnValue(42))
    }})
    mockFoo.bar(2) === 42
  }
}
```
The above will fail, as what is passed to the method `bar` at runtime is actually the equivalent of `() => 2` of type `scala.Function0` and not `Int`.
And two different instances of this lambda are not equal!

To fix this `specs2-jmock` will evaluate the lambdas and compare the values they produce.

```scala
import com.wixpress.common.specs2.JMock
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: => Int, s: String) }
class Test extends Specification with JMock {
  "test" in {
    val mockFoo = mock[FooTrait]
    checking {
      allowing(mockFoo).bar(1, "foo") willReturn 42
    }
    mockFoo.bar(1, "foo") === 42    
  }  
}
```
This actually works, but only with values. Matchers won't work as `having` will be called out of order.\
If you need matchers, you will need to use the new DSL flavor:

*Note*: at runtime there is no difference between a method argument declared as `a: => T` and as `a: () => T`, 
so this will work the same for both types.

```scala
import com.wixpress.common.specs2.JMock
import org.jmock._
import org.specs2.mutable.Specification

trait FooTrait { def bar(i: => Int, s: String) }
class Test extends Specification with JMock {
  "test" in {
    val mockFoo = mock[FooTrait]
    checking {
      expect.allowing(mockFoo)(_.bar(having(beGreaterThan(1)), "foo")) willReturn 42
    }
    mockFoo.bar(2, "foo") === 42    
  }  
}
```


# License
Use of this source code is governed by a BSD-style license which you can find [here](/LICENSE.md).

