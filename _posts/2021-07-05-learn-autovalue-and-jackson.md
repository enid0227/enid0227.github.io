---
layout: post
title: "Learn AutoValue and Jackson"
date: 2021-07-05 21:20:15 -0700
categories: autovalue, jackson
---

## What is in this post?

How to use **AutoValue** with **Jackson** to serialize and deserialize JSON.

## Background

As part of my [personal project](https://github.com/enid0227/stocktwits-java-api), I found myself
need to build a client library for a public web service. To get the custom client library going, I
decided to start with the serialization/deserialization process which is a crucial part of the
client library. While customized JSON handling is achievable, I find it less reliable than relying
on the established library to do the trick. After some research on the Internet, I decided to choose
**AutoValue** and **Jackson** to achieve the objective.

## Libraries

### AutoValue

[GitHub Project Link](https://github.com/google/auto/tree/master/value)

#### What is AutoValue

> "AutoValue is a great tool for eliminating the drudgery of writing mundane value classes in Java.
> It encapsulates much of the advice in Effective Java Chapter 2, and frees you to concentrate on
> the more interesting aspects of your program. The resulting program is likely to be shorter,
> clearer, and freer of bugs. Two thumbs up."
>
> -- Joshua Bloch, author, Effective Java

#### What does it have to do with the client library

In this client library I'm building, I use
[value class/object](https://dzone.com/articles/value-objects) to represent the JSON response
returned from the service for the following reasons:

##### 1. Immutability

See
[Benefits of Immutable Objects in Java](https://www.linkedin.com/pulse/20140528113353-16837833-6-benefits-of-programming-with-immutable-objects-in-java/)
to learn more about why immutability is good. And in the client library context, once we received
JSON from the server, it should never be changed.

##### 2. Auto-Generate Value Class Boilerplate

Value class boilerplate methods such as `...Setters, toString(), .equals(), hashCode()` are boring
to implement, but they are critical for many reasons. AutoValue saves this problem and greately
reduce the lines of code need to be manually maintained.

- [AutoValue - Say ByeBye to Class Boilerplate](http://blogs.quovantis.com/autovalue-say-bye-bye-to-value-classes-boilerplate/)
- [Offical Why AutoValue Presentation](https://docs.google.com/presentation/d/14u_h-lMn7f1rXE1nDiLX0azS3IkgjGl5uxp5jGJ75RE/edit#slide=id.g2a5e9c4a8_00)

### Jackson

[GitHub Project Link](https://github.com/FasterXML/jackson)

#### What is Jackson

> Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as
> "JSON for Java".
>
> -- GitHub README

#### What does it have to do with the client library

The response returned from web service is in JSON format. Using a well-established library makes
everything easier.

## Ok, both libraries are great, how can we use them together?

In the below guide/examples, I'm going to demonstrate how to use AutoValue with Jackson library.

_This guide assumes you have some knowledge how to use AutoValue. Please refer to
[Official User Guide](https://github.com/google/auto/blob/master/value/userguide/index.md)_

### Sample Response & AutoValue

Sample AutoValue to respresent the response before applying any Jackson annotations.

**Sample API Response**

```json
{
  "num_friends": 3,
  "num_blocked": 0,
  "num_invite_pending": 1,
  "friends": [
    {
      "name": "John Doe",
      "id": 1
    },
    {
      "name": "Jane Doe",
      "id": 2
    },
    {
      "name": "Justin Doe",
      "id": 3
    }
  ]
}
```

**Sample AutoValue**

```java
// Friend.java - with static create method
@AutoValue
public abstract class Friend {
  public static Friend create(String name, int userId) {
    return new AutoValue_Friend(name, userId);
  }

  public abstract String name();
  public abstract int userId();
}

// FriendResponse.java - with builder
@AutoValue
public abstract class FriendResponse {
  public static Builder builder() {
    return new AutoValue_Friend.Builder();
  }

  public abstract int numFriends();
  public abstract int numBlocked();
  public abstract int numInvitePending();
  public abstract ImmutableList<Friend> friends();

  @AutoValue.Builder
  public abstract class Builder {
    public abstract Builder setNumFriends(int numFriends);
    public abstract Builder setNumBlocked(int numBlocked);
    public abstract Builder setNumInvitePending(int numInvitePending);
    public abstract Builder setFriends(ImmutableList<Friend> friends);
    public abstract FriendResponse build();
  }
}

```

### Serialization

In the previous section, we have already defined all the necessary property for the JSON response in
the AutoValue class definitions. How can we serialize the Java object into JSON string?

#### @JsonProperty annotation

Use `@JsonProperty` to let jackson library knows how a Java object attribute maps to a JSON
attribute. The property name can be different if you want the java object attribute and the
serialized attribute to have a different key name. _Example: `userId (java)--> id (JSON)_

#### @JsonSerialize annotation

Use `@JsonSerialize` to let jackson library knows how to serialize a object.

**Annotated AutoValue Classes**

```java
@AutoValue
@JsonSerialize(as = Friend.class)
public abstract class Friend {
  // ignore create part

  @JsonProperty("name")
  public abstract String name();

  @JsonProperty("id")
  public abstract int userId();
}

// FriendResponse.java - with builder
@AutoValue
@JsonSerialize(as = FriendResponse.class)
public abstract class FriendResponse {

  @JsonProperty("num_friends")
  public abstract int numFriends();

  @JsonProperty("num_blocked")
  public abstract int numBlocked();

  @JsonProperty("num_invite_pending")
  public abstract int numInvitePending();

  @JsonProperty("friends")
  public abstract ImmutableList<Friend> friends();

  // ignore builder parts
}
```

Now, the Java object is ready to be serialized into JSON with jackson library using
[ObjectMapper](https://fasterxml.github.io/jackson-databind/javadoc/2.7/com/fasterxml/jackson/databind/ObjectMapper.html).

```java
final ObjectMapper objectMapper = JsonMapper.builder().build();
String jsonString = objectMapper.writeValueAsString(Friend.create("John Doe", 1));
// returns {"id":1,"name": "John"}
```

### Deserialization

The deserialization counterpart is almost identical to the serialization one. However, depends on
the creation mechanism you choose for the AutoValue `create()` vs `buidler()`, the annotations
needed are different.

### @JsonCreator annotation

Annotated `@JsonProperty` to the `create()` method parameters with the right JSON attribute to map
use for the given parameter. Also, annotate the `create()` method itself with `@JsonCreator`
([javadoc](https://javadoc.io/doc/com.fasterxml.jackson.core/jackson-annotations/2.9.5/com/fasterxml/jackson/annotation/JsonCreator.html))

```java
@AutoValue
public abstract class Friend {

  @JsonCreator
  public static Friend create(
    @JsonProperty("name") String name,
    @JsonProperty("id") int userId) {
    return new AutoValue_Friend(name, userId);
  }
  // ignore accessors part
}
```

### @JsonDeserialize(builder = ...) annotation

Annotate `@JsonProperty` to the `setter` methods inside the builder definition, and annotate the
AutoValue class with `@JsonDeserialize(builder = AutoValue_XXX.Builder.class`
([javadoc](https://fasterxml.github.io/jackson-databind/javadoc/2.9/com/fasterxml/jackson/databind/annotation/JsonDeserialize.html))

```java
@AutoValue
@JsonDeserialize(builder = AutoValue_FriendResponse.Builder.class)
public abstract class FriendResponse {
  public static Builder builder() {
    return new AutoValue_Friend.Builder();
  }

  // ignore accessors parts

  @AutoValue.Builder
  public abstract class Builder {
    @JsonProperty("num_friends")
    public abstract Builder setNumFriends(int numFriends);

    @JsonProperty("num_blocked")
    public abstract Builder setNumBlocked(int numBlocked);

    @JsonProperty("num_invite_pending")
    public abstract Builder setNumInvitePending(int numInvitePending);

    @JsonProperty("friends")
    public abstract Builder setFriends(ImmutableList<Friend> friends);

    public abstract FriendResponse build();
  }
}
```

Now, the JSON response can be deserialized into Java AutoValue using object mapper.

```java
import com.fasterxml.jackson.datatype.guava.GuavaModule;

// ...

ObjectMapper objectMapper =
    JsonMapper().builder()
        .addModule(new GuavaModule()) // ImmutableList dependency
        .build();
FriendResponse response = objectMapper.readValue(
  "{\"num_friends\":3,\"num_blocked\":0,\"num_invite_pending\":1,\"friends\":[{\"name\":\"John Doe\",\"id\":1},{\"name\":\"Jane Doe\",\"id\":2},{\"name\":\"Justin Doe\",\"id\":3}]}",
  FriendResponse.class);
```

## Other Notes

### Configure ObjectMapper

The jackson library uses `ObjectMapper` to handle the serialization and the deserialization. The
behavior of the mapper can be configured to meet your needs. Below is an example mapper I use for
all my convesions.

```java
return JsonMapper.builder()
        // use convert string to Instant
        .addModule(new JavaTimeModule())
        // Immutable collection
        .addModule(new GuavaModule())
        // hide null value per google json guide
        // https://google.github.io/styleguide/jsoncstyleguide.xml?showone=Empty/Null_Property_Values#Empty/Null_Property_Values
        .serializationInclusion(Include.NON_NULL)
        // don't serialize Instant to integer/float timestamp. convert to string instead
        .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
        // don't fail on unknown property (not defined in @JsonProperty) for better maintainability
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
        // sort for testing/debugging purpose]
        .configure(MapperFeature.SORT_PROPERTIES_ALPHABETICALLY, true)
        .build();
```

## TL;DR

### Serialization

- Annotate AutoValue **getters** with `JsonProperty` to let Jackson know how a AutoValue attribute
  should map to a JSON attribute

### Desrialization

- Annotate AutoValue **setters** with `JsonProperty` to let Jackson know how a JSON attribute should
  map to a AutoValue attribute
- Annotate `JsonCreator` if AutoValue uses `create()` method to create an instance
- Annotate `JsonDeserialize(builder = ...)` if AutoValue uses `Builder` to create an instance

## Other Resources

I've used this combination extensively in opensource project:
[stocktwits-java-api](https://github.com/enid0227/stocktwits-java-api). More specificly, in the
[value](https://github.com/enid0227/stocktwits-java-api/tree/main/src/main/java/com/stocktwitlist/api/value)
package.

## Thank you for reading!

Hope this guide helps!
