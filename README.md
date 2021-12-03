![Github CI](https://github.com/konovod/myecs/actions/workflows/ci.yml/badge.svg)
# MyECS

##### Table of Contents  
* [Main parts of ecs](#Main-parts-of-ecs)
  * [Entity](#Entity)
  * [Component](#Component)
  * [System](#System)
* [Special components](#Special-components)
  * [ECS::SingleFrame](#ECS::SingleFrame)
  * [ECS::MultipleComponents](#ECS::MultipleComponents)
  * [ECS::SingletonComponent](#ECS::SingletonComponent)
* [Other classes](#Other-classes)
  * [ECS::World](#ECS::World)
  * [ECS::Filter](#ECS::Filter)
  * [ECS::Systems](#ECS::Systems)
* [Engine integration](#Engine-integration)
* [Benchmarks](#Benchmarks)
* [Plans](#Plans)
* [Contributors](#Contributors)

## Main parts of ecs

### Entity
Сontainer for components. Consists from UInt64 and pointer to World`:
```crystal
struct Entity
  getter id : EntityID
  getter world : World
    ...
```

```crystal
# Creates new entity in world context. 
# Basically doesn't cost anything as it just increase entity counter, entity don't take space without components.
entity = world.new_entity

# destroying entity just removes all components from it. 
# For now IDs are not reused, so it is safe to hold entity even when it was destroyed and add components later
entity.destroy
```

### Component
Container for user data without / with small logic inside. Based on Crystal struct's:
```crystal
record Comp1 < ECS::Component,
  x : Int32,
  y : Int32,
  name : String
```
Components can be added, requested, removed:
```crystal
entity = world.new_entity
entity.add(Comp1.new(0, 0, "name"))
comp1 = entity.getComp1  # method is autogenerated from component class name. Will raise if component isn't present
comp2 = entity.getComp2? # will return nil if component isn't present
entity.remove(Comp1)

entity.replace(Comp1, Comp2.new) # basically shortcut for deleting one component and adding another.
```

They can be updated (changed) using several ways:
```crystal
entity = world.new_entity
entity.add(Comp1.new(0, 0, "name"))

# Replace Comp1 with another instance of Comp1. 
# Will raise if component isn't present
entity.update(Comp1.new(1, 1, "name1")) 

entity.set(Comp1.new(2, 2, "name2")) # Adds Comp1 or replace it if already present

ptr = entity.getComp1_ptr # autogenerated method, returns Pointer(Comp1), so can access it (unsafely) directly
ptr.value.x = 5
# important - after deleting component could be reused, so pointer could point to the component of some another entity.
# so don't save it if you are not sure that it won't be deleted
```

### System
Сontainer for logic for processing filtered entities. 
User class can implement `init`, `execute`, `teardown`, `filter` and `process` (in any combination. Just skip methods you don't need).
```crystal
class UserSystem < ECS::System
  # @world : ECS::World - world
  # @active : Bool - allows to temporary enable or disable system

  def initialize(@world : ECS::World)
    super(@world) # constructor should pass @world property
  end

  def init
    # Will be called once during ECS::Systems.init call
  end

  def execute
    # Will be called on each ECS::Systems.execute call
  end

  def teardown
    # Will be called once during ECS::Systems.teardown call
  end

  def filter(world)
    # Called once during ECS::Systems.init, after #init call.
    # If this method present, it should return a filter that will be applied to a world
    world.of(SomeComponent)
  end

  def process(entity)
    # will be called during each ECS::Systems.execute call, before #execute, for each entity that match the #filter
  end
end
```

### Special components
#### ECS::SingleFrame
annotation `@[ECS::SingleFrame]` is for components that have to live 1 frame (usually - events). The main difference is that they are supposed to be deleted at once, so their storage can be simplified (no need to track free indexes). They should be deleted by adding `ECS::RemoveAllOf(T)` system in a right place of systems list.

```crystal
require "./src/myecs"

@[ECS::MultipleComponents]
@[ECS::SingleFrame]
record SomeRequest < ECS::Component, data : String

class ExecuteSomeRequestsSystem < ECS::System
  def filter(world) : ECS::Filter?
    world.of(SomeRequest)
  end

  def process(ent)
    req = ent.getSomeRequest
    puts "request #{req.data} called for #{ent}"
  end
end

world = ECS::World.new
systems = ECS::Systems.new(world)
  .add(ExecuteSomeRequestsSystem)
  .add(ECS::RemoveAllOf(SomeRequest))
systems.init
# now you can add multiple SomeRequest to the same entity
world.new_entity.add(SomeRequest.new("First")).add(SomeRequest.new("Second"))
systems.execute
```
#### ECS::MultipleComponents
Note above example also shows the use of `@[ECS::MultipleComponents]`. This is for components that can be added multiple times. They have some limitations though - filters can't iterate over several of components with this annotation (as this would usually mean cartesian product, unlikely needed in practice), there is no way to get multiple components outside of filter (planned), `delete` deletes all of components on target entity and there is no way to delete only one.

#### ECS::SingletonComponent
annotation `@[ECS::SingletonComponent]` is for data sharing. It creates component that is considered present on every entity (iteration on it isn't possible though). So you can do

```crystal
@[ECS::SingletonComponent]
record Config < ECS::Component, values : Hash(String, Int32)

class InitConfigSystem < ECS::System
  def init
    config = ...some config initializatio
    @world.new_entity.add(Config.new(config))
  end
end

class SomeAnotherSystem < ECS::System
  def execute
    config = @world.new_entity.getConfig.values # gets the same values
  end
end
```

### Other classes

#### ECS::World
Root level container for all entities / components, is iterated with ECS::Systems:
```crystal
world = ECS::World.new

# you can delete all entities
world.delete_all

# you can create entity
world.new_entity

# you can iterate all entities
world.each_entity do |entity|
  puts entity.id
end

# you can create filters
world.any_of([Comp1, Comp2]).any_of([Comp3, Comp4])
```

#### ECS::Filter
Allows to iterate over entities with specified conditions.
Created by call `world.new_filter` or just by adding any conditions to `world`.

Filters that is possible:
  - `any_of([Comp1, Comp2])`: at least one of the components must be present on entity
  - `all_of([Comp1, Comp2])`: all of the components must be present on entity
  - `of(Comp1)`: alias for `all_of([Comp1])`
  - `exclude([Comp1])`: none of the components could be present on entity
  - `select{|ent| some_check(ent) }`: user-provided filter procedure, that must return true for entity to be passed.

All of them can be called 0, 1, or many times using method chaining. 
So `any_of([Comp1, Comp2]).any_of([Comp3, Comp4])` means that either Comp1 or Comp2 should be present AND either Comp3 or Comp4 should be present.

#### ECS::Systems
Group of systems to process `EcsWorld` instance:
```crystal
world = ECS::World.new
systems = ECS::Systems.new(world)

systems
  .add(MySystem1.new(world))
  .add(MySystem2) # shortcut for add(MySystem2).new(@world)
  .add(MySystem3)

systems.init
loop do
  systems.execute
end
systems.teardown
```
You can add Systems to Systems to create hierarchy.
You can inherit from `ECS::Systems` to add systems automatically:
```crystal
class SampleSystem < ECS::Systems
  def initialize(@world)
    super
    add InitPlayerSystem
    # note that shortcut `add KeyReactSystem` isn't applicaple here because 
    # system require other params in initialize
    add KeyReactSystem.new(@world, pressed: CONFIG_PRESSED, down: CONFIG_DOWN)
    add ReactPlayerSystem
    add MovePlayerSystem
    add RotatePlayerSystem
    add StopRotatePlayerSystem
    add SyncPositionWithPhysicsSystem
    add DrawDebugSystem
  end
end
```
### Engine integration
huh, this is integration with my nonoengine:
```crystal
require "./ecs"
require "./basic_systems"
require "./physics_systems"
require "./demo_systems"

@[ECS::SingleFrame]
struct QuitEvent < ECS::Component
end

world = ECS::World.new
systems = ECS::Systems.new(world)
  .add(BasicSystems)
  .add(PhysicSystems)
  .add(SampleSystem)

systems.init
quitter = world.of(QuitEvent)
loop do
  systems.execute
  break if quitter.find_entity?
  world.clear_single_frame
end
systems.teardown

...

require "./libnonoengine.cr"
require "./ecs"

class BasicSystems < ECS::Systems
  def initialize(@world)
    super
    add EngineSystem.new(world)
    # add RenderSystem.new(world)
    add ShouldQuitSystem.new(world)
  end
end

class EngineSystem < ECS::System
  def init
    Engine[Params::Antialias] = 4
    Engine[Params::VSync] = 1

    Engine.init "resources"
  end

  def execute
    Engine.process
  end
end

class ShouldQuitSystem < ECS::System
  def execute
    @world.new_entity.add(QuitEvent.new) if !Engine::Keys[Key::Quit].up?
  end
end

```

see `bench_ecs.cr` for some examples, and `spec` folder for some more. Proper documentation and examples are planned, but not soon.

## Benchmarks
I'm comparing it with https://github.com/spoved/entitas.cr with some "realistic" scenario - creating world with 1_000_000 entities, adding and removing components in it, iterating over components, replacing components with another etc.
You can see I'm not actually beating it in all areas (I'm much slower in access but much faster in creation), but my ECS looks fast enough for me. What is I'm proud - 0.0B/op for all operations (after initial growth of pools)

my ECS:
```
***********************************************
              create empty world 342.45k (  2.92µs) (± 5.00%)  27.4kB/op           fastest
          create benchmark world   7.45  (134.14ms) (± 1.23%)   273MB/op  45936.43× slower
create and clear benchmark world   7.39  (135.35ms) (± 0.75%)   273MB/op  46350.85× slower
***********************************************
                   EmptySystem 151.33M (  6.61ns) (± 1.54%)  0.0B/op        fastest
             EmptyFilterSystem  32.62M ( 30.66ns) (± 1.33%)  0.0B/op   4.64× slower
SystemAddDeleteSingleComponent   8.88M (112.60ns) (±17.89%)  0.0B/op  17.04× slower
 SystemAddDeleteFourComponents   1.73M (577.71ns) (± 8.06%)  0.0B/op  87.42× slower
         SystemAskComponent(0)  63.08M ( 15.85ns) (± 2.34%)  0.0B/op   2.40× slower
         SystemAskComponent(1)  60.03M ( 16.66ns) (± 2.98%)  0.0B/op   2.52× slower
         SystemGetComponent(0)  63.00M ( 15.87ns) (± 3.07%)  0.0B/op   2.40× slower
         SystemGetComponent(1)  42.17M ( 23.72ns) (± 1.65%)  0.0B/op   3.59× slower
   SystemGetSingletonComponent 103.17M (  9.69ns) (± 2.85%)  0.0B/op   1.47× slower
***********************************************
         SystemCountComp1 260.93  (  3.83ms) (± 0.31%)  0.0B/op        fastest
        SystemUpdateComp1  97.02  ( 10.31ms) (± 0.52%)  0.0B/op   2.69× slower
SystemUpdateComp1UsingPtr 214.56  (  4.66ms) (± 0.24%)  0.0B/op   1.22× slower
       SystemReplaceComp1   9.55  (104.70ms) (± 2.63%)  0.0B/op  27.32× slower
         SystemPassEvents  14.27  ( 70.07ms) (± 0.65%)  0.0B/op  18.28× slower
***********************************************
         FullFilterSystem  17.62  ( 56.75ms) (± 1.77%)  0.0B/op   1.93× slower
    FullFilterAnyOfSystem  15.76  ( 63.45ms) (± 1.08%)  0.0B/op   2.16× slower
      SystemComplexFilter  33.70  ( 29.67ms) (± 0.56%)  0.0B/op   1.01× slower
SystemComplexSelectFilter  34.08  ( 29.35ms) (± 0.50%)  0.0B/op        fastest
```
Entitas.cr (it is slightly outdated so you will have problems to make it work)
```
***********************************************
              create empty world   2.50M (399.61ns) (±16.26%)  1.66kB/op             fastest
          create benchmark world   1.55  (643.32ms) (±11.33%)  1.57GB/op  1609878.49× slower
create and clear benchmark world   1.04  (958.39ms) (± 0.87%)  1.67GB/op  2398325.42× slower
***********************************************
                   EmptySystem 320.60M (  3.12ns) (± 1.66%)    0.0B/op         fastest
             EmptyFilterSystem 244.01M (  4.10ns) (± 4.15%)    0.0B/op    1.31× slower
SystemAddDeleteSingleComponent 537.81k (  1.86µs) (±11.43%)  1.59kB/op  596.13× slower
 SystemAddDeleteFourComponents 487.55k (  2.05µs) (±15.30%)   1.6kB/op  657.57× slower
         SystemAskComponent(0) 216.22M (  4.63ns) (± 4.59%)    0.0B/op    1.48× slower
         SystemAskComponent(1) 216.62M (  4.62ns) (± 1.87%)    0.0B/op    1.48× slower
         SystemGetComponent(0) 217.34M (  4.60ns) (± 4.63%)    0.0B/op    1.48× slower
         SystemGetComponent(1) 216.66M (  4.62ns) (± 3.60%)    0.0B/op    1.48× slower
***********************************************
         SystemCountComp1   8.96k (111.57µs) (± 0.21%)    0.0B/op          fastest
        SystemUpdateComp1  22.47  ( 44.50ms) (± 0.43%)    0.0B/op   398.85× slower
SystemUpdateComp1UsingPtr   7.07  (141.54ms) (± 0.99%)    0.0B/op  1268.61× slower
       SystemReplaceComp1   4.46  (224.01ms) (± 1.19%)  3.81MB/op  2007.79× slower
***********************************************
     FullFilterSystem 215.91M (  4.63ns) (± 2.55%)  0.0B/op     1.00× slower
FullFilterAnyOfSystem 216.47M (  4.62ns) (± 2.01%)  0.0B/op          fastest
  SystemComplexFilter  39.19k ( 25.52µs) (± 0.31%)  0.0B/op  5524.39× slower
```

## Plans
### Short-term
- [ ] Reuse entity identifier, allows to replace `@sparse` hash with array
- [ ] better API for multiple components - iterating, array
- [ ] optimally delete multiple components (linked list)
- [ ] check that all singleframe components are deleted somewhere
- [ ] benchmark comparison with flecs (https://github.com/jemc/crystal-flecs)
### Future
- [ ] Callbacks on adding\deleting components
- [ ] Work with arena allocator to drop GC completely
## Contributors
- [Andrey Konovod](https://github.com/konovod) - creator and maintainer
