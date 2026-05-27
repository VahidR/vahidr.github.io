---
title: "Java Deserialization Attacks As I See Them"
date: 2026-05-27T11:31:59+01:00
type: "post"
tags: ["java", "security"]
---

## What is Serialization?

Serialization means turning an object from memory into a flat format. The flat format can be bytes, JSON, or XML. 
After you serialize an object, you can send it over the network, save it to disk, or store it in a database.

Deserialization is the opposite. It takes the flat format and builds the object back into memory.

People use serialization for many things, not only for the network:

- Sending data over the network (for example, JSON in REST APIs)
- Saving game state or cache to disk
- Storing objects in a database
- Sending messages through queues like Kafka

## Java's Built-in Serialization

Java has its own serialization system. You make a class implement `Serializable`. Then you can write it to bytes with `ObjectOutputStream` and read it back with `ObjectInputStream`.

This system is old. Today, most people avoid it because it has serious security problems. Let's see why.

## The Core Problem: Deserialization Attacks

The danger lives in this one line:

```java
Object obj = objectInputStream.readObject();
```

When Java reads bytes with `readObject()`, it does much more than just parse data. It:

1. Creates new objects from classes on your classpath
2. Fills their fields with values from the byte stream
3. Calls special "magic methods" on them, like `readObject()`, `readResolve()`, and `finalize()`

The last point is the big problem. The attacker chooses which classes get created. And the magic methods on those classes run automatically — **before** your own code can check anything.

## Gadget Chains

An attacker usually does not find one single class that runs dangerous code. Instead, they build a "gadget chain". A gadget chain is a sequence of objects. When one object is deserialized, it calls a method on another object. That object calls a method on a third object. The chain continues until it reaches something dangerous, like `Runtime.exec()`.

The famous example is the **Apache Commons Collections** chain from 2015 (CVE-2015-4852). It worked roughly like this:

- A JDK class gets deserialized
- Its `readObject` calls a method on a `Map`
- The `Map` is a special `LazyMap` from Commons Collections
- The `LazyMap` uses a `ChainedTransformer`
- The transformer chain calls `Runtime.getRuntime().exec("calc.exe")`

The attacker sends one byte stream. Your server calls `readObject()`. You get remote code execution. No password needed. No memory bug. Just Java doing what it was designed to do.

A tool called **ysoserial** collects many of these chains for popular libraries (Spring, Hibernate, Groovy, and more). With ysoserial, an attacker can just pick a chain that matches the libraries on your server.

## Why This Bug Class is So Bad

A few things make these attacks especially dangerous:

- **Your code is not "wrong"**. The bug comes from how Java works plus the libraries on your classpath. You can be at risk through a library you never use directly.
- **The attack surface is large**: RMI, JMX, JNDI, HTTP session cookies, message queues, caches. Anywhere bytes get deserialized.
- **Constructors do not run**. Java rebuilds objects field by field with reflection. Safety checks in your constructors do nothing.
- **By the time you can check the object, it is too late**. The magic methods already ran during parsing.

## What About JSON?

JSON itself is safe. The danger comes back when a JSON library does **polymorphic deserialization** — when the JSON says "make this exact class" and the library obeys.

Jackson had many CVEs about this (the `enableDefaultTyping` feature). XStream and SnakeYAML had similar problems.

The rule is simple: **if the attacker controls which class gets created, you have a deserialization vulnerability**, no matter the format.

Modern Jackson turns off polymorphic typing by default. SnakeYAML's `SafeConstructor` does the same for YAML.

## How to Defend

In order from best to worst:

1. **Do not use Java's built-in serialization for untrusted input.** Use JSON or another data-only format. This is the real fix.
2. **Use serialization filters** (`ObjectInputFilter`, added in JDK 9). They let you allow only certain classes. Useful when you cannot migrate.
3. **Keep your classpath small**. Fewer libraries means fewer possible gadgets.
4. **Isolate the network** for services that must use RMI or JMX.

## Recent Attacks in 2026

The bug class is not new, but new CVEs keep coming every year. Here is one in 2026:

### CVE-2026-27172 — Apache Camel (the newest one)

In May 2026, a new flaw was found in the `camel-consul` component of Apache Camel. The `ConsulRegistry` class read Java-serialized values from the Consul KV store and passed them to `ObjectInputStream.readObject()` without a filter. An attacker who could write to the Consul KV store could inject a bad object and get code execution inside the Camel process.

The sad part: the same bug class was already fixed in other Camel components (CVE-2024-22369, CVE-2024-23114, CVE-2026-25747). The fix simply missed this one component.
You can see the fix here: [Apache Camel Pull Request #21530](https://github.com/apache/camel/pull/21530/files?diff=split&w=0)
## The Pattern

If you line up these CVEs, you see the same story over and over:

- A library calls `readObject()` on data that could come from an attacker.
- No `ObjectInputFilter` is set.
- A gadget chain on the classpath turns this into code execution.

The technique has not really changed since 2015. What changes is the surface — new libraries, new agents, new features. The fix is also the same: use `ObjectInputFilter`, or better, do not use Java serialization at all for untrusted input.

---

For comments, please send me 📧 [*<span style="border-bottom: 1px dashed #666;">an email</span>*](mailto:vahid.rafiei@gmail.com).
