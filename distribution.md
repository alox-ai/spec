# Distributed Actors

> see `actor-model.md`

Actors disitributed across many machines gives us the power to write complex
concurrent applications with little regard for the underlying hardware. For most
cases, distribution of actors will come at no mental expense to the user. 

## Stages

_Stages_ offer a balance in terms of performance. If actors are free to live on
any scheduler on any processor, passing references to memory between them is 
impossible. Instead of copying data all over the place, we can use a _stage_.
Stages are guaranteed to be in the same memory space, so passing references
between actors on the same stage is possible.

```rust
stage WorldStage

actor World on WorldStage {
    let entities: List<Entity>
    
    behave tick() {
        for (entity in entities) {
            if (entity.alive) {
                entity.tick(&this)
            }
        }
    }
}

trait Entity {
    let alive: Boolean
    let health: Int32

    new() {
        this.alive = true
        this.health = 20
    }

    behave tick(world: &World)

    behave die()
}

actor Sheep on WorldStage is Entity {
    let grassEaten: Int32

    new() {
        Entity.new()
        this.grassEaten = 0
    }
    
    behave tick(world: &World) {
        this.grassEaten += 2;
    }
}

actor Cow on WorldStage is Entity {
    let milk: Int32

    new() {
        Entity.new()
        this.grassEaten = 0
    }

    behave pong(world: &World) {
        this.milk -= 1;
        if (this.milk <= 0) {
            this.die()
        }
    }
}
```
