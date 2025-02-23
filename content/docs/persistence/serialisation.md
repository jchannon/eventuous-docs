---
title: "Serialisation"
description: "How events are serialized and deserialized"
date: 2021-03-10
images: []
weight: 330
---

As described on the [Domain events]({{< ref "events" >}}) page, events must be (de)serializable. Eventuous doesn't care about the serialisation format, but requires you to provide a serializer instance, which implements the `IEventSerializer` interface.

The serializer interface is very simple:

```csharp
interface IEventSerializer {
    object? Deserialize(ReadOnlySpan<byte> data, string eventType);

    byte[] Serialize(object evt);
}
```

### Type map

For deserialization, the serializer will get the binary payload and the event type as string. Event store is unaware of your event types, it just stores the payload in a binary format to the database, along with the event type as string. It is up to you how your strong event types map to the event type string.

{{< alert icon="✋" color="warning" >}}
We do not advise using fully-qualified type names as event types. It will block your ability to refactor the domain model code.
{{< /alert >}}

Therefore, we need to have a way to map strong types of the events to strings, which are used to identify those types in the database and for serialisation. For that purpose, Eventuous uses the `TypeMap`. It is a singleton, which is available globally. When you add new events to your domain model, remember to also add a mapping for those events. The mapping is static, so you can implement it anywhere in the application. The only requirement is that the mapping code must execute when the application starts.

For example, if you have a place where domain events are defined, you can put the mapping code there, as a static member:

```csharp
static class BookingEvents {
    // events are defined here

    public static void MapBookingEvents() {
        TypeMap.AddType<RoomBooked>("RoomBooked");
        TypeMap.AddType<BookingPaid>("BookingPaid");
        TypeMap.AddType<BookingCancelled>("BookingCancelled");
        TypeMap.AddType<BookingImported>("BookingImported");
    }
}
```

Then, you can call this code in your `Startup`:

```csharp
BookingEvents.MapBookingEvents();
```

### Auto-registration of types

For convenience purposes, you can avoid manual mapping between type names and types by using the `EventType` attribute.

Annotate your events with it like this:

```csharp
[EventType("V1.FullyPaid")]
public record BookingFullyPaid(string BookingId, DateTimeOffset FullyPaidAt);
```

Then, use the registration code in the bootstrap code:

```csharp
TypeMap.RegisterKnownEventTypes();
```

The registration won't work if event classes are defined in another assembly, which hasn't been loaded yet. You can work around this limitation by specifying one or more assemblies explicitly:

```csharp
TypeMap.RegisterKnownEventTypes(typeof(BookingFullyPaid).Assembly);
```

If you use the .NET version that supports module initializers, you can register event types in the module. For example, if the domain event classes are located in a separate project, add the file `DomainModule.cs` to that project with the following code:

```csharp
using System.Diagnostics.CodeAnalysis;
using System.Runtime.CompilerServices;
using Eventuous;

namespace Bookings.Domain; 

static class DomainModule {
    [ModuleInitializer]
    [SuppressMessage("Usage", "CA2255", MessageId = "The \'ModuleInitializer\' attribute should not be used in libraries")]
    internal static void InitializeDomainModule() => TypeMap.RegisterKnownEventTypes();
}
```

Then, you won't need to call the `TypeMap` registration in the application code at all.

### Default serializer

Eventuous provides a default serializer implementation, which uses `System.Text.Json`. You just need to register it in the `Startup` to make it available for the infrastructure components, like [aggregate store]({{< ref "aggregate-store" >}}) and [subscriptions]({{< ref "subs-concept" >}}).

Normally, you don't need to register or provide the serializer instance to any of the Eventuous classes that perform serialization and deserialization work. It's because they will use the default serializer instance instead.

However, you can register the default serializer with different options, or a custom serializer instead:

```csharp
services.AddSingleton<IEventSerializer>(
    new DefaultEventSerializer(
        new JsonSerializerOptions(JsonSerializerDefaults.Default)
    )
);
```

You might want to avoid registering the serializer and override the one that Eventuous uses as the default instance:

```csharp
var defaultSerializer = new DefaultEventSerializer(
    new JsonSerializerOptions(JsonSerializerDefaults.Default)
);
DefaultEventSerializer.SetDefaultSerializer(serializer);
```

### Metadata serializer

In many cases you might want to store event metadata in addition to the event payload. Normally, you'd use the same way to serialize both the event payload and its metadata, but it's not always the case. For example, you might store your events in Protobuf, but keep metadata as JSON.

Eventuous only uses the metadata serializer when the event store implementation, or a producer can store metadata as a byte array. For example, EventStoreDB supports that, but Google PubSub doesn't. Therefore, the event store and producer that use EventStoreDB will use the metadata serializer, but the Google PubSub producer will add metadata to events as headers, and won't use the metadata serializer.

For the metadata serializer the same principles apply as for the event serializer. Eventuous has a separate interface `IMetadataSerializer`, which has a default instance created on startup by implicitly. You can register a custom metadata serializer as a singleton or override the default one by calling `DefaultMetadataSerializer.SetDefaultSerializer` function. 
