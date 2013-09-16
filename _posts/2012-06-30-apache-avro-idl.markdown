---
layout: post
title:  "Apache Avro IDL"
date:   2012-06-30
tags: java avro idl
comments: true
---

We have been using Apache Avro at my workplace primarily for remote procedure calls (RPC). The simple integration with
dynamic languages and its fast, binary data format provide a better alternative to using XML-based protocols like SOAP.

Avro allows for 2 different protocol specification formats: JSON and IDL. Until very recently, we had been using the
JSON format but it never felt natural or convenient. Case in point below:

{% highlight json %}
{
  "namespace": "org.sample",
  "protocol": "AccountService",
  "types": [
    {
      "name": "Account",
      "fields": [
        {
          "name": "id",
          "type": "long"
        },
        {
          "name": "name",
          "type": "string"
        },
        {
          "name": "description",
          "type": [
            "null",
            "string"
          ],
          "default": null
        }
      ],
      "type": "record"
    }
  ],
  "messages": {
    "addAccount": {
      "response": "Account",
      "request": [
        {
          "name": "name",
          "type": "string"
        },
        {
          "name": "description",
          "type": [
            "null",
            "string"
          ]
        }
      ]
    }
  }
}
{% endhighlight %}

After reading an excellent post by James Baldassari on RPC with Avro, I decided to use IDL instead.

Avro's Interface Definition Language (IDL) provides a higher-level language for describing the protocol and feels more
similar to common programming languages like Java and Python.

The protocol specification above could alternatively be expressed in IDL as follows:

    @namespace("org.sample")
    protocol AccountService {

        record Account {
        long id;
        string name;
        union {null, string} description = null;
        }

        Account addAccount(string name, union {null, string} description);
    }

Even though code generation is not required to implement RPC protocols in Avro, it serves two purposes in statically
typed languages like Java: optimization and method-like invocation for the client.

The [avro-compiler][avro-compiler] module provides an Ant task to generate Java code from the JSON-format protocol but there is no task
to do the same for IDL. Fortunately though, the avro-compiler module already included an IDL parser to convert IDL to
JSON, so it was very simple to write a custom Ant task for this purpose:

{% highlight java linenos %}
import org.apache.avro.Protocol;
import org.apache.avro.compiler.idl.Idl;
import org.apache.avro.compiler.idl.ParseException;
import org.apache.avro.compiler.specific.ProtocolTask;
import org.apache.avro.compiler.specific.SpecificCompiler;
import org.apache.tools.ant.BuildException;

import java.io.File;
import java.io.IOException;

public class IdlTask extends ProtocolTask {
    @Override
    protected void doCompile(File src, File dir) throws IOException {
        Idl parser = new Idl(src);
        try {
            Protocol protocol = parser.CompilationUnit();
            SpecificCompiler compiler = new SpecificCompiler(protocol);
            compiler.setStringType(getStringType());
            compiler.compileToDestination(src, dir);
        } catch (ParseException e) {
            throw new BuildException(e);
        }
    }
}
{% endhighlight %}

Hopefully this will save you some hassle.

[avro-compiler]: http://central.maven.org/maven2/org/apache/avro/avro-compiler/