The Capability System
=====================

Capabilities allow exposing features in a dynamic and flexible way without having to resort to directly implementing many interfaces.

In general terms, each capability provides a feature in the form of an interface, a default implementation which can be requested, and a storage handler for at least this default implementation. The storage handler can support other implementations, but this is up to the capability implementor, so look it up in their documentation before trying to use the default storage with non-default implementations.

Forge adds capability support to TileEntities, Entities, ItemStacks, Worlds, and Chunks, which can be exposed either by attaching them through an event or by overriding the capability methods in your own implementations of the objects. This will be explained in more detail in the following sections.

Forge-provided Capabilities
---------------------------

Forge provides three capabilities: `IItemHandler`, `IFluidHandler` and `IEnergyStorage`

`IItemHandler` exposes an interface for handling inventory slots. It can be applied to TileEntities (chests, machines, etc.), Entities (extra player slots, mob/creature inventories/bags), or ItemStacks (portable backpacks and such). It replaces the old `IInventory` and `ISidedInventory` with an automation-friendly system.

`IFluidHandler` exposes an interface for handling fluid inventories. It can also be applied to TileEntities, Entities, or ItemStacks. It replaces the old `IFluidHandler` with a more consistent and automation-friendly system.

`IEnergyStorage` exposes an interface for handling energy containers. It can be applied to TileEntities, Entities, or ItemStacks. It is based on the RedstoneFlux API by TeamCoFH.

Using an Existing Capability
----------------------------

As mentioned earlier, TileEntities, Entities, and ItemStacks implement the capability provider feature through the `ICapabilityProvider` interface. This interface adds the method `#getCapability`, which can be used to query the capabilities present in the associated provider objects.

In order to obtain a capability, you will need to refer it by its unique instance. In the case of the `IItemHandler`, this capability is primarily stored in `CapabilityItemHandler#ITEM_HANDLER_CAPABILITY`, but it is possible to get other instance references by using the `@CapabilityInject` annotation.

```Java
@CapabilityInject(IItemHandler.class)
static Capability<IItemHandler> ITEM_HANDLER_CAPABILITY = null;
```

This annotation can be applied to fields and methods. When applied to a field, it will assign the instance of the capability (the same one gets assigned to all fields) upon registration of the capability and left to the existing value (`null`) if the capability was never registered. Because local static field accesses are fast, it is a good idea to keep your own local copy of the reference for objects that work with capabilities. This annotation can also be used on a method, in order to get notified when a capability is registered, so that certain features can be enabled conditionally.

The `#getCapability` method has a second parameter, of type `Direction`, which can be used to request the specific instance for that one face. If passed `null`, it can be assumed that the request comes either from within the block or from some place where the side has no meaning, such as a different dimension. In this case a general capability instance that does not care about sides will be requested instead. The return type of `#getCapability` will correspond to a `LazyOptional` of the type declared in the capability passed to the method. For the Item Handler capability, this is `LazyOptional<IItemHandler>`. If the capability is not available for a particular provider, it will return an empty `LazyOptional` instead.

Exposing a Capability
---------------------

In order to expose a capability, you will first need an instance of the underlying capability type. Note that you should assign a separate instance to each object that keeps the capability, since the capability will most probably be tied to the containing object.

There are two ways to obtain such an instance: through the `Capability` itself or by explicitly instantiating an implementation of it. The first method is designed to use a default implementation via `Capability#getDefaultInstance`, if those default values are useful for you. In the case of the Item Handler capability, the default implementation will expose a single slot inventory, which is most probably not what you want.

The second method can be used to provide custom implementations. In the case of `IItemHandler`, the default implementation uses the `ItemStackHandler` class, which has an optional argument in the constructor, to specify a number of slots. However, relying on the existence of these default implementations should be avoided, as the purpose of the capability system is to prevent loading errors in contexts where the capability is not present, so instantiation should be protected behind a check testing if the capability has been registered (see the remarks about `@CapabilityInject` in the previous section).

Once you have your own instance of the capability interface, you will want to notify users of the capability system that you expose this capability and provide a `LazyOptional` of the interface reference. This is done by overriding the `#getCapability` method, and comparing the capability instance with the capability you are exposing. If your machine has different slots based on which side is being queried, you can test this with the `side` parameter. For Entities and ItemStacks, this parameter can be ignored, but it is still possible to have side as a context, such as different armor slots on a player (`Direction#UP` exposing the player's helmet slot), or about the surrounding blocks in the inventory (`Direction#WEST` exposing the input slot of a furnace). Do not forget to fall back to `super`, otherwise existing attached capabilities will stop working.

Capabilities must be invalidated at the end of the provider's lifecycle via `LazyOptional#invalidate`. For owned TileEntities and Entities, the `LazyOptional` can be invalidated within `#invalidateCaps`. For non-owned providers, a runnable supplying the invalidation should be passed into `AttachCapabilitiesEvent#addListener`.

```java
// Somewhere in your TileEntity subclass
LazyOptional<IItemHandler> inventoryHandlerLazyOptional;

// Supplied instance (e.g. () -> inventoryHandler)
// Ensure laziness as initialization should only happen when needed
inventoryHandlerLazyOptional = LazyOptional.of(inventoryHandlerSupplier);

@Override
public <T> LazyOptional<T> getCapability(Capability<T> cap, Direction side) {
  if (cap == CapabilityItemHandler.ITEM_HANDLER_CAPABILITY) {
    return inventoryHandlerLazyOptional.cast();
  }
  return super.getCapability(cap, side);
}

@Override
public void invalidateCaps() {
  super.invalidateCaps();
  inventoryHandlerLazyOptional.invalidate();
}
```

`Item`s are a special case since their capability providers are stored on an `ItemStack`. Instead, a provider should be attached through `Item#initCapabilities`. This should hold your capabilities for the lifecycle of the stack.

It is strongly suggested that direct checks in code are used to test for capabilities instead of attempting to rely on maps or other data structures, since capability tests can be done by many objects every tick, and they need to be as fast as possible in order to avoid slowing down the game.

Attaching Capabilities
----------------------

As mentioned, attaching capabilities to existing providers, `World`s, and `Chunk`s can be done using `AttachCapabilitiesEvent`. The same event is used for all objects that can provide capabilities. `AttachCapabilitiesEvent` has 5 valid generic types providing the following events:

* `AttachCapabilitiesEvent<Entity>`: Fires only for entities.
* `AttachCapabilitiesEvent<TileEntity>`: Fires only for tile entities.
* `AttachCapabilitiesEvent<ItemStack>`: Fires only for item stacks.
* `AttachCapabilitiesEvent<World>`: Fires only for worlds.
* `AttachCapabilitiesEvent<Chunk>`: Fires only for chunks.

The generic type cannot be more specific than the above types. For example: If you want to attach capabilities to `PlayerEntity`, you have to subscribe to the `AttachCapabilitiesEvent<Entity>`, and then determine that the provided object is an `PlayerEntity` before attaching the capability.

In all cases, the event has a method `#addCapability` which can be used to attach capabilities to the target object. Instead of adding capabilities themselves to the list, you add capability providers, which have the chance to return capabilities only from certain sides. While the provider only needs to implement `ICapabilityProvider`, if the capability needs to store data persistently, it is possible to implement `ICapabilitySerializable<T extends INBT>` which, on top of returning the capabilities, will provide NBT save/load functions.

For information on how to implement `ICapabilityProvider`, refer to the [Exposing a Capability][expose] section.

Creating Your Own Capability
----------------------------

In general terms, a capability is declared and registered through a single method call to `CapabilityManager.INSTANCE.register(...)`. One possibility is to define a static `register()` method inside a dedicated class for the capability, but this is not required by the capability system. For the purpose of this documentation, we will be describing each part as a separate named class, although anonymous classes are an option.

```java
CapabilityManager.INSTANCE.register(<capability interface class>, <storage>, <default implementation factory>);
```

The first parameter to this method is the type that describes the capability feature. In our example, this will be `IExampleCapability.class`.

The second parameter is an instance of a class that implements `Capability$IStorage<T>`, where T is the same class we specified in the first parameter. This storage class will help manage saving and loading for the default implementation, and it can, optionally, also support other implementations. This is just a helper and does not save or load data without being called within `ICapabilitySerializable`.

```java
private static class Storage
    implements Capability.IStorage<IExampleCapability> {

  @Override
  public INBT writeNBT(Capability<IExampleCapability> capability, IExampleCapability instance, Direction side) {
    // return an NBT tag
  }

  @Override
  public void readNBT(Capability<IExampleCapability> capability, IExampleCapability instance, Direction side, INBT nbt) {
    // load from the NBT tag
  }
}
```

The last parameter is a callable factory that will return new instances of the default implementation.

```java
private static class Factory implements Callable<IExampleCapability> {

  @Override
  public IExampleCapability call() throws Exception {
    return new Implementation();
  }
}
```

Finally, we will need the default implementation itself, to be able to instantiate it in the factory. Designing this class is up to you, but it should at least provide a basic skeleton that people can use to test the capability, if it is not a fully usable implementation itself.

Persisting Chunk and TileEntity capabilities
--------------------------------------------

Unlike Worlds, Entities, and ItemStacks, Chunks and TileEntities are only written to disk when they have been marked as dirty. A capability implementation with persistent state for a Chunk or a TileEntity should therefore ensure that whenever its state changes, its owner is marked as dirty.

`ItemStackHandler`, commonly used for inventories in TileEntities, has an overridable method `void onContentsChanged(int slot)` designed to be used to mark the TileEntity as dirty.

```java
public class MyTileEntity extends TileEntity {

  private final IItemHandler inventory = new ItemStackHandler(...) {
    @Override
    protected void onContentsChanged(int slot) {
      super.onContentsChanged(slot);
      setChanged();
    }
  }

  ...
}
```

Synchronizing Data with Clients
-------------------------------

By default, capability data is not sent to clients. In order to change this, the mods have to manage their own synchronization code using packets.

There are three different situations in which you may want to send synchronization packets, all of them optional:

1. When the entity spawns in the world, or the block is placed, you may want to share the initialization-assigned values with the clients.
2. When the stored data changes, you may want to notify some or all of the watching clients.
3. When a new client starts viewing the entity or block, you may want to notify it of the existing data.

Refer to the [Networking][network] page for more information on implementing network packets.

Persisting across Player Deaths
-------------------------------

By default, the capability data does not persist on death. In order to change this, the data has to be manually copied when the player entity is cloned during the respawn process.

This can be done via `PlayerEvent$Clone` by reading the data from the original entity and assigning it to the new entity. In this event, the `wasDead` field can be used to distinguish between respawning after death and returning from the End. This is important because the data will already exist when returning from the End, so care has to be taken to not duplicate values in this case.

Migrating from IExtendedEntityProperties
---------------------------

Although the Capability system can do everything IEEPs (IExtendedEntityProperties) did and more, the two concepts don't fully match 1:1. This section will explain how to convert existing IEEPs into Capabilities.

This is a quick list of IEEP concepts and their Capability equivalent:

* Property name/id (`String`): Capability key (`ResourceLocation`)
* Registration (`EntityConstructing`): Attaching (`AttachCapabilitiesEvent<Entity>`), the real registration of the `Capability` happens during `FMLCommonSetupEvent`.
* NBT read/write methods: Does not happen automatically. Attach an `ICapabilitySerializable` in the event and run the read/write methods from the `serializeNBT`/`deserializeNBT`.

Features you probably will not need (if the IEEP was for internal use only):

* The Capability system provides a default implementation concept, meant to simplify usage by third party consumers, but it doesn't really make much sense for an internal Capability designed to replace an IEEP. You can safely return `null` from the factory if the capability is for internal use only.
* The Capability system provides an `IStorage` system that can be used as a helper to read/write data from those default implementations.

The following steps assume you have read the rest of the document and you understand the concepts of the capability system.

Quick conversion guide:

1. Convert the IEEP key/id string into a `ResourceLocation` (which will use your MODID as a namespace).
2. In your handler class (not the class that implements your capability interface), create a field that will hold the Capability instance.
3. Change the `EntityConstructing` event to `AttachCapabilitiesEvent`, and instead of querying the IEEP, you will want to attach an `ICapabilityProvider` (probably `ICapabilitySerializable`, which allows saving/loading from NBT).
4. Create a registration method if you don't have one (you may have one where you registered your IEEP's event handlers) and in it, run the capability registration function.

[expose]: #exposing-a-capability
[network]: ../networking/index.md
